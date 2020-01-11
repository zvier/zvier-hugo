---
title: "208 Implement Trie"
date: 2019-12-08T16:25:25+08:00
draft: true
categories: ["技术"]
tags: ["leetcode"]
---
赵客缦胡缨，吴钩霜雪明。银鞍照白马，飒沓如流星。——李白《侠客行》
<!--more-->
# 题目描述
[208. 实现 Trie (前缀树)](https://leetcode-cn.com/problems/implement-trie-prefix-tree/)   
实现一个 Trie (前缀树)，包含 insert, search, 和 startsWith 这三个操作。

示例:

{{< highlight go "linenos=inline" >}}
Trie trie = new Trie();

trie.insert("apple");
trie.search("apple");   // 返回 true
trie.search("app");     // 返回 false
trie.startsWith("app"); // 返回 true
trie.insert("app");
trie.search("app");     // 返回 true
说明:

你可以假设所有的输入都是由小写字母 a-z 构成的。
保证所有输入均为非空字符串。
{{< /highlight >}}

# 思路分析
前缀树实现，childs字段可以开map也可以开数组实现，题目中只有插入和查询操作，故尾节点只需要标示即可。  

# 源码实现
{{< highlight go "linenos=inline" >}}
type Trie struct {
	// 也可以换成数组
	childs map[byte]*Trie
	isWord bool
}

/** Initialize your data structure here. */
func Constructor() Trie {
	t := Trie{
		childs: make(map[byte]*Trie),
	}
	return t
}

/** Inserts a word into the trie. */
func (this *Trie) Insert(word string) {
	cursor := this
	// 遍历每一个元素，查看相应节点是否存在，不存在则创建
	for i := 0; i <= len(word)-1; i++ {
		next := cursor.childs[word[i]]
		if next == nil {
			next = &Trie{
				childs: make(map[byte]*Trie),
			}
			cursor.childs[word[i]] = next
		}
		cursor = next
	}
	// 标记是一个word
	cursor.isWord = true
}

/** Returns if the word is in the trie. */
func (this *Trie) Search(word string) bool {
	cursor := this
	for i := 0; i <= len(word)-1; i++ {
		next := cursor.childs[word[i]]
		if next == nil {
			return false
		}
		cursor = next
	}
	if !cursor.isWord {
		return false
	}
	return true
}

/** Returns if there is any word in the trie that starts with the given prefix. */
func (this *Trie) StartsWith(prefix string) bool {
	cursor := this
	for i := 0; i <= len(prefix)-1; i++ {
		next := cursor.childs[prefix[i]]
		if next == nil {
			return false
		}
		cursor = next
	}
	return true
}
{{< /highlight >}}

