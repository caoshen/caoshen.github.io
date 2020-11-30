---
title: 圆圈中最后剩下的数字
date: 2020-03-30 22:59:36
tags: 算法
categories: 算法
---

# 圆圈中最后剩下的数字

0,1,,n-1这n个数字排成一个圆圈，从数字0开始，每次从这个圆圈里删除第m个数字。求出这个圆圈里剩下的最后一个数字。

例如，0、1、2、3、4这5个数字组成一个圆圈，从数字0开始每次删除第3个数字，则删除的前4个数字依次是2、0、4、1，因此最后剩下的数字是3。

示例 1：

```
输入: n = 5, m = 3
输出: 3
```

示例 2：

```
输入: n = 10, m = 17
输出: 2
```

限制：

1 <= n <= 10^5
1 <= m <= 10^6

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

需要用倒推的思路解决。最后一个数的下标一定是 0，如果 m = 3，那么上一轮有2个数备选的情况下这个数的位置是1，以此类推。

## 解答

```java
class Solution {
    public int lastRemaining(int n, int m) {
        // 最后一轮得到的数字的下标是 0，只有一个
        int ans = 0;
        // 开始反推 lastIndex = (curIndex + m) % i，i 是当前这一轮的剩余的个数。从剩余2人的情况开始
        for (int i = 2; i <= n; ++i) {
            ans = (ans + m) % i;
        }
        return ans;
    }
}
```
