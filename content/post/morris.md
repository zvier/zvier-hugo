---
title: "morris遍历算法"
date: 2020-01-11T13:37:06+08:00
draft: true
categories: ["技术"]
tags: ["算法"]
---
<!--more-->
# 简述
morris遍历适用于二叉树遍历，与递归、非递归遍历相比，递归遍历会产生一个O(n)的递归栈空间，而非递归遍历空间复杂度也是O(n)，morris遍历利用树的叶子节点为空的特点，可以将空间复杂度降为O(1)，而时间复杂度保持O(N)。

# 算法
要使用O(1)空间进行二叉树的遍历，最大的难点在于遍历到子节点时，如何重新回到父节点，为此，morris遍历的本质就是通过建立一种机制，利用叶子节点中的左右指针，指向某种顺序遍历下的前驱节点或后继节点，从而对于没有左子树的节点只到达一次，有左子树的节点到达两次，具体算法描述如下:

# morris中序遍历
1. 如果当前节点左孩子为空，输出当前节点，当前节点更新为当前节点的右孩子
{{< highlight go >}}
cur = cur.right
{{< /highlight >}}
2. 如果cur有左孩子，则在其左子树中找到当前节点在中序遍历下的前趋节点(左子树中的最右节点)
3. 如果前驱节点的右孩子为空，将它的有孩子设置为当前节点，当前节点更新为当前节点的左孩子
{{< highlight go >}}
cur = cur.left
{{< /highlight >}}
4. 如果前驱节点的右孩子为当前节点，将它的右孩子重新设置为空，恢复原二叉树形状，并输出当前节点，当前节点更新为当前节点的右孩子
{{< highlight go >}}
cur = cur.right
{{< /highlight >}}
5. 如果cur无左右孩子，遍历结束

以下为morris中序列遍历源码实现
{{< highlight go "linenos=inline">}}
func inorderTraversal(root *TreeNode) []int {
	result := make([]int, 0, 2)
	cur := root
	for cur != nil {
		if cur.Left == nil {
			result = append(result, cur.Val)
			cur = cur.Right
		} else {
			prev := cur.Left
			for prev.Right != nil && prev.Right != cur {
				prev = prev.Right
			}
			if prev.Right == nil {
				prev.Right = cur
				cur = cur.Left
			}
			if prev.Right == cur {
				prev.Right = nil
				result = append(result, cur.Val)
				cur = cur.Right
			}
		}
	}
	return result
}
{{< /highlight >}}


