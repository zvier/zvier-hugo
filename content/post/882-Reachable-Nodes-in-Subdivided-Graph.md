---
title: "882.Reachable-Nodes-in-Subdivided-Graph"
date: 2019-10-19T15:56:52+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
<!--more-->
# 题目描述
Starting with an undirected graph (the "original graph") with nodes from 0 to N-1, subdivisions are made to some of the edges.

The graph is given as follows: edges[k] is a list of integer pairs (i, j, n) such that (i, j) is an edge of the original graph,

and n is the total number of new nodes on that edge.

Then, the edge (i, j) is deleted from the original graph, n new nodes (x_1, x_2, ..., x_n) are added to the original graph,

and n+1 new edges (i, x_1), (x_1, x_2), (x_2, x_3), ..., (x_{n-1}, x_n), (x_n, j) are added to the original graph.

Now, you start at node 0 from the original graph, and in each move, you travel along one edge.

Return how many nodes you can reach in at most M moves.

# 题目解释
给定一个N个节点的原始无向图，用如下方式表示:  
1. edges[k]是一个由多个(i, j, n)形式的元组构成的列表，其中(i, j)是图中的边，n是这条边上的新节点   
2. 然后从原始图中删除边(i, j)，将边上的新节点(x_1, x_2, ..., x_n)添加到原始图中  
3. 再将n+1条新边(i, x_1), (x_1, x_2), (x_2, x_3), ..., (x_{n-1}, x_n), (x_n, j)添加到原始图中   

现在，从图中的0节点出发移动M次，每次移动只能沿着一条边行进，求这M次移动可达的节点数

图中有N个编号的大节点，夹杂若干没编号的小节点，可以把该问题看成一个飞行路径规划问题，N个大节点当作N个省会城市，具备中转功能，中间小节点当作普通小城市，只能在直线上前进或后退，当然是可以跳过城市直飞的，只要在直线方向上即可。规划一条路径，看如何使得M次飞行经过的城市最多，输出城市数  

# 题目分析
实际上，该问题还是图的遍历问题:
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

