---
layout:     post
title:      "Louvain Clustering -- Modularity"
subtitle:   ""
date:       2014-9-2 16:50:00
author:     "YuanBao"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog: true
tags:
    - algorithm
---

在针对计算机网络，信息网络，生物网络图的研究过程中，研究者发现了大量的结构特性，比如 [small-world property](http://en.wikipedia.org/wiki/Watts_and_Strogatz_model)，[clustering property](http://en.wikipedia.org/wiki/Clustering_coefficient) 等。而在这些图特性中，Community Structure（社交结构）得到了学术界最为广泛的关注和最为深入的探索。

在一个网络中，一个Community Structure代表图中的某些节点的集合。这些节点内部具有高度的耦合性（表现在内部的连通性非常高，内部节点之间的边非常丰富），同时与外界的其他节点保持着较为稀疏的连接。在一个Community Structure内部，节点之间的紧密连接使得信息的传输速度非常快。同时，Community Structure的存在也使得网络具有天然可被分割的性质。

Community Structure在真实的网络中应用非常广泛。在Social network中，基于社群结构的分析能够提高好友推荐，信息传播，广告定位的精确性；在Citation network中，图的社群特性能够精确的区分不同的research topic；在代谢网络中，不同节点的生物功能也能够通过社群结构的特性反应出来。甚至于在生物的大脑中，都展现出了高度的Community Sturcture。由于Community Structure特性在各个领域广泛的应用，如何将图分解为合适的Communities得到了许多学者大量的探索和研究。本文将要介绍这些方法中最为高效的一种：Louvain Method （查了一下，Louvain是比利时的一个城市，当时这篇文章的作者都在Louvain的时候提出了这个方法，因此该算法一直被称为Louvain Method或者Louvain Clustering）。之所以说Louvain Method高效，是因为在作者当时的环境下，“分析出一个具有一亿一千八百万规模的网络的社群结构只需要152分钟”。

事实上，Louvain Method是一种基于图的Modularity的算法。因此我将有关Louvain Method的介绍分为两个部分。本文作为上半部分首先具体介绍一下什么是图的Modularity。

##图的Modularity
Modularity是一种图结构的度量，它度量了将网络划分为不同的Communities的强度。给定一个图$G$和一个社群关系函数$C$，其中$C(v) = 1$表明节点$v$在社群$C$中，那么图$G$的Modularity被定义为社群$C$中边的数目减去对应随机图的$C$中的边数，再除以图中的总边数。这个定义一定把你给绕晕了，因为看完你还是不知道图的Modularity应该怎么计算。那下面我们就来一个case study。

假设图$G$有$m$条边以及$n$个顶点，其邻接矩阵为$A_{uv}$,节点$v$的度被记为$k_{v}$。为了方便，不妨假设图$G$是一个无向图，即$A(u,v) = A(v,u)$，同时假设只需要将图分为两个社群1和2。如果节点$v$属于社群1，那么$s_{v} = 1$，否则$s_{v} = -1$。好了有了这些记号，我们可以分为三步来计算这个图的Modularity。具体是哪三步呢？请再回过头去看一下Modularity的定义：

 1. 计算社群中边的数目$\alpha$
 2. 计算对应随机图中社群内边的数目$\beta$
 3. 计算最终的Modularity为$( \alpha - \beta ) / 2m $

####计算$\alpha$
事实上，在本例中社群中边的数目很好计算。记$\delta(v,u)$指示节点$v$和$u$是否在同一个社群中，那么根据我们上述对于$s_{v}$的定义，$\delta(v,u)$可以很容易地表示如下:
$$\delta(v,u) = \frac{s_{v}s_{u} + 1}{2}$$
很明显，只有当$u$和$v$都在同一个community中时，$\delta(v,u)$才有可能等于1，否则其等于0。那么结合邻接矩阵的定义，得到$\alpha$如下:
$$\alpha = \sum_{vu}A(v,u) * (\frac{s_{v}s_{u} + 1}{2})$$

####计算$\beta$
为了计算对应随机图中的社群内边数，我首先要有一个能够生成对应随机图的机制。针对原有的网络图，我们将每条边从中间打断，使得每条边变成了两个stub。通过随机的方式再将任意两个stub连接起来（允许在节点上形成环），构成了一个与原来图对应的随机图$G^{*}$（很明显$G^{*}$的任意节点度都和原来图的节点度一致）。

接下来我们就可以计算此随机图中位于相同社群中的边数。对于任意两个节点$v$和$w$，在stub的随机连接中，产生连接$v$和$w$边的期望数目为：
$$E(Num(e_{vw})) = k_{v} * \frac{k_{w}}{2m}$$
其中，$2m$为所有的stub的总的数目。结合针对$\delta(v,u)$的定义，可以得到位于相同社区中边的数目为:
$$\beta =\sum_{vw} \frac{k_{v} * k_{w}}{2m} * (\frac{s_{v}s_{u} + 1}{2})$$

####计算最终的Modularity
根据Modularity的定义，得到上述图$G$的modularity为：
$$Mod = \frac{1}{2m}\sum_{uv}(\alpha - \beta) = \frac{1}{2m}\sum_{uv}(A(v,u) - \frac{k_{v} * k_{w}}{2m})* (\frac{s_{v}s_{u} + 1}{2})$$


##Modularity的矩阵表示
我们定义$S_{vr} = 1$表示顶点$v$属于群组$r$。那么可以得到$\delta(c_{v}, c_{w}) = \sum_{r}{S_{vr}S_{wr}}$
因此就有：
$$Mod = \frac{1}{2m}\sum_{uv}(A(v,u) - \frac{k_{v} * k_{w}}{2m})*S_{vr}S_{wr} = \frac{1}{2m}Tr(S^{T}BS) $$其中$B_{vw} = A_{vw} - \frac{k_{v}*k_{w}}{2m}$.
