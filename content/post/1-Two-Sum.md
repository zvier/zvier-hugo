---
title: "1 Two Sum"
date: 2019-11-17T10:31:50+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
项庄舞剑，意在沛公。《史记·项羽本纪》
<!--more-->

# 题目描述
[1. Two Sum](https://leetcode.com/problems/two-sum/)  
Given an array of integers, return indices of the two numbers such that they add up to a specific target.

You may assume that each input would have exactly one solution, and you may not use the same element twice.

Example:

{{< highlight go "linenos=inline" >}}
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
{{< /highlight >}}

# 思路分析
## 方法一: 暴力求解
此题很简单，最容易想到的莫过于遍历每个元素x，然后在剩余元素中查找是否存在target-x，该法时间复杂度为O(n^2)，空间复杂度O(1)。

## 方法二: 哈希表辅助+两次遍历
上面暴力求解过程中，如果先用哈希表辅助存储各个数的位置，在获取target-x就可以在O(1)时间内快速定位到目标位置，整体时间复杂度可降到O(n)，不过空间复杂度上来了，为O(n)

## 方法二: 哈希表辅助+一次遍历
一遍构建哈希表，一遍查看，代码更简洁  
# 源码实现
{{< highlight go "linenos=inline" >}}
func towSum(nums []int, target int) []int {
	mp := make(map[int]int, len(nums))
	for i, num := range nums {
		mp[num] = i
	}

	for ia, num := range nums {
		diff := target - num
		ib, ok := mp[diff]
		if ok && ib != ia {
			return []int{ia, ib}
		}
	}
	return nil
}

{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func twoSum(nums []int, target int) []int {
	mp := make(map[int]int, len(nums))
	for i, num := range nums {
		diff := target - num
		j, ok := mp[diff]
		if ok {
			return []int{i, j}
		}
		mp[num] = i
	}
	return nil
}
{{< /highlight >}}
