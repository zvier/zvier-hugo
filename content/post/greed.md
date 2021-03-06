---
title: "贪婪算法"
date: 2019-10-16T19:54:29+08:00
draft: true
categories: ["技术"]
tags: ["算法"]
---
# 简述
<!--more-->
# 找零问题  
已知有d1=25, d2=10, d3=5, d4=1四种硬币面额，请用最少的硬币数找出金额为n=48的零钱。面对这个问题，我们可以比较轻松的给出答案: 1个d1+2个d2+3个d1=48。这里其实我们就潜意识贪婪地以最佳选择序列策略，从几种可能中确定一种最优方案。贪婪的想法促使我们在做人生的每一步选择中，都优先尽可能利用面值更大的硬币，以把余下的数量降大最低。

# 贪婪算法的核心
贪婪算法的核心在于它建议通过一系列步骤来构造问题的解，每一步都对目前构造的问题求解，直到获得问题的完整解。当然，步步最优，也并不意味着全局最优，所以为了能获取全局最优，每一步都必须满足下面条件:
1. 可行: 即每求解一步，都必须满足问题的约束。  
2. 局部最优: 每一步求解，都是贪婪地选择局部最优，寄希望于通过一系列的局部最优解，产生全局最优解。    
3. 不可取消: 每一步求解，一旦作出决定，在算法的后续步骤中，无法返回改变。   

贪婪算法的难点在于判断问题确实可以通过步步最优得到全局最优。
# 参考
1. 算法设计与分析基础
