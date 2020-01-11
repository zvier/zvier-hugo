---
title: "505 迷宫II"
date: 2019-11-24T16:19:53+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
试问岭南应不好，却道：此心安处是吾乡。——苏轼《定风波·南海归赠王定国侍人寓娘》
<!--more-->
# 题目描述
[505. 迷宫 II](https://leetcode-cn.com/problems/the-maze-ii)  
由空地和墙组成的迷宫中有一个球。球可以向上下左右四个方向滚动，但在遇到墙壁前不会停止滚动。当球停下时，可以选择下一个方向。  
给定球的起始位置，目的地和迷宫，找出让球停在目的地的最短距离。距离的定义是球从起始位置（不包括）到目的地（包括）经过的空地个数。如果球无法停在目的地，返回 -1。  
迷宫由一个0和1的二维数组表示。 1表示墙壁，0表示空地。你可以假定迷宫的边缘都是墙壁。起始位置和目的地的坐标通过行号和列号给出。

示例 1:
{{< highlight go "linenos=inline" >}}
输入 1: 迷宫由以下二维数组表示

0 0 1 0 0
0 0 0 0 0
0 0 0 1 0
1 1 0 1 1
0 0 0 0 0

输入 2: 起始位置坐标 (rowStart, colStart) = (0, 4)
输入 3: 目的地坐标 (rowDest, colDest) = (4, 4)

输出: 12

解析: 一条最短路径 : left -> down -> left -> down -> right -> down -> right。
             总距离为 1 + 1 + 3 + 1 + 2 + 2 + 2 = 12。

{{< /highlight >}}
示例 2:
{{< highlight go "linenos=inline" >}}
输入 1: 迷宫由以下二维数组表示

0 0 1 0 0
0 0 0 0 0
0 0 0 1 0
1 1 0 1 1
0 0 0 0 0

输入 2: 起始位置坐标 (rowStart, colStart) = (0, 4)
输入 3: 目的地坐标 (rowDest, colDest) = (3, 2)

输出: -1

解析: 没有能够使球停在目的地的路径。
{{< /highlight >}}

注意:

迷宫中只有一个球和一个目的地。
球和目的地都在空地上，且初始时它们不在同一位置。
给定的迷宫不包括边界 (如图中的红色矩形), 但你可以假设迷宫的边缘都是墙壁。
迷宫至少包括2块空地，行数和列数均不超过100。

# 思路分析一——深度优先搜索
坐标题的遍历往往都是上下左右四个方向走，设定计数器记录从起始位置到停留点[i, j]的最小步数，若非更小，则没必要继续。现在从起始位置开始，朝一个方向径直走，每遇到到一个节点计数加1，如果碰到墙壁，考虑转弯

# 源码描述
{{< highlight go "linenos=inline" >}}
func shortestDistance(maze [][]int, start []int, destination []int) int {
    // 初始化二维计数器
	n, m := len(maze), len(maze[0])
	maxDistance := n * m
	steps := make([][]int, 0, len(maze))
	for _, rmaze := range maze {
		row := make([]int, 0, len(rmaze))
		for i := 0; i <= len(rmaze)-1; i++ {
			row = append(row, maxDistance)
		}
		steps = append(steps, row)
	}
	steps[start[0]][start[1]] = 0

    // 从起始点开始访问
	visit(&maze, &steps, start)
	step := steps[destination[0]][destination[1]]
	if step == maxDistance {
		return -1
	}
	return step
}

func visit(pmaze, psteps *[][]int, start []int) {
	maze := *pmaze
	steps := *psteps

    // 设定四个方向，对于每一个暂停点，四个方向都要跑遍
	dirs := [][]int{[]int{0, 1}, []int{0, -1}, []int{-1, 0}, []int{1, 0}}
	for _, dir := range dirs {
		i := start[0] + dir[0]
		j := start[1] + dir[1]
		count := 0
		// if not hit the wall, walk straight，径直走，直到遇到墙壁
		for i >= 0 && j >= 0 && i <= len(maze)-1 && j <= len(maze[0])-1 && maze[i][j] == 0 {
			i += dir[0]
			j += dir[1]
			count++
		}

		newDistance := steps[start[0]][start[1]] + count
		// 注意此时[i, j]已经碰壁，暂停点坐标要回退一步
		oldDistance := steps[i-dir[0]][j-dir[1]]
		// 如果距离更小，更小计数器，重新访问
		if newDistance < oldDistance {
			(*psteps)[i-dir[0]][j-dir[1]] = newDistance
			visit(pmaze, psteps, []int{i - dir[0], j - dir[1]})
		}
	}
}
{{< /highlight >}}

# 思路分析二——深度优先搜索优化
上面深度优先搜索还可以继续做个小优化，计数器其实可以就地计数，没必要再开一个二维数组，代码量和内存使用都可以进一步减少。

# 源码实现二
{{< highlight go "linenos=inline" >}}
func shortestDistance(maze [][]int, start []int, destination []int) int {
	n, m := len(maze), len(maze[0])
	maxDistance := n * m
	baseSteps := 2
	for i, rmaze := range maze {
		for j := 0; j <= len(rmaze)-1; j++ {
			if maze[i][j] != 1 {
				maze[i][j] = maxDistance
			}
		}
	}
	maze[start[0]][start[1]] = baseSteps

	visit(&maze, start)
	step := maze[destination[0]][destination[1]]
	if step == maxDistance {
		return -1
	}
	return step - baseSteps
}

func visit(pmaze *[][]int, start []int) {
	maze := *pmaze

	dirs := [][]int{[]int{0, 1}, []int{0, -1}, []int{-1, 0}, []int{1, 0}}
	for _, dir := range dirs {
		i := start[0] + dir[0]
		j := start[1] + dir[1]
		count := 0
		// if not hit the wall, walk straight
		for i >= 0 && j >= 0 && i <= len(maze)-1 && j <= len(maze[0])-1 && maze[i][j] != 1 {
			i += dir[0]
			j += dir[1]
			count++
		}

		newDistance := maze[start[0]][start[1]] + count
		oldDistance := maze[i-dir[0]][j-dir[1]]
		if newDistance < oldDistance {
			(*pmaze)[i-dir[0]][j-dir[1]] = newDistance
			visit(pmaze, []int{i - dir[0], j - dir[1]})
		}
	}
}
{{< /highlight >}}
