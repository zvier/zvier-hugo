---
title: "3 Longest Substring Without Repeating Characters"
date: 2019-11-16T20:58:58+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
高城鼓动兰釭灺，睡也还醒，醉也还醒，忽听孤鸿三两声。人生只似风前絮，欢也飘零，悲也飘零，都作连江点点萍。——王国维《采桑子》
<!--more-->
# 题目描述
[3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)   
Given a string, find the length of the longest substring without repeating characters.

Example 1:

{{< highlight go "linenos=inline" >}}
Input: "abcabcbb"
Output: 3
Explanation: The answer is "abc", with the length of 3.
{{< /highlight >}}
Example 2:

{{< highlight go "linenos=inline" >}}
Input: "bbbbb"
Output: 1
Explanation: The answer is "b", with the length of 1.
{{< /highlight >}}
Example 3:

{{< highlight go "linenos=inline" >}}
Input: "pwwkew"
Output: 3
Explanation: The answer is "wke", with the length of 3.
             Note that the answer must be a substring, "pwke" is a subsequence and not a substring.
{{< /highlight >}}
# 思路分析
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
# 源码实现一
{{< highlight go "linenos=inline" >}}
func lengthOfLongestSubstring(s string) int {
    dp := make(map[int]int, len(s))
	max := 0
	for i := 0; i <= len(s)-1; i++ {
		if i == 0 {
			max = 1
			dp[0] = 0
			continue
		}

		start := dp[i-1]
		j := i - 1
		for ; j >= start; j-- {
			if s[j] == s[i] {
				break
			}
		}
		j++
		dp[i] = j
		length := i - j + 1
		if length > max {
			max = length
		}
	}
	return max
}
{{< /highlight >}}

# 源码实现
{{< highlight go "linenos=inline" >}}
func lengthOfLongestSubstring(s string) int {
	mp := make(map[byte]int, len(s))
	dp := make([]int, len(s))
	maxLen := 0
	for i := 0; i <= len(s)-1; i++ {
		lastLongStrStartIndex := 0
		if i > 0 {
			lastLongStrStartIndex = dp[i-1]
		}
		start := lastLongStrStartIndex
		if lastAppearIndex, ok := mp[s[i]]; ok {
			if lastAppearIndex >= start {
				start = lastAppearIndex + 1
			}
		}
		dp[i] = start

		length := i - start + 1
		if length > maxLen {
			maxLen = length
		}
		mp[s[i]] = i
	}
	return maxLen
}
{{< /highlight >}}
