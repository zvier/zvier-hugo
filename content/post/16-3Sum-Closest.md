---
title: "16 3Sum Closest"
date: 2019-12-09T06:14:46+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
白兔捣药秋复春，嫦娥孤栖与谁邻。今人不见古时月，今月曾经照古人。  
<!--more-->
# 题目描述
[16. 最接近的三数之和](https://leetcode-cn.com/problems/3sum-closest)
给定一个包括 n 个整数的数组 nums 和 一个目标值 target。找出 nums 中的三个整数，使得它们的和与 target 最接近。返回这三个数的和。假定每组输入只存在唯一答案。

{{< highlight go "linenos=inline" >}}
例如，给定数组 nums = [-1，2，1，-4], 和 target = 1.

与 target 最接近的三个数的和为 2. (-1 + 2 + 1 = 2).
{{< /highlight >}}

# 思路分析
同三数之和问题，先排序，固定一个元素，开双指针分别指向固定元素后的数组前后，按条件移动指针直到满足最接近，如果差值为0可以直接返回

# 源码实现
{{< highlight go "linenos=inline" >}}
func threeSumClosest(nums []int, target int) int {
	quicksort(&nums, 0, len(nums)-1)
	// 初始化结果和最小差值
	result := nums[0] + nums[1] + nums[2]
	min := abs(result - target)
	for i := 0; i < len(nums)-1-1; i++ {
		l, r := i+1, len(nums)-1
		for l < r {
			sum := nums[i] + nums[l] + nums[r]
			diff := sum - target
			absValue := abs(diff)
			// 差值小于已记录的最小差值，更新
			if absValue < min {
				min = absValue
				result = sum
			}
			if diff > 0 {
				r--
			}
			if diff < 0 {
				l++
			}
			if diff == 0 {
				return result
			}
		}
	}
	return result
}

func abs(num int) int {
	if num < 0 {
		return -num
	}
	return num
}

func quicksort(pnums *[]int, lo, high int) {
	if lo >= high {
		return
	}
	j := partition(pnums, lo, high)
	quicksort(pnums, lo, j-1)
	quicksort(pnums, j+1, high)
}

func partition(pnums *[]int, lo, high int) int {
	nums := *pnums
	l, r := lo, high
	for {
		for nums[l] <= nums[lo] && l < high {
			l++
		}
		for nums[r] >= nums[lo] && r > lo {
			r--
		}
		if l >= r {
			break
		}
		swap(pnums, l, r)
	}
	swap(pnums, r, lo)
	return r
}

func swap(pnums *[]int, i, j int) {
	nums := *pnums
	tmp := nums[i]
	nums[i] = nums[j]
	nums[j] = tmp
}
{{< /highlight >}}

