---
title: "438 Find All Anagrams in a String"
date: 2019-11-17T21:26:53+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
大行不顾细谨，大礼不辞小让。《史记·项羽本纪》
<!--more-->
# 题目描述
[438. Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/)  
Given a string s and a non-empty string p, find all the start indices of p's anagrams([ˈænəɡræm], 易位构词，变位词) in s.

Strings consists of lowercase English letters only and the length of both strings s and p will not be larger than 20,100.

The order of output does not matter.

Example 1:
{{< highlight go "linenos=inline" >}}
Input:
s: "cbaebabacd" p: "abc"

Output:
[0, 6]

Explanation:
The substring with start index = 0 is "cba", which is an anagram of "abc".
The substring with start index = 6 is "bac", which is an anagram of "abc".
{{< /highlight >}}

Example 2:
{{< highlight go "linenos=inline" >}}
Input:
s: "abab" p: "ab"

Output:
[0, 1, 2]

Explanation:
The substring with start index = 0 is "ab", which is an anagram of "ab".
The substring with start index = 1 is "ba", which is an anagram of "ab".
The substring with start index = 2 is "ab", which is an anagram of "ab".
{{< /highlight >}}

# 思路分析
滑动窗口法，开两个字符计数map，一个计数所需字符，另一个计数滑动窗口字符，一开始滑动窗口右指针右移向右扩张，当滑动窗口字符拥有所需字符以及字符数时，这里还需要判断一下长度是否合适，因为是异位构词，满足条件则将起始索引加入到结果列表，最后进一步移动左指针，收缩滑动窗口，如此往复。  

# 源码实现

{{< highlight go "linenos=inline" >}}
func findAnagrams(s string, p string) []int {
	needs := make(map[byte]int, len(p))
	window := make(map[byte]int, len(p))
	for i, _ := range p {
		needs[p[i]]++
	}

	result := make([]int, 0, len(p))
	left, match := 0, 0
	for right, _ := range s {
		rchar := s[right]
		if _, ok := needs[rchar]; ok {
			window[rchar]++
			if window[rchar] == needs[rchar] {
				match++
			}
		}

		for match == len(needs) {
			if right-left+1 == len(p) {
				result = append(result, left)
			}
			lchar := s[left]
			if _, ok := needs[lchar]; ok {
				window[lchar]--
				// 一旦窗口内左边界字符数小于所需数时，则对应字符数匹配失效，match数-1
				// 当前左边界右移循环也会跳出，因为窗口已经不能满足所需，需要开始下一轮向右扩大窗口
				if window[lchar] < needs[lchar] {
					match--
				}
			}
			left++
		}
	}
	return result
}
{{< /highlight >}}
