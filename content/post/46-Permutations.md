---
title: "46_Permutations"
date: 2019-12-11T21:38:54+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---

<!--more-->
# 题目描述
给定一个没有重复数字的序列，返回其所有可能的全排列。

示例:
{{< highlight go "linenos=inline" >}}
输入: [1,2,3]
输出:
[
  [1,2,3],
  [1,3,2],
  [2,1,3],
  [2,3,1],
  [3,1,2],
  [3,2,1]
]
{{< /highlight >}}

# 思路分析
枚举所有解一般可用回溯法不断探索，如果在求解过程中可用确定不是解，回溯到上一步，抛弃该路径，并改变策略做下一次尝试。

回溯法通常可以构造一个树形空间，比如这里可以确定一个根节点，为原始数组，然后向下发枝生长，生长的条件就是用0号元素分别与数组中的每个元素交换，如此生长出[1,2,3], [2,1,3], [3,2,1]三个子节点，紧接着，这些子节点还可以进一步生长，用1号元素分别与数组后面的元素交换，依次往后，直到生长完毕，找到问题的解。  
2， 3，

# 源码实现
{{< highlight go "linenos=inline" >}}
func permute(nums []int) [][]int {
        result := make([][]int, 0, len(nums))
        // 初始状态，第0次生长，将第0号元素分别与其它元素交换
        backtrace(0, &nums, &result)
        return result
}

func backtrace(step int, nums *[]int, result *[][]int) {
        // 为了获得一个解，生长到len(*num)次就无需再生长了，此时是收获的季节，将得到的结果收割
        if step == len(*nums) {
                dst := make([]int, len(*nums))
                // 注意切片的拷贝
                copy(dst, *nums)
                *result = append(*result, dst)
        }
        // 如果还没到收获的季节，将当前元素分别与后面元素交换，并继续生长
        for i := step; i <= len(*nums)-1; i++ {
                (*nums)[step], (*nums)[i] = (*nums)[i], (*nums)[step]
                backtrace(step+1, nums, result)
                // 复原现场，回溯尝试下一个分支
                (*nums)[step], (*nums)[i] = (*nums)[i], (*nums)[step]
        }
}
{{< /highlight >}}

