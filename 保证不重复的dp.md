---
title: 保证不重复的dp
date: 2023-10-04 09:43:30
tags:
    - 算法
    - DP
---
一道通过dp遍历顺序保证不重复的题目，咋一看很简单，细看有点难
<!-- more -->
# 题目
(leetcode 518. 零钱兑换 II)[https://leetcode.cn/problems/coin-change-ii/solutions/821278/ling-qian-dui-huan-ii-by-leetcode-soluti-f7uh/]
# 题解
这个题很容易想到dp，但是普通的dp一般是外层遍历金额，内层遍历coins。这么做的问题是会重复计数，例如：coins=[1,2] amount = 3，当计算3时，会算到3种[1+1+1,2+1,1+2]，但是后两种是重复的。
为了保证计数不重复，外层遍历的是coins，内层才是金额，不断的加加加。
不重复计数原因：当前的coin一定是之前没算过的
# 代码
```c++
class Solution {
public:  
    int change(int amount, vector<int>& coins) {
        vector<int> dp(amount + 1, 0);
        dp[0] = 1;
        for (int coin : coins) {
            for (int i = coin; i <= amount; ++i) {
                dp[i]+=dp[i - coin];
            }
        }
        return dp[amount];
    }
};
```