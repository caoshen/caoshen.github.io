---
title: 多数元素
date: 2020-03-12 23:10:25
tags: 算法
categories: 算法
---

# 多数元素

给定一个大小为 n 的数组，找到其中的多数元素。多数元素是指在数组中出现次数大于 ⌊ n/2 ⌋ 的元素。

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

示例 1:

```
输入: [3,2,3]
输出: 3
```

示例 2:

```
输入: [2,2,1,1,1,2,2]
输出: 2
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/majority-element
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

比较容易想到的方法是用一个Map保存每个数和它出现的次数，然后取出现次数最多的那一个数。

也可以先对数组排序，如果有一个数出现次数大于 ⌊ n/2 ⌋ ，那么排序后数组中间那个数就会是次数最多的数。

另外一种方法是先选取一个候选者，如果它后面的数和它相同，那么候选次数加一，否则候选次数减一。当候选次数变为0就换一个候选者。因为题目中的数组总是存在多数元素，所以最后一个候选者就是要求的数。

[官方题解](https://leetcode-cn.com/problems/majority-element/solution/duo-shu-yuan-su-by-leetcode-solution/)

## 解答

方法一：

```java
class Solution {
    public int majorityElement(int[] nums) {
        Map<Integer, Integer> countMap = new HashMap<>();

        for (int i : nums) {
            Integer count = countMap.get(i);
            if (count == null) {
                countMap.put(i, 1);
            } else {
                countMap.put(i, count + 1);
            }
        }
        int maxCount = 0;
        int maxIndex = 0;
        for (Map.Entry<Integer, Integer> entry : countMap.entrySet()) {
            if (entry.getValue() > maxCount) {
                maxCount = entry.getValue();
                maxIndex = entry.getKey();
            }
        }
        return maxIndex;
    }
}
```

方法二：

```java
class Solution {
    public int majorityElement(int[] nums) {
        int count = 0;
        int candidate = 0;
        for (int n : nums) {
            if (count == 0) {
                candidate = n;
            }
            if (n == candidate) {
                count++;
            } else {
                count--;
            }
        }
        return candidate;
    }
}
```
