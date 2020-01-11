---
title: "8. String to Integer(atoi)"
date: 2019-12-15T08:05:03+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
愿我如星君如月，夜夜流光相皎洁。——范成大《车遥遥篇》
<!--more-->

# 题目描述
[8. 字符串转换整数 (atoi)](https://leetcode-cn.com/problems/string-to-integer-atoi/)  
请你来实现一个 atoi 函数，使其能将字符串转换成整数。

首先，该函数会根据需要丢弃无用的开头空格字符，直到寻找到第一个非空格的字符为止。

当我们寻找到的第一个非空字符为正或者负号时，则将该符号与之后面尽可能多的连续数字组合起来，作为该整数的正负号；假如第一个非空字符是数字，则直接将其与之后连续的数字字符组合起来，形成整数。

该字符串除了有效的整数部分之后也可能会存在多余的字符，这些字符可以被忽略，它们对于函数不应该造成影响。

注意：假如该字符串中的第一个非空格字符不是一个有效整数字符、字符串为空或字符串仅包含空白字符时，则你的函数不需要进行转换。

在任何情况下，若函数不能进行有效的转换时，请返回 0。

说明：

假设我们的环境只能存储 32 位大小的有符号整数，那么其数值范围为 [−231,  231 − 1]。如果数值超过这个范围，请返回  INT_MAX (231 − 1) 或 INT_MIN (−231) 。

示例 1:

{{< highlight go "linenos=inline" >}}
输入: "42"
输出: 42
{{< /highlight >}}
示例 2:

{{< highlight go "linenos=inline" >}}
输入: "   -42"
输出: -42
解释: 第一个非空白字符为 '-', 它是一个负号。
     我们尽可能将负号与后面所有连续出现的数字组合起来，最后得到 -42 。
{{< /highlight >}}
示例 3:

{{< highlight go "linenos=inline" >}}
输入: "4193 with words"
输出: 4193
解释: 转换截止于数字 '3' ，因为它的下一个字符不为数字。
{{< /highlight >}}
示例 4:

{{< highlight go "linenos=inline" >}}
输入: "words and 987"
输出: 0
解释: 第一个非空字符是 'w', 但它不是数字或正、负号。
     因此无法执行有效的转换。
{{< /highlight >}}
示例 5:

{{< highlight go "linenos=inline" >}}
输入: "-91283472332"
输出: -2147483648
解释: 数字 "-91283472332" 超过 32 位有符号整数范围。
     因此返回 INT_MIN (−231) 。
{{< /highlight >}}

# 思路分析
此题看似简单，实则还是有好几个需要注意的地方和技巧，对于用户输入要考虑全面

# 源码实现
{{< highlight go "linenos=inline" >}}
func myAtoi(str string) int {
	// 跳过前缀空白字符
	i := 0
	for ; i <= len(str)-1; i++ {
		if str[i] != ' ' {
			break
		}
	}

	// 这里注意跳过空白后，字符串是否还有剩余，没有则及时退出
	// 类似，对于前一步处理，是循环遍历完退出还是break退出，注意区别是否有不同
	if i == len(str) {
		return 0
	}

	// 如果有符号位，处理符号位
	sign := 1
	if str[i] == '+' || str[i] == '-' {
		if str[i] == '-' {
			sign = -1
		}
		i++
	}

	// 处理数据部分，这里正负同样处理，所以注意构造边界值
	result := 0
	limit := (1 << 31) - (sign+1)/2
	digitLimit := (1-sign)/2 + 7
	divLimit := limit / 10
	for ; i <= len(str)-1; i++ {
		if str[i] < '0' || str[i] > '9' {
			break
		}
		digit := int(str[i] - '0')
		// 注意用result和边界值除以10去比较，result*10可能超限
		// 如果result等于边界值，那就看各位是否会超限，同时也注意正负的各位边界不一样
		// 1<<31 = 2147483648
		if result < divLimit || (result == divLimit && digit <= digitLimit) {
			result = result*10 + digit
		} else {
			return sign * limit
		}
	}
	return sign * result
}
{{< /highlight >}}

