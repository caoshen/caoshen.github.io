---
title: 统计重复个数
date: 2020-04-19 19:04:31
tags: 算法
---

# 统计重复个数

由 n 个连接的字符串 s 组成字符串 S，记作 S = [s,n]。例如，["abc",3]=“abcabcabc”。

如果我们可以从 s2 中删除某些字符使其变为 s1，则称字符串 s1 可以从字符串 s2 获得。例如，根据定义，"abc" 可以从 “abdbec” 获得，但不能从 “acbbe” 获得。

现在给你两个非空字符串 s1 和 s2（每个最多 100 个字符长）和两个整数 0 ≤ n1 ≤ 106 和 1 ≤ n2 ≤ 106。现在考虑字符串 S1 和 S2，其中 S1=[s1,n1] 、S2=[s2,n2] 。

请你找出一个可以满足使[S2,M] 从 S1 获得的最大整数 M 。

示例：

```
输入：
s1 ="acb",n1 = 4
s2 ="ab",n2 = 2

返回：
2
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/count-the-repetitions
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

先考虑 s2 在 S1 中出现的循环节，然后根据循环节中出现的 s1 和 s2 的个数计算得到 s2 出现的总数，最后除以 n2 得到 S2 出现的次数。

建议参考官方题解：https://leetcode-cn.com/problems/count-the-repetitions/solution/tong-ji-zhong-fu-ge-shu-by-leetcode-solution/

## 解答

```java
class Solution {
    public int getMaxRepetitions(String s1, int n1, String s2, int n2) {
        if (n1 == 0) {
            return 0;
        }
        // 寻找循环节过程中 s1 出现的个数
        int s1Count = 0;
        // 寻找循环节过程中 s2 出现的个数
        int s2Count = 0;
        // s2 的索引
        int index = 0;
        // 保存索引对应的 s1、s2 个数
        HashMap<Integer, int[]> recallMap = new HashMap<>();
        // 循环开始前的头部
        int[] preLoop = new int[2];
        // 循环节
        int[] inLoop = new int[2];

        while(true) {
            // 尝试下一个 s1 匹配
            s1Count++;
            // 遍历 s1
            for(int i = 0; i < s1.length(); ++i) {
                char c = s1.charAt(i);
                if (c == s2.charAt(index)) {
                    // 匹配到 s2 的一个字符
                    index++;
                    // 完全匹配到 s2
                    if (index == s2.length()) {
                        s2Count++;
                        index = 0;
                    }
                }
            }
            // S1 所有的 s1 都匹配完了，遍历完毕，返回 s2 的个数除以 s2 的乘数
            if (s1Count == n1) {
                return s2Count / n2;
            }
            // 出现了之前的循环节，表示找到了 s2 循环
            if (recallMap.containsKey(index)) {
                preLoop[0] = recallMap.get(index)[0];
                preLoop[1] = recallMap.get(index)[1];
                inLoop[0] = s1Count - preLoop[0];
                inLoop[1] = s2Count - preLoop[1];
                break;
            } else {
                // 没有找到循环节，记录 s1Count 和 s2Count 给下一次循环使用
                recallMap.put(index, new int[]{s1Count, s2Count});
            }
        }

        // ans 是 S1 包含的 s2 的数量，考虑之前的 preLoop 头部，和循环节 inLoop
        int ans = preLoop[1] + (n1 - preLoop[0]) / inLoop[0] * inLoop[1];

        // S1 的末尾还剩一些 s1，开始暴力匹配
        int rest = (n1 - preLoop[0]) % inLoop[0];
        // 剩余 rest 个 s1
        for (int j = 0; j < rest; ++j) {
            for(int i = 0; i < s1.length(); ++i) {
                char c = s1.charAt(i);
                if (c == s2.charAt(index)) {
                    // 匹配到 s2 的一个字符
                    index++;
                    // 完全匹配到 s2
                    if (index == s2.length()) {
                        ans++;
                        index = 0;
                    }
                }
            }
        }
        // S1 包含 ans 个 s2，那么 S1 包含 ans / n2 个 S2
        return ans / n2;
    }
}
```
