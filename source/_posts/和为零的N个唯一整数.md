# 和为零的N个唯一整数

给你一个整数 n，请你返回 任意 一个由 n 个 各不相同 的整数组成的数组，并且这 n 个数相加和为 0 。

 

示例 1：

```
输入：n = 5
输出：[-7,-1,1,3,4]
解释：这些数组也是正确的 [-5,-1,1,2,3]，[-3,-1,2,-2,4]。
```

示例 2：

```
输入：n = 3
输出：[-1,0,1]
```

示例 3：

```
输入：n = 1
输出：[0]
```

提示：

1 <= n <= 1000

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/find-n-unique-integers-sum-up-to-zero
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

如果 n = 1，返回 0。

如果 n 是偶数，返回成对的整数，比如 1/-1。

如果 n 是奇数，返回 0 和成对的整数，比如 0、1/-1。

## 解答

```java
class Solution {
    public int[] sumZero(int n) {
        if (n == 1) {
            return new int[]{0};
        }
        if ((n & 1) == 0) {
            int[] result = new int[n];
            for (int i = 0; i < n; i += 2) {
                int val = i + 1;
                result[i] = val;
                result[i + 1] = -val;
            }
            return result;
        } else {
            int[] result = new int[n];
            result[0] = 0;
            for (int i = 1; i < n; i += 2) {
                int val = i + 1;
                result[i] = val;
                result[i + 1] = -val;
            }
            return result;
        }

    }
}
```