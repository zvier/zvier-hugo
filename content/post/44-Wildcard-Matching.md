---
title: "44 Wildcard Matching"
date: 2019-11-03T13:13:12+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
<!--more-->
# 题目描述
[44. Wildcard Matching](https://leetcode.com/problems/wildcard-matching/)  

Given an input string (s) and a pattern (p), implement wildcard pattern matching with support for '?' and '*'.

'?' Matches any single character.
'*' Matches any sequence of characters (including the empty sequence).
The matching should cover the entire input string (not partial).

Note:

s could be empty and contains only lowercase letters a-z.
p could be empty and contains only lowercase letters a-z, and characters like ? or *.
Example 1:
{{< highlight go "linenos=inline" >}}

Input:
s = "aa"
p = "a"
Output: false
Explanation: "a" does not match the entire string "aa".
{{< /highlight >}}
Example 2:
{{< highlight go "linenos=inline" >}}
Input:
s = "aa"
p = "*"
Output: true
Explanation: '*' matches any sequence.
{{< /highlight >}}
Example 3:

{{< highlight go "linenos=inline" >}}
Input:
s = "cb"
p = "?a"
Output: false
Explanation: '?' matches 'c', but the second letter is 'a', which does not match 'b'.
{{< /highlight >}}
Example 4:
{{< highlight go "linenos=inline" >}}
Input:
s = "adceb"
p = "*a*b"
Output: true
Explanation: The first '*' matches the empty sequence, while the second '*' matches the substring "dce".
{{< /highlight >}}
Example 5:
{{< highlight go "linenos=inline" >}}

Input:
s = "acdcb"
p = "a*c?b"
Output: false
{{< /highlight >}}
# 思路
对于类似这种字符串与字符串的匹配问题，动态规划是求解的一大神器，通过构造一个二维矩阵，很容易推导出一个状态转移方程。

这里我们定义一个dp二维数组，dp[i][j]表示s中前i个字符和p中前j个字符是否匹配。dp大小为(len(s)+1) x (len(p) +1)，以便处理dp[0][0]，也就是s，p均为空的情况，通过扩展数组外围也是一个比较常用的技巧。

另外，对于dp数组的第0行，表示s为空，此时对于p的每一个字符，若值为'*'，则dp[0][i]=dp[0][i-1]，所有在正式动态规划之前，我们需要考虑这两个情况，初始化dp。对于正式的动态规划，判断s的前i+1个字符是否与p的前j+1个字符匹配，有如下情况:

1. 若遇到*，因为*能匹配空字符串或任意长度字符串，那只要s的i+1个字符与p的前j个字符匹配，s的i+1个字符与p的j+1个字符也必然匹配，若s的前i个字符与p的j+1个字符匹配，那s的前i+1个字符必然也匹配都行
2. 若遇到?，则看s的前i个字符是否与p的前j个字符匹配即可
3.  若遇到普通字母，则除了s的前i个字符必须与p的前j个字符匹配，还需要当前待匹配的两个字符想等
# 算法实现
{{< highlight go "linenos=inline" >}}
func isMatch(s string, p string) bool {
	n := len(s)
	m := len(p)
	dp := make([][]bool, 0, n+1)
	for i := 0; i <= n; i++ {
		row := make([]bool, m+1)
		dp = append(dp, row)
	}
	dp[0][0] = true
	for j, r := range p {
		if r == '*' {
			dp[0][j+1] = dp[0][j]
		}
	}

	for i, _ := range s {
		for j, _ := range p {
			if p[j] == '*' {
				dp[i+1][j+1] = dp[i+1][j] || dp[i][j+1]
			}
			if p[j] == '?' {
				dp[i+1][j+1] = dp[i][j]
			}
			if p[j] != '*' && p[j] != '?' {
				dp[i+1][j+1] = dp[i][j] && s[i] == p[j]
			}
		}
	}
	return dp[n][m]
}
{{< /highlight >}}

