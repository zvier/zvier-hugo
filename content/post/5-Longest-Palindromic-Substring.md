---
title: "5"
date: 2019-11-16T20:09:10+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
<!--more-->
# 题目描述
[5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)  
Given a string s, find the longest palindromic substring in s. You may assume that the maximum length of s is 1000.

Example 1:

{{< highlight go "linenos=inline" >}}
Input: "babad"
Output: "bab"
Note: "aba" is also a valid answer.
Example 2:
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
Input: "cbbd"
Output: "bb"
{{< /highlight >}}
# 思路分析
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
# 源码实现
{{< highlight go "linenos=inline" >}}
func isPalindrome(s string) bool {
	if len(s) == 0 || len(s) == 1 {
		return true
	}
	if s[0] != s[len(s)-1] {
		return false
	}
	return isPalindrome(s[1 : len(s)-1])
}

func longestPalindrome(s string) string {
	result := ""
	maxLen := 0
	startIndex := 0
	for i := 0; i <= len(s)-1; i++ {
		palindromeLen := 0
		if startIndex-1 >= 0 && s[startIndex-1] == s[i] {
			startIndex = startIndex - 1
		} else {
			for j := startIndex; j <= i; j++ {
				if s[j] == s[i] && isPalindrome(s[j:i+1]) {
					startIndex = j
					break
				}
			}
		}
		palindromeLen = i - startIndex + 1
		if palindromeLen > maxLen {
			maxLen = palindromeLen
			result = s[startIndex : i+1]
		}
	}
	return result
}
{{< /highlight >}}

