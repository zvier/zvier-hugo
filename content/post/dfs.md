---
title: "深度优先搜索(dfs)"
date: 2019-11-02T18:53:59+08:00
draft: true
categories: ["技术"]
tags: ["算法"]
---
<!--more-->

# 简述
图的许多性质都和路径有关，因此我们需要有一种方法，沿着图的边从一个顶点移动到另一个顶点。深度优先遍历(dfs)就是期中一种

# 算法描述
搜索图，只需要用一个递归方法遍历所有顶点，在访问每一个顶点时：
1. 将它标记为已访问
2. 递归地访问它的所有没被标记过的邻居顶点  
递归方法会标记给定的顶点，并调用自己来访问该顶点的相邻顶点列表中所有没有标记过的顶点。如果图是连通的，那每个邻接列表中的元素都会被检查到

# 文件路径
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

# 参考
1. 算法

