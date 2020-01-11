---
title: "104 Maximum Depth of Binary Tree"
date: 2019-12-29T08:24:14+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
近乡情更怯，不敢问来人。——宋之问《渡汉江》

<!--more-->
# 思路分析一
普通递归

# 源码实现一
执行用时 : 4 ms , 在所有 golang 提交中击败了 93.06% 的用户   
内存消耗 : 4.4 MB , 在所有 golang 提交中击败了 74.78% 的用户
{{< highlight go "linenos=inline" >}}
func maxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}
	leftMaxDepth := maxDepth(root.Left)
	rightMaxDepth := maxDepth(root.Right)
	maxDepth := leftMaxDepth + 1
	if rightMaxDepth > leftMaxDepth {
		maxDepth = rightMaxDepth + 1
	}
	return maxDepth
}
{{< /highlight >}}

# 思路分析二
尾递归

# 源码实现二
执行用时 : 4 ms , 在所有 golang 提交中击败了 93.06% 的用户  
内存消耗 : 4.4 MB , 在所有 golang 提交中击败了 57.39% 的用户
{{< highlight go "linenos=inline" >}}
func maxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}
	max := 0
	_maxDepth(root, 1, &max)
	return max
}

func _maxDepth(root *TreeNode, n int, max *int) {
	if root == nil {
		return
	}
	if root.Left == nil && root.Right == nil {
		if n > *max {
			*max = n
		}
		return
	}
	_maxDepth(root.Left, n+1, max)
	_maxDepth(root.Right, n+1, max)
}
{{< /highlight >}}
