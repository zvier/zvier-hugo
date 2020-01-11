---
title: "56.合并区间"
date: 2019-12-01T21:41:36+08:00
draft: true
categories: [""]
tags: ["leetcode"]
---
<!--more-->
过平山堂下，半生弹指声中。十年不见老仙翁，壁上龙蛇飞动。欲吊文章太守，仍歌杨柳春风。休言万事转头空，未转头时皆梦。 ——苏轼《西江月·平山堂》

# 题目描述
[56. 合并区间](!https://leetcode-cn.com/problems/merge-intervals/submissions/)  
给出一个区间的集合，请合并所有重叠的区间。

示例 1:

{{< highlight go "linenos=inline" >}}
输入: [[1,3],[2,6],[8,10],[15,18]]
输出: [[1,6],[8,10],[15,18]]
解释: 区间 [1,3] 和 [2,6] 重叠, 将它们合并为 [1,6].
{{< /highlight >}}
示例 2:

{{< highlight go "linenos=inline" >}}
输入: [[1,4],[4,5]]
输出: [[1,5]]
解释: 区间 [1,4] 和 [4,5] 可被视为重叠区间。
{{< /highlight >}}


# 思路分析——排序
先按start排序，然后逐一合并可合并的区间

# 源码实现一
{{< highlight go "linenos=inline" >}}
func merge(intervals [][]int) [][]int {
	// 一段选择排序
	for i := 0; i < len(intervals); i++ {
		minIndex := i
		for j := i + 1; j <= len(intervals)-1; j++ {
			if intervals[j][0] < intervals[minIndex][0] {
				minIndex = j
			}
		}
		tmp := intervals[i]
		intervals[i] = intervals[minIndex]
		intervals[minIndex] = tmp
	}

	result := make([][]int, 0, len(intervals))
	// 遍历每一个区间，如果当前为空，之间将区间加入
	for _, interval := range intervals {
		if len(result) == 0 {
			result = append(result, interval)
			continue
		}

		// 如果当前区间的起始小于或等于上一个区间的结束，必然存在重叠，需要合并
		lastInterval := result[len(result)-1]
		if interval[0] <= lastInterval[1] {
			// 合并时，结束值取上一个区间和当前区间结束值的最大值
			maxEnd := max(lastInterval[1], interval[1])
			result[len(result)-1][1] = maxEnd
		} else {
			// 如果当前区间超脱上一个区间的结束，那直接当前区间append到结果中
			result = append(result, interval)
		}
	}
	return result
}

func max(num1, num2 int) int {
	if num1 > num2 {
		return num1
	}
	return num2
}
{{< /highlight >}}

# 源码实现一
{{< highlight go "linenos=inline" >}}
func merge(intervals [][]int) [][]int {
	// 来个快排
	quicksort(&intervals, 0, len(intervals)-1)
	result := make([][]int, 0, len(intervals))
	for _, interval := range intervals {
		if len(result) == 0 {
			result = append(result, interval)
			continue
		}

		lastInterval := result[len(result)-1]
		if interval[0] <= lastInterval[1] {
			maxEnd := max(lastInterval[1], interval[1])
			result[len(result)-1][1] = maxEnd
		} else {
			result = append(result, interval)
		}
	}
	return result
}

func quicksort(pintervals *[][]int, lo, hi int) {
	// lo>=hi，表示排序区间小于或等于一个元素，无需再排了
	if lo >= hi {
		return
	}
    // 划分区间，划分点左边小于或等于划分点，右边大于或等于划分点
	j := partition(pintervals, lo, hi)
	// 对划分点两边的区间继续排序
	quicksort(pintervals, lo, j-1)
	quicksort(pintervals, j+1, hi)
}

func partition(pintervals *[][]int, lo, hi int) int {
	intervals := *pintervals
	// 设定左右两个游标
	i, j := lo, hi
	for {
		// 从左边往右找到第一个大于轴的元素，如果i==hi时，无需继续找了
        // 如果hi位置的元素大于或等于轴，因为时区间最后一个元素，不管
		// 如果hi位置的元素小于轴，会被下面的j找出来，最后被交换掉
		for intervals[i][0] <= intervals[lo][0] && i < hi {
			i++
		}
		// 从右边往左找到第一个小于轴的元素，如果j==lo时，可以停止找了
		for intervals[j][0] >= intervals[lo][0] && j > lo {
			j--
		}
        // 如果i和j开始交叉了，划分完成
		if i >= j {
			break
		}
		// 只要i和j没有交叉，那交换这样的两个元素
		swap(pintervals, i, j)
	}
	// 将轴和第一次交叉时的元素交换，停止左右扫描后的区间，i指向的是小于或等于轴的元素
	// j指向的是大于或等于轴的元素，所以只能将轴与j元素交互
	swap(pintervals, lo, j)
	return j
}

func swap(pintervals *[][]int, i, j int) {
	intervals := *pintervals
	tmp := intervals[i]
	intervals[i] = intervals[j]
	intervals[j] = tmp
}
{{< /highlight >}}
