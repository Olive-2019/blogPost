---
title: 多状态DP：买卖股票的最佳时机 III
date: 2023-09-30 10:57:29
tags:
---
一道很有意思的DP题目
<!-- more -->

# 题目
[leetcode链接：买卖股票的最佳时机 III](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iii/)
# 题解
根据题意可以将买卖股票划分成五个状态
1. 完全没交易
2. 买了第一次
3. 卖了第一次，还没买第二次
4. 买了第二次
5. 卖了第二次
这五个状态可以从上向下转换，将2-4编号为buy1 sell1 buy2 sell2，可以得出，状态转移方程
```
buy1 = max(buy1, -prices[i]);
sell1 = max(buy1 + prices[i], sell1);
buy2 = max(sell1 - prices[i], buy2);
sell2 = max(sell2, buy2 + prices[i]);
```
这个题有意思的点就在于状态多，识别状态有点困难
# 代码
```c
class Solution {
public:
    int max(int a, int b) {
        if (a > b) return a;
        return b;
    }
    int maxProfit(vector<int>& prices) {
        int buy1 = -prices[0], buy2 = buy1, sell1 = 0, sell2 = 0;
        for (int i = 0; i < prices.size(); ++i) {
            buy1 = max(buy1, -prices[i]);
            sell1 = max(buy1 + prices[i], sell1);
            buy2 = max(sell1 - prices[i], buy2);
            sell2 = max(sell2, buy2 + prices[i]);
        }
        return max(0, max(sell1, sell2));
    }
};
```
# 升级版题目
## 题目
[leetcode链接：买卖股票的最佳时机 IV](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iv/)
## 题解
类似于上一题，将不同状态进一步抽象为数组表示
## 代码
```c
class Solution {
public:
    int max(int a, int b) {
        if (a < b) return b;
        return a;
    }
    int maxProfit(int k, vector<int>& prices) {
        vector<int> buy(k + 1, -prices[0]), sell(k + 1, 0);
        int ans = 0;
        for (int i = 0; i < prices.size(); ++i) {
            buy[0] = max(buy[0], -prices[i]);
            for (int j = 1; j <= k; ++j) {
                buy[j] = max(buy[j], sell[j - 1] - prices[i]);
                sell[j] = max(sell[j], buy[j] + prices[i]);
                ans = max(ans, sell[j]);
            }
        }
        return ans;
    }
};
```