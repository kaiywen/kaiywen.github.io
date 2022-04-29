---
layout:     post
title:      "TiDB Percolator 事务实现解析" 
subtitle:   ""
date:       2021-03-30 17:59:21
author:     "YuanBao"
header-img: "img/scene1.jpg"
header-mask: 0.25
catalog: true
tags:
 - Distributed
 - Protocal
---

2019 年读源码时做的笔记，今天偶然翻到，搬到这里来。

看了一下 tidb 和 tikv 的源码，总结一下其事务实现如下，很多细节已经与 Percolator 不同。

<!--more-->

## tidb 的 percolator 实现

* tidb 端，等价于 percolator client 端，代码主要在：[tikv](https://github.com/pingcap/tidb/blob/master/store/tikv)
* tikv 端，等价于 percolator server 端，代码主要在：[tidb](https://github.com/tikv/tikv/tree/master/src/storage)

### tidb (client）端主要流程
#### 基本流程
1. 在 tidb 端入口由 `txn.go` 调用 `commit` 后创建一个 `twoPhaseCommitter`，`twoPhaseCommitter` 调用 `execute` 开始执行事务；
2. tidb 在解析完请求之后，会将涉及到的 key 分为三类：
   * Put 代表需要进行修改的 key;
   * Delete 代表需要删除的 key;
   * Lock 代表此外还需要加锁的 key;
   根据涉及到的 key 以及 value 的 size 来估算事务的大小，从而确定本次事务 lock 的 TTL，这里的 key type 会被 tikv 使用；
3. 根据路由 cache，将涉及到的所有 key 划分到其归属的 region(server)，每个 region(server) 对应一个 key batch，从而形成一个 batch 数组；选择第一个 batch 的第一个 key 作为 primary key；如果后续的任何请求出现路由失败，都需要根据最新的路由重新划分 batch，并进行重试；
4. 开始对每个 Batch 进行 `Prewrite`，**这里的实现与论文不同，所有 Batch 的 `Prewrite` 都是并行执行的，而不是先执行 Primary 的 batch 之后再执行 secondary batch**，执行之后会返回每个 Batch `Prewrite` 的结果，处理如下：
   * 如果当前 Batch 遇到 `Lock Conflict`，那么 tidb 会调用 `lockResolver.ResolveLocks` 向 tikv 发送 `ResolveLocks` 请求，请求如果未返回错误，则对当前的 batch 进行 `Prewrite` 重试；（ResloveLocks 逻辑见下文）
   * 如果当前 Batch 遇到 `Write Conflict`，直接取消当前事务执行，进行 backoff 之后重新获取 `start_ts`，重执行当前事务；
   * 如果遇到其他未知错误，返回 client 错误；
5. `Prewrite` 成功之后，开始进行 `commit` 操作；`commit` 操作获取到 `commit_ts` 之后，保证先 commit primary 所在的 batch，之后再异步并行地 commit secondary batch；Primary batch commit 成功后，标记当前事务执行成功，此时如果 secondary batch 的异步 commit 有失败，锁处理交由上述的 `ResolveLock` 来清理；
6. 如果 Primary 的 `commit == false && undetermined == false`，则向tikv 发送 `cleanup` 操作。这里引入 `undetermined` 是为了避免 rpc 出现异常，例如当 rpc 出现超时，client 不能判断本次 commit 究竟是成功还是失败，会标记 undetermined 为 true，此时不能进行 `cleanup`；

#### ResolveLock 流程
tidb 使用一个 lock_resolver 模块来记录事务的执行状态并完成 `ResolveLocks` 过程；
1. 对于 `Prewrite` 返回的冲突列表 `locks` 进行过滤，如果 `Lock.ttl` 已经过期，那么加入到待处理的 `expired_locks` 列表内，否则不进行任何处理；
2. 对于 `expired_locks` 内每个待处理的 lock，tidb 调用 `GetTxnStatus` 去 tikv 查询其对应的 Primary Lock 的状态，也就是当前 lock 所在的事务的状态，tidb 会将这个事务状态缓存下来；如果查询返回了 committed_ts，表明当前 lock 所在的事务已经 `committed`；
3. 查询到状态之后，tidb 向 tikv 发送 `resolveLock(start_ts, committed_ts)` 请求k，tikv 会根据 `committed_ts` 值来决定 rollback 还是 commit 当前 lock 对应的数据；**根据代码，这里对于所有 key 的 `GetTxnStatus` 和 `resolveLock(l)` 的请求都是串行执行的**；
4. 如果任何 `GetTxnStatus(l)` 和 `resolveLock(l)` 返回错误，则 `ResolveLocks` 返回错误；如果都执行成功并且 len(locks) == len(expired_locks)，那么返回 `OK`，意味着针对当前 region 的操作（Prewrite 或者 Get）可以立即重试，否则需要进行 backoff 之后再重试；

#### cleanup 流程
cleanup 在 tidb 确认 commit 失败之后调用，实际上 tidb 在执行 cleanup 时发送的是 **RollBack** 请求；**与 commit 一样，cleanup 必须先执行 Primary batch，之后可以并行执行  secondary batches**.

#### Get 操作
上述的 tidb 测流程都是写操作，读操作的流程如下：
1. tidb 向 tikv 发送 `GetRequest`;
2. 如果 tikv 返回成功则结束，否则检查错误是否为 `Lock confict`；
3. 如果出现 `Lock conflict`，那么还是调用 `lockResolver.ResolveLocks` 去尝试 resolve；
4. 如果 resolve 成功，则进行 Get 重试；否则返回 Get 失败；

### tikv (server) 端主要流程：
tikv 端实现需要一个高效的 MvccReader，能够根据 `key+ts` 来快速的seek 到指定的位置。这里 tikv 主要基于 RocksDB 实现，其 MvccReader 的主要优化思路：
1. 利用 prefix_seek；
2. 对于一些连续的请求，上层的 seek 都尝试往下执行 BOUND 次的 next 来查找数据；如果未找到数据，再调用 seek；
这里 wbt 的 Mvcc 读取是否能够高效也是一个考虑点；

#### Prewrite
tikv 接收到 tidb 的 `Prewrite` 请求后，处理逻辑主要如下：
1. 对于每一个 key，创建一个 `MvccTxn`，并调用 `Prewrite`，对于每个 key 的 `Prewrite` 结果都记录在内存中，待所有 key 处理完成后一并持久化；
2. `Prewrite` 首先调用 `reader.seek_write(key, TimeStamp::max())` 来在 `write` 列查找当前最大的一个 `commit_ts`，如果 `write.commit_ts >= key.start_ts`， 表明产生 write conflict，返回写冲突错误。这里细节需要说明，正常的 Percolator 论文实现中，此时查到的 write 列是不可能出现 `write.commit_ts == key.start_ts` 的状态，这里由于 RollBack 机制（见下文）的设计，**如果查询到的 write 列中数据满足 write.commit_ts == key.start_ts，那么同样返回写冲突即可**；
3. 调用 `self.reader.load_lock(&key)` 来获取当前 key 的 lock 数据，如果 lock 存在并满足 `lock.ts == key.start_ts` 则直接返回成功（当前 key 已经被相同的 Prewrite 请求加过锁了，即允许 client 端发送的 Prewrite 请求成为幂等操作，多次收到相同的 Prewrite 不会产生异常）。如果 `lock.ts ！= key.start_ts`，则记录 lock conflict 错误，继续下一个 key；
4. 依照论文中的步骤对当前 key 插入 lock 以及 data 数据，tikv 这里有两个优化需要说明
   * 调用 `is_short_value` 来判断当前的 value 是否是小 value，如果是小 value，那么将 value 与 lock 直接保存在一起写入，否则按照论文要求分开写入；
   * 将不同的 key 类型（Put，Delete，Lock）在 Lock 列中保存为不同的LockType，在后续进行 RollBack 时利用 LockType 进行优化操作；

#### Commit
tikv 对 `Commit` 的处理逻辑如下：
1. 对每个 key 分别调用 `txn.commit(k)`，首先 `load_lock(k)`，如果满足 `lock.ts == key.start_ts`，那么检查成功，直接写入 write 列，并删除 lock 列；
2. 如果检查不成功，那么调用 `get_txn_commit_info(k，start_ts)` 去获取 write 列的提交信息，与 lock_type 类似，write 列记录的数据同样有 write_type。**这里的实现是 Percolator 原文未提及的细节**：
   * 如果此时 write_type == RollBack，表明有其他的并发事务已经进行了 rollback（参见上文 tidb 流程中的 cleanup 介绍），此时不能再完成 commit，返回对应错误；
   * 如果 write_type 为其他（Put，Delete），表明其他的并发事务（由于网络原因产生的重试）已经提交了当前 key，可以直接返回 OK；

#### RollBack
Percolator 论文中未提及 RollBack 相关的操作，tikv 在收到 tidb 的 rollback 请求之后（tidb 调用 cleanup 时产生），对每个 key 调用一次 `txn.cleanup(k)`，具体逻辑如下:
1. 调用 `load_lock(&key, start_ts)` 并检查，如果 `lock.ts == key.start_ts` 满足；并且当前 lock 的 ttl 过期，删除对应的 lock 数据和 data 数据。**由于在先前进行 Prewrite 的过程中对不同的 key 写入了不同类型的 Lock (Put,Delete,Lock)**，删除时如果 LockType 不为 Put，表明针对当前key 的操作不是写入操作，无需删除 data，否则执行 `delete_data`。删除完成之后，在当前 key 的 write 列中写入 (RollBack, start_ts, start_ts) 数据（即当前 commit 为 rollback 类型，并且 commit_ts 设置为 start_ts）；
2. 如果 lock 不存在，则获取 (key, start_ts) 在 write 列中对应的提交信息：
   * 如果提交信息中 write_type == RollBack，表明当前 key 已经被其他并发事务回滚，继续下一个key；
   * 如果 write_type 为其他，表明当前 key 已经被提交，终止并直接返回 rollback 失败；
   * 如果提交信息不存在，那么同样在当前 key 的 write 列中写入 (RollBack, start_ts, start_ts)；
3. RollBack 与 Prewrite 一样，上述对每个 key 的处理结果都暂存在内存中，所有 key 都处理完成之后，一并持久化到底层存储；
4. 持久化完成之后，返回回滚成功；

这里之所以需要在 Write 列中写入 (RollBack, start_ts, start_ts)，是为了避免乱序的 Prewrite 造成的影响。考虑到网络原因产生了 Prewrite 重试，如果一个 key 在 rollback 之后又收到了一个因重试产生的 Prewrite，该 Prewrite 会产生锁，在锁 TTL 过期之前都会阻塞当前 key 的读写。在 write 列中写入 (RollBack, start_ts, start_ts) 可以避免这种情况（回过头去在看下 Prewrite 流程就明白了）。

#### ResolveLock
关于 tikv 的 resolve lock 的操作，前面介绍了，tidb 对于一个 lock 的 resolve 操作会分为 `GetTxnStatus(l)` 和 `resolveLock(l)` 两步，第一步是获取 Primary key 的信息，第二步根据 primary key 的状态进行操作；

1. tidb 的 `GetTxnStatus(l)` 实际上是发送了一个 `cleanup` 请求给 tikv，tikv 接收到请求后会调用 RollBack，如果 primary key 已经被提交，那么对其 rollback 会失败，并且返回给 tidb 对应的 commit_ts，否则回滚成功；
2. tidb 发送 `resolveLock` 请求时，实际携带的参数是 lock 对应的 start_ts，tikv 收到 resolve 请求之后：
   * 在当前 region 内找到一批 `lock.start_ts == start_ts` 的锁集合；
   * 对当前锁集合中的每个 lock，如果 `resolveLock` 请求中携带的 commit_ts 为 0，则执行回滚，否则执行提交；执行结果都记录在内存中
   * 获取下一批锁集合迭代处理，如果此时锁集合为空，则将内存中的结果进行持久化，并返回成功；

#### Get 请求处理
1. tikv 收到 `Get(key, start_ts)` 请求之后，检查当前 key 的 lock，如果满足 `lock.ts < start_ts`，表明当前时刻被上锁，返回 Lock Conflict，tidb 会进行 ResolveLock；
2. 调用 `read.seek_write(key, start_ts)` 来获取 start_ts 之前最大 commit_ts 的提交记录：
   * 如果提交记录为空或者 `write_type == DELETE`，表明当前 key 已经被删除，返回空结果；
   * 如果 `write_type == Put`，返回对应值，读取成功；
   * 如果 `write_type == rollback`，表明当前版本被回滚，此时令 `start_ts = write.commit_ts-1`，迭代查找上一个版本；
