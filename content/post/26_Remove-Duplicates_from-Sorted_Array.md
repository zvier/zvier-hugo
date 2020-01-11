---
title: "26 Remove Duplicates from Sorted Array"
date: 2019-12-07T22:00:49+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
秋风清，秋月明，落叶聚还散，寒鸦栖复惊。相思相见知何日？此时此夜难为情。——李白《秋风词》
<!--more-->
# 题目描述
给定一个排序数组，你需要在原地删除重复出现的元素，使得每个元素只出现一次，返回移除后数组的新长度。

不要使用额外的数组空间，你必须在原地修改输入数组并在使用 O(1) 额外空间的条件下完成。

示例 1:

{{< highlight go "linenos=inline" >}}
给定数组 nums = [1,1,2],

函数应该返回新的长度 2, 并且原数组 nums 的前两个元素被修改为 1, 2。

你不需要考虑数组中超出新长度后面的元素。
{{< /highlight >}}
示例 2:

{{< highlight go "linenos=inline" >}}
给定 nums = [0,0,1,1,1,2,2,3,3,4],

函数应该返回新的长度 5, 并且原数组 nums 的前五个元素被修改为 0, 1, 2, 3, 4。

你不需要考虑数组中超出新长度后面的元素。
{{< /highlight >}}

# 思路分析——双指针
开双指针，一个指向已删除重复的最后一个元素，一个指向当前待处理的元素，如果待处理元素的值与已去重的最后一个元素不等，那加入到已去重数组，更新其指针，然后继续处理下一个元素。  

时间复杂度：O(n)，假设数组的长度是 n，那么i和j分别最多走n步。空间复杂度：O(1)


# 源码实现
{{< highlight go "linenos=inline" >}}
func removeDuplicates(nums []int) int {
	i := 0
	for j := 1; j <= len(nums)-1; j++ {
		if nums[j] != nums[i] {
			i++
			nums[i] = nums[j]
		}
	}
	return i + 1
}
{{< /highlight >}}


# 思路分析——优化
如果待处理元素的值与已去重的最后一个元素不等，且这两个元素相邻，没必要移动

# 源码实现
{{< highlight go "linenos=inline" >}}
func removeDuplicates(nums []int) int {
	i := 0
	for j := 1; j <= len(nums)-1; j++ {
		if nums[j] != nums[i] {
			i++
			// 注意i经过+1处理，已经指向待更新位置
			if j-i > 0 {
				nums[i] = nums[j]
			}
		}
	}
	return i + 1
}
{{< /highlight >}}

# 思路分析——另一个角度
前面思路都是将待处理元素与已保留元素最后一个比较，另外一种方式就是与前一个元素比较，因为不管已保留部分怎么被更新，前一个元素总是和原始数组中的前一个元素相同

# 源码实现
{{< highlight go "linenos=inline" >}}
func removeDuplicates(nums []int) int {
	i := 1
	for j := 1; j <= len(nums)-1; j++ {
		if nums[j] != nums[j-1] {
			if j-i > 0 {
				nums[i] = nums[j]
			}
			i++
		}
	}
	return i
}
{{< /highlight >}}
