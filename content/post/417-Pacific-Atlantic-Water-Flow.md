---
title: "417 Pacific Atlantic Water Flow"
date: 2019-11-03T07:35:06+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
眼看他起朱楼，眼看他宴宾客，眼看他楼塌了。这青苔碧瓦堆，俺曾睡风流觉，将五十年兴亡看饱。金粉未消亡，闻得六朝香，满天涯烟草断人肠。怕催花信紧，风风雨雨，误了春光。 ——孔尚任《桃花扇》
<!--more-->

# 题目描述
Given an m x n matrix of non-negative integers representing the height of each unit cell in a continent, the "Pacific ocean" touches the left and top edges of the matrix and the "Atlantic ocean" touches the right and bottom edges.

Water can only flow in four directions (up, down, left, or right) from a cell to another one with height equal or lower.

Find the list of grid coordinates where water can flow to both the Pacific and Atlantic ocean.

Note:

The order of returned grid coordinates does not matter.
Both m and n are less than 150.


Example:

{{< highlight go "linenos=inline" >}}
Given the following 5x5 matrix:

  Pacific ~   ~   ~   ~   ~
       ~  1   2   2   3  (5) *
       ~  3   2   3  (4) (4) *
       ~  2   4  (5)  3   1  *
       ~ (6) (7)  1   4   5  *
       ~ (5)  1   1   2   4  *
          *   *   *   *   * Atlantic

Return:

[[0, 4], [1, 3], [1, 4], [2, 2], [3, 0], [3, 1], [4, 0]] (positions with parentheses in above matrix).
{{< /highlight >}}

[417. Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow/)

# 题目解析
给定二维数组，注意数组左边界和上边界都是太平洋，右边界和下边界都是大西洋，路径连通性问题，典型的图搜素，dfs或bfs都行。可以分别从太平洋和大西洋出发，标记出能到达的点，最后求交集，便是要求的坐标。  

为了标记已访问的点或者可到达的点，个人习惯原地标记，不必要开辟新的二维数组作为原数组的画像，当然前提是输入都是正数，所以这里预处理下，所有数据都加1。

# DFS代码实现
## 主逻辑
{{< highlight go "linenos=inline" >}}
func pacificAtlantic(matrix [][]int) [][]int {
	if len(matrix) == 0 {
		return nil
	}

	preProcess(&matrix)
	pResult := make([][]int, 0, len(matrix))
	aResult := make([][]int, 0, len(matrix))

	for i := 0; i <= len(matrix[0])-1; i++ {
		visit(&matrix, 0, i, &pResult)
	}
	for j := 1; j <= len(matrix)-1; j++ {
		visit(&matrix, j, 0, &pResult)
	}

	reset(&matrix)
	for i := 0; i <= len(matrix[0])-1; i++ {
		visit(&matrix, len(matrix)-1, i, &aResult)
	}
	for j := 0; j < len(matrix)-1; j++ {
		visit(&matrix, j, len(matrix[0])-1, &aResult)
	}

	reset(&matrix)
	result := insection(matrix, aResult, pResult)
	return result
}
{{< /highlight >}}

## 预处理
{{< highlight go "linenos=inline" >}}
func preProcess(matrix *[][]int) {
	m := *matrix
	for i, row := range m {
		for j, _ := range row {
			m[i][j] = m[i][j] + 1
		}
	}
}
{{< /highlight >}}

## 求交集
{{< highlight go "linenos=inline" >}}
func insection(matrix [][]int, a, b [][]int) [][]int {
	result := make([][]int, 0, len(a))
	for _, va := range a {
		x := va[0]
		y := va[1]
		matrix[x][y] = -matrix[x][y]
	}
	for _, vb := range b {
		x := vb[0]
		y := vb[1]
		if matrix[x][y] < 0 {
			result = append(result, vb)
		}
	}
	return result
}
{{< /highlight >}}

## 遍历
{{< highlight go "linenos=inline" >}}
func visit(matrix *[][]int, i, j int, result *[][]int) {
	m := *matrix
	if m[i][j] < 0 {
		return
	}
	cur := m[i][j]
	m[i][j] = -m[i][j]
	*result = append(*result, []int{i, j})
	if i-1 >= 0 && m[i-1][j] >= cur {
		visit(matrix, i-1, j, result)
	}

	if i+1 <= len(m)-1 && m[i+1][j] >= cur {
		visit(matrix, i+1, j, result)
	}

	if j-1 >= 0 && m[i][j-1] >= cur {
		visit(matrix, i, j-1, result)
	}

	if j+1 <= len(m[0])-1 && m[i][j+1] >= cur {
		visit(matrix, i, j+1, result)
	}
}
{{< /highlight >}}

## 复位
{{< highlight go "linenos=inline" >}}
func reset(matrix *[][]int) {
	m := *matrix
	for i, row := range m {
		for j, _ := range row {
			if m[i][j] < 0 {
				m[i][j] = -m[i][j]
			}
		}
	}
}
{{< /highlight >}}

## BFS代码实现
BFS解法需要借助queue 来辅助，一开始把边上的点分别加入queue中，然后对应的map 标记 true，然后开始 BFS 遍历，遍历结束后还是找 pacific 和 atlantic 均标记为 true 的点加入结果 res 中返回即可
