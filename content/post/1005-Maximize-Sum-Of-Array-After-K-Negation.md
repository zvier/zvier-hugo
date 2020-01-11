---
title: "1005 Maximize Sum of Array After K Negation"
date: 2019-11-10T22:24:21+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
# 简述
<!--more-->
# 题目描述
[1005. Maximize Sum Of Array After K Negations](https://leetcode.com/problems/maximize-sum-of-array-after-k-negations/)   
Given an array A of integers, we must modify the array in the following way: we choose an i and replace A[i] with -A[i], and we repeat this process K times in total.  (We may choose the same index i multiple times.)

Return the largest possible sum of the array after modifying it in this way.

{{< highlight go "linenos=inline" >}}
Example 1:

Input: A = [4,2,3], K = 1
Output: 5
Explanation: Choose indices (1,) and A becomes [4,-2,3].
{{< /highlight >}}
Example 2:

{{< highlight go "linenos=inline" >}}
Input: A = [3,-1,0,2], K = 3
Output: 6
Explanation: Choose indices (1, 2, 2) and A becomes [3,1,0,2].
{{< /highlight >}}
Example 3:

{{< highlight go "linenos=inline" >}}
Input: A = [2,-3,-1,5,-4], K = 2
Output: 13
Explanation: Choose indices (1, 4) and A becomes [2,3,-1,5,4].
{{< /highlight >}}
 

Note:

{{< highlight go "linenos=inline" >}}
1 <= A.length <= 10000
1 <= K <= 10000
-100 <= A[i] <= 100
{{< /highlight >}}

# 思路分析
题目简单，但有好几个需要注意的地方   
# 源码实现
{{< highlight go "linenos=inline" >}}
func largestSumAfterKNegations(A []int, K int) int {
    sum := 0
    // 计数
	counter := make(map[int]int, 201)
	for _, num := range A {
		counter[num]++
	}

	// 优先处理负数，并且尽可能让绝对值大的负数调整为负数，注意同一个负数肯能出现多次
	for n := -100; n < 0; n++ {
		cnt := counter[n]
		for cnt > 0 && K > 0 {
			counter[n]--
			counter[-n]++
			cnt--
			K--
		}
	}

	// 此时如果K还大于0的话，就找出最小的非负数反转直到K为0 
	if K > 0 {
		minPositive := 0
		for n := 0; n <= 100; n++ {
			if _, ok := counter[n]; ok {
				minPositive = n
				break
			}
		}
		symbol := -2*(K%2) + 1
        // 注意若反转，正数计数要-1，负数计数+1
		if symbol == -1 {
			counter[minPositive]--
			counter[-minPositive]++
		}
	}

    // 统计和
	for num, cnt := range counter {
		sum = sum + num*cnt
	}
	return sum
}
{{< /highlight >}}

