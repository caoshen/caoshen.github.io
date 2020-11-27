---
title: 正则表达式匹配
date: 2020-06-20 21:36:40
tags: 算法
categories: 算法
---

# 正则表达式匹配

给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

```
'.' 匹配任意单个字符
'*' 匹配零个或多个前面的那一个元素
```

所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。

说明:

s 可能为空，且只包含从 a-z 的小写字母。
p 可能为空，且只包含从 a-z 的小写字母，以及字符 . 和 *。

示例 1:

```
输入:
s = "aa"
p = "a"
输出: false
解释: "a" 无法匹配 "aa" 整个字符串。
```

示例 2:

```
输入:
s = "aa"
p = "a*"
输出: true
解释: 因为 '*' 代表可以匹配零个或多个前面的那一个元素, 在这里前面的元素就是 'a'。因此，字符串 "aa" 可被视为 'a' 重复了一次。
```

示例 3:

```
输入:
s = "ab"
p = ".*"
输出: true
解释: ".*" 表示可匹配零个或多个（'*'）任意字符（'.'）。
```

示例 4:

```
输入:
s = "aab"
p = "c*a*b"
输出: true
解释: 因为 '*' 表示零个或多个，这里 'c' 为 0 个, 'a' 被重复一次。因此可以匹配字符串 "aab"。
```

示例 5:

```
输入:
s = "mississippi"
p = "mis*is*p*."
输出: false
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/regular-expression-matching
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

因为形如 .* 的字符串可以匹配无数类型的字符串，所以使用动态规划的方法，通过上一段字符串的匹配情况推导出当前字符串是否能匹配。

## 解答

```java
class Solution {
    public boolean isMatch(String s, String p) {
        int lenSrc = s.length();
        int lenPattern = p.length();
        // 注意这里的状态方程保存了 (lenSrc + 1) * (lenPatth + 1)种，因为需要考虑空字符串的比较
        boolean[][] f = new boolean[lenSrc + 1][lenPattern + 1];
        // 两个空字符串是匹配的
        f[0][0] = true;

        // 计算两个字符串在各个位置的状态。
        // 需要考虑源字符串为空串，而正则字符串不为空的场景，所以 i 从 0 开始。
        // 而源字符串不为空，但是正则字符串为空，这种场景是匹配不上的，属于无效状态，所以 j 从 1 开始。
        for (int i = 0; i <= lenSrc; ++i) {
            for (int j = 1; j <= lenPattern; ++j) {
                // p[j] 是一个字母
                if (p.charAt(j - 1) != '*') {
                    // 比较 p 和 s 最后一个字符
                    if (isCharMatch(s, p, i, j)) {
                        f[i][j] = f[i - 1][j - 1];
                    } else {
                        f[i][j] = false;
                    }
                } else {
                    // p[j] 是 *
                    // 假设 p[j - 1] 是 a，可以把 * 算作 0 次字符，那么 p[j - 1 : j] 变为空串，只需比较 p[j - 2]
                    f[i][j] = f[i][j - 2];
                    // 如果 s[i] == p[j - 1]，可以在 i 维度上进一步动态规划。假设 s[i] = a，p[j - 1] = a，p[j -2] = *，那么 a* 可以匹配无数个 a(a..a)，既然 s[i] == p[j - 1]，可以进一步匹配 s[i - 1] 和 p[j - 1]，依次向前类推。
                    if (isCharMatch(s, p, i, j - 1)) {
                        // 在 f[i][j] 是 false 的情况下，进一步考察 f[i - 1][j] 是否为 true。
                        f[i][j] = f[i][j] || f[i - 1][j];
                    }
                }
            }
        }

        return f[lenSrc][lenPattern];
    }

    private boolean isCharMatch(String s, String p, int i, int j) {
        if (i == 0) {
            // s 是空串
            return false;
        }
        if (p.charAt(j - 1) == '.') {
            return true;
        }
        return s.charAt(i - 1) == p.charAt(j - 1);
    }
}
```
