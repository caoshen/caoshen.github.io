---
title: 寻找峰值
date: 2019-12-11 23:51:05
tags:
  - 数组
  - 递归
categories: 算法
---

# 寻找峰值

峰值元素是指其值大于左右相邻值的元素。

给定一个输入数组 nums，其中 nums[i] ≠ nums[i+1]，找到峰值元素并返回其索引。

数组可能包含多个峰值，在这种情况下，返回任何一个峰值所在位置即可。

你可以假设 nums[-1] = nums[n] = -∞。

示例 1:

```
输入: nums = [1,2,3,1]
输出: 2
解释: 3 是峰值元素，你的函数应该返回其索引 2。
```

示例 2:

```
输入: nums = [1,2,1,3,5,6,4]
输出: 1 或 5 
解释: 你的函数可以返回索引 1，其峰值元素为 2；
     或者返回索引 5， 其峰值元素为 6。
```

说明:

你的解法应该是 O(logN) 时间复杂度的。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/find-peak-element
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 解答

首先想到的是顺序遍历，但是顺序遍历的时间复杂度是 O(N)，不满足要求。虽然也能通过用例。

```java
class Solution {
    public static int findPeakElement(int[] nums) {
        if (nums == null) {
            return 0;
        }
        if (nums.length == 0 || nums.length == 1) {
            return 0;
        }

        for (int i = 0; i < nums.length; ++i) {
            if (i == 0) {
                if (nums[0] > nums[1]) {
                    return 0;
                }
            } else if (i == nums.length - 1) {
                if (nums[i - 1] < nums[i]) {
                    return i;
                }
            } else {
                int left = nums[i - 1];
                int right = nums[i + 1];
                int mid = nums[i];
                if (left < mid && mid > right) {
                    return i;
                }
            }
        }
        return 0;
    }
}
```

 题目要求 O(logN) 时间复杂度，那么一般是树状结构的遍历，比如二叉树。

如果中间节点是峰值，直接返回中间节点。否则取左右节点中较大的那个节点，在它所属的区间内遍历。

```java
class Solution {
    public static int findPeakElement(int[] nums) {
        if (nums == null) {
            return 0;
        }
        if (nums.length == 0 || nums.length == 1) {
            return 0;
        }
        if (nums.length == 2) {
            return nums[0] > nums[1] ? 0 : 1;
        }
        return findPeakElementRange(nums, 0, nums.length - 1);
    }

    public static int findPeakElementRange(int[] nums, int left, int right) {
        if (left >= right) {
            return left;
        }
        if (left + 1 == right) {
            return nums[left] > nums[right] ? left : right;
        }
        int total = left + right;
        int mid = ((total & 1) == 1) ? (total - 1) / 2: total / 2;
        if (nums[mid - 1] < nums[mid] && nums[mid] > nums[mid + 1]) {
            return mid;
        } else if (nums[mid - 1] < nums[mid + 1]) {
            return findPeakElementRange(nums, mid + 1, right);
        } else {
            return findPeakElementRange(nums, left, mid - 1);
        }
    }
}
```