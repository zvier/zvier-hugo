---
title: "union find"
date: 2019-10-19T19:32:40+08:00
draft: true
categories: ["技术"]
tags: ["算法"]
---
帘外雨潺潺，春意阑珊。 罗衾不耐五更寒。 梦里不知身是客，一晌贪欢。
独自莫凭栏，无限江山， 别时容易见时难。 流水落花春去也，天上人间。——李煜
<!--more-->
# 动态连通性问题
假设集合S有n个元素，需要将其动态划分为一系列不相交的子集S1, S2, ...,
Sk，初始化时可划分为n个子集，每个子集包含S中的一个不同元素，然后对这些子集做一系列求并集和查找操作，按规则合并子集，直到不能继续合并。显然，求并集的操作次数不会超过n-1次，因为每个求并操作要求至少会把子集大小增加1，而在整个集合S中只有n个元素。

# 基本操作
1. makeset(x): 用于生成一个单元素集合{x}，这个操作对集合S的每一个元素只能应用一次，相当于对给定n个元素集合初始化初始化为n个单元素的子集合  
2. find(x): 返回一个包含x的子集
3. union(x, y): 构造分别包含x和y的不相交子集Sx和Sy的并集，并把它添加到子集的集合中，以代替被删除后的Sx和Sy
3. connected(x, y): 如果x, y在同一个连通分量，则返回true
4. count(): 返回连通分量的数量

# 示例
假设有集合S={1, 2, 3, 4, 5, 6}  

1. makeset(i)可以用于生成集合{i}，把这个操作执行6次，就初始化了一个由6个单元素集合组成的结构
{{< highlight go "linenos=inline" >}}
{1}, {2}, {3}, {4}, {5}, {6}
{{< /highlight >}}

2. 然后执行union(1, 4)和union(5, 2)产生出
{{< highlight go "linenos=inline" >}}
{1, 4}, {5, 2}, {3}, {6}
{{< /highlight >}}

3. 如果再执行union(4, 5)和union(3, 6)，则最后得到了两个不相交的子集
{{< highlight go "linenos=inline" >}}
{1, 4, 5, 2}, {3, 6}
{{< /highlight >}}

# 实现
以下Union-Find算法均以触点为索引的id数组确定两个触点是否存相通
## quick-find算法
quick-find算法认为当前仅当id[p]==id[q]时，p和q连通。union(p,
q)调用会先检查p和q是否已经连通，如果连通则return，否则我们需要将p所在连通分量和q所在的连通分量合并，即要将两个集合中所有元素的id值统一

### find
{{< highlight go "linenos=inline" >}}
func find(id []int, p) {
    return id[p]
}
{{< /highlight >}}

### union
{{< highlight go "linenos=inline" >}}
func union(id *[]int, p, q int) {
    pid := find(*id, p)
    qid := find(*id, q)
    if pid == qid {
        return
    }
    for i, _ := range *id {
        if (*id)[i] == qid) {
            (*id)[i] = pid
        }
    }
}
{{< /highlight >}}

## quick-union算法
与quick-find相对应的是quick-union算法，自然quick-union算法的快体现在以牺牲find为代价加速union。同样，quick-union算法也基于以触点为索引的id数组，不过id数组元素的含义不再是对应触点的分量名，而是触点同分量中的另一个触点或它自己，从而形成一系列链接。  

### find
find时，我们可以从给定触点开始，由链接得到另外一个触点，并依此继续得到下一个触点，最终抵达根触点，根触点就是触点链接到自己的触点，那判断两个触点是否在同一连通分量，当前仅当由这个两个触点出发，能到达同一根触点。


### union
为了保证上述find逻辑，union(p,
q)的实现可以这样，分别找到p，q的根触点，将两个根触点链接，即实现了另一个分量的重命名，简单高效。
{{< highlight go "linenos=inline" >}}
func union(id *[]int, p, q int) {
    pRoot := find(*id, p)
    qRoot := find(*id, q)
    if(pRoot == qRoot) {
        return
    }
    (*id)[pRoot] = (*id)[qRoot]
}
{{< /highlight >}}
