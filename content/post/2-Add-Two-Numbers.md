---
title: "2 Add Two Numbers"
date: 2019-11-10T15:49:55+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
# 简述
<!--more-->
# 题目描述
[Add Two Numbers](https://leetcode.com/problems/add-two-numbers/)  
You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.

You may assume the two numbers do not contain any leading zero, except the number 0 itself.

Example:
{{< highlight go "linenos=inline" >}}
Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
Explanation: 342 + 465 = 807.
{{< /highlight >}}

# 思路分析
此题思路简单，注意条件即可，特别是最后的carry处理  
# 源码实现
{{< highlight go "linenos=inline" >}}
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
	var dummy ListNode
	cur := &dummy
	carry := 0
	for l1 != nil || l2 != nil || carry != 0 {
		val := carry
		if l1 != nil {
			val = val + l1.Val
			l1 = l1.Next
		}
		if l2 != nil {
			val = val + l2.Val
			l2 = l2.Next
		}
		carry = val / 10
		val = val % 10
		cur.Next = &ListNode{Val: val}
		cur = cur.Next
	}
	return dummy.Next
}
{{< /highlight >}}

