---
title: "76. Minimum Window Substring"
date: 2019-11-17T19:23:44+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
人为刀俎，我为鱼肉。《史记·项羽本纪》
<!--more-->
# 题目描述
[76. Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/)  
Given a string S and a string T, find the minimum window in S which will contain all the characters in T in complexity O(n).

Example:

{{< highlight go "linenos=inline" >}}
Input: S = "ADOBECODEBANC", T = "ABC"
Output: "BANC"
{{< /highlight >}}
Example:

{{< highlight go "linenos=inline" >}}
Input: S = "aa", T = "aa"
Output: "aa"

{{< /highlight >}}
Note:

If there is no such window in S that covers all characters in T, return the empty string "".  
If there is such window, you are guaranteed that there will always be only one unique minimum window in S.
# 思路分析
比较容易想到是的滑动窗口法，先开两个map用于字符统计，一个用于统计需要的字符，另一个用于统计滑动窗口中已有的字符，窗口一开始向右不断滑动，窗口中每增加一个必要的字符时，对应计数+1，同时一旦发现和该字符所需要的数相等时，匹配计数器+1，当右指针滑到满足所有字符要求数时，开始操作左指针右移缩小窗口，期间更新结果和窗口统计数，直到窗口不再满足条件为止才开始进行下一轮右指针移动。

# 源码实现
{{< highlight go "linenos=inline" >}}
func minWindow(s string, t string) string {
	needs := make(map[byte]int, len(t))
	window := make(map[byte]int, len(t))
	for i, _ := range t {
		needs[t[i]]++
	}
	minLen := len(s) + 1
	match := 0
	left, start := 0, 0
	for right, _ := range s {
		rchar := s[right]
		if _, ok := needs[rchar]; ok {
			window[rchar]++
			if window[rchar] == needs[rchar] {
				match++
			}
		}
		for match == len(needs) {
			windowLen := right - left + 1
			if windowLen < minLen {
				start = left
				minLen = windowLen
			}
			lchar := s[left]
			if _, ok := needs[lchar]; ok {
				window[lchar]--
			}
			if window[lchar] < needs[lchar] {
				match--
			}
			left++
		}
	}
	if minLen > len(s) {
		return ""
	}
	return s[start : start+minLen]
}
{{< /highlight >}}

