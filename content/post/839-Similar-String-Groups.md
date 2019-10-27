---
title: "839 Similar String Groups"
date: 2019-10-19T20:12:22+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
一轮明月已上林梢，渐觉风生袖底，月到波心，俗虑尘怀，爽然顿释。——沈复《浮生六记》
<!--more-->
# 题目描述
Two strings X and Y are similar if we can swap two letters (in different positions) of X, so that it equals Y.

For example, "tars" and "rats" are similar (swapping at positions 0 and 2), and "rats" and "arts" are similar, but "star" is not similar to "tars", "rats", or "arts".

Together, these form two connected groups by similarity: {"tars", "rats", "arts"} and {"star"}.  Notice that "tars" and "arts"are in the same group even though they are not similar. Formally, each group is such that a word is in the group if and only if it is similar to at least one other word in the group.

We are given a list A of unique strings. Every string in A is an anagram([ˈænəɡræm] n. 相同字母异序词，易位构词，变位词) of every other string in A.  How many groups are there?

Example 1:

Input: ["tars","rats","arts","star"]
Output: 2
Note:

A.length <= 2000
A[i].length <= 1000
A.length * A[i].length <= 20000
All words in A consist of lowercase letters only.
All words in A have the same length and are anagrams of each other.
The judging time limit has been increased for this question.

# 题目解释
相似字符串组问题。题目首先定义了字符串之间的一种相似关系：对于字符串X和Y，如果通过交换X中两个不同位置上的字符，就可以得到Y的话，那X和Y是相似的。

现给定一个字符串数组，字符串组中的元素都是同系列字符异构而成，先要将相似的字符串放到一个群组里，同一个群组里的字符串不必任意两个都相似，只要能通过某些结点最终连着就行，有点像连通图觉，将所有连通的结点算作一个群组，问整个数组可以分为多少个群组。

比如例子中给定的字符串数组["tars","rats","arts","star"]，"tars"和"rats"相似，"rats"和"arts"相似，于是可以将"tars"，"rats"，"arts"放置到一个群组，而"star"和谁都不相似，单独放在另一个群组，于是最终形成了2个群组。  

# 算法实现
# 相似字符串判断
根据定义，由于字符串都是同源字符异构组成，那相似字符串当且仅当字符串中两个位置字符不同。
{{< highlight go "linenos=inline" >}}
func isSimilar(str1, str2 string) bool {
    cnt := 0
    for i, _ := range str1 {
        if str1[i] == str2[i] {
            continue
        }
        cnt++
        if cnt > 2 {
            return false
        }
    }
    return cnt == 2
}
{{< /highlight >}}

# DFS实现 
## 主逻辑
DFS方式就是在遍历原始序列时，针对每一个元素，最大深度地遍历到低，然后边遍历边标记。我们可以用一个map类型的变量visited来保存这种标记，将原始列表，当前待遍历的元素，以及map都传给一个visit辅助函数，当然在visit前，可以check一下是否已经遍历，已经遍历过的就直接跳过。
{{< highlight go "linenos=inline" >}}
func numSimilarGroups(A []string) int {
    visited := make(map[string]bool, len(A))
    res := 0
    for _, str := range A {
        if _, ok := visited[str]; ok {
            continue
        }
        res++
        visit(A, str, visited)
    }
    return res
}
{{< /highlight >}}

## 辅助函数
辅助函数要实现对元素的深度递归访问，值得注意的是，在访问元素时，由于还要深度遍历，那range时，也要注意元素是否被访问过

{{< highlight go "linenos=inline" >}}
func visit(input []string, str string, visited map[string]bool) {
    visited[str] = true
    for _, instr := range input {
        if _, ok := visited[instr]; ok {
            continue
        }
        if isSimilar(instr, str) {
            visit(input, instr, visited)
        }
    }
}
{{< /highlight >}}

# BFS实现
BFS的实现需要借助队列，在遍历原始数组时，先将待遍历的元素加入到queue队列，然后不断遍历这个队列直到为空，我们在访问队列中的元素时，会宽度处理，如果遇到同类元素，也将其加入到对列，这样一次访问下来，就可以形成一个组。
## 主逻辑
{{< highlight go "linenos=inline" >}}
func numSimilarGroups(A []string) int {
	visited := make(map[string]bool, len(A))
	res := 0
	queue := make([]string, 0, len(A))
	for _, str := range A {
		if _, ok := visited[str]; ok {
			continue
		}
		visited[str] = true
		res++
		queue = append(queue, str)
		for len(queue) > 0 {
			curstr := queue[0]
			queue = queue[1:]
			visit(A, &queue, curstr, visited)
		}
	}
	return res
}
{{< /highlight >}}

# Union-Find
本题本质上就是一个动态连通性问题，非常适合用Union-Find算法求解释。以下实用朴素Union-Find算法实现

## Union
Union函数实现触点x，y的连通，即归并到同一个连通分量，将触点所代表的集合合并
{{< highlight go "linenos=inline" >}}
func union(id *[]int, x, y int) {
	ix := find(*id, x)
	iy := find(*id, y)
	if ix == iy {
		return
	}

	for i, _ := range *id {
		if (*id)[i] == iy {
			(*id)[i] = ix
		}
	}
}
{{< /highlight >}}

## Find
find函数的功能就是查找触点i所在的连通分量
{{< highlight go "linenos=inline" >}}
func find(id []int, i int) int {
	return id[i]
}
{{< /highlight >}}

## numSimilarGroups
主逻辑，先初始化触点，然后归并，最后统计
{{< highlight go "linenos=inline" >}}
func numSimilarGroups(A []string) int {
	// init
	id := make([]int, len(A), len(A))
	for i, _ := range A {
		id[i] = i
	}

	for i, _ := range A {
		for j := i + 1; j < len(A); j++ {
			if !isSimilar(A[i], A[j]) {
				continue
			}
			union(&id, i, j)
		}
	}

	// statistics the count of group after union-find
	mp := make(map[int]bool, len(A))
	for _, v := range id {
		mp[v] = true
	}
	res := len(mp)
	return res
}

{{< /highlight >}}
