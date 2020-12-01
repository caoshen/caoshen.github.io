---
title: 两数相加 II
date: 2020-04-14 22:34:02
tags: 算法
categories: 算法
---

# 两数相加 II

给你两个 非空 链表来代表两个非负整数。数字最高位位于链表开始位置。它们的每个节点只存储一位数字。将这两数相加会返回一个新的链表。

你可以假设除了数字 0 之外，这两个数字都不会以零开头。

进阶：

如果输入链表不能修改该如何处理？换句话说，你不能对列表中的节点进行翻转。

示例：

```
输入：(7 -> 2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 8 -> 0 -> 7
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/add-two-numbers-ii
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

因为高位在链表开始位置，但是加法从低位开始，所以需要用栈保存两个链表的数。

## 解答

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    private Stack<Integer> s1 = new Stack<>();
    private Stack<Integer> s2 = new Stack<>();

    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        while(l1 != null) {
            s1.push(l1.val);
            l1 = l1.next;
        }
        while(l2 != null) {
            s2.push(l2.val);
            l2 = l2.next;
        }
        int addNum = 0;
        ListNode list = new ListNode(0);
        while (!s1.isEmpty() && !s2.isEmpty()) {
            int i1 = s1.pop();
            int i2 = s2.pop();
            int val = i1 + i2 + addNum;
            int remainder = val % 10;
            addNum = val / 10;
            ListNode temp = new ListNode(remainder);
            temp.next = list.next;
            list.next = temp;
        }
        while (!s1.isEmpty()) {
            int i1 = s1.pop();
            int val = i1 + addNum;
            int remainder = val % 10;
            addNum = val / 10;
            ListNode temp = new ListNode(remainder);
            temp.next = list.next;
            list.next = temp;
        }
        while (!s2.isEmpty()) {
            int i2 = s2.pop();
            int val = i2 + addNum;
            int remainder = val % 10;
            addNum = val / 10;
            ListNode temp = new ListNode(remainder);
            temp.next = list.next;
            list.next = temp;
        }
        if (addNum > 0) {
            ListNode temp = new ListNode(addNum);
            temp.next = list.next;
            list.next = temp;
        }
        return list.next;
    }
}
```
