---
title: 反转链表
date: 2020-05-16 23:27:30
tags: 链表
categories: 算法
---

# 反转链表

定义一个函数，输入一个链表的头节点，反转该链表并输出反转后链表的头节点。

示例:

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

限制：

0 <= 节点个数 <= 5000

注意：本题与主站 206 题相同：https://leetcode-cn.com/problems/reverse-linked-list/

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

采用头插法将当前节点的下一个节点依次插入到头部。构造一个假头节点方便插入。

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
    public ListNode reverseList(ListNode head) {
        if (head == null) {
            return head;
        }
        ListNode p = head;
        // 假头节点
        ListNode h = new ListNode(0);
        h.next = p;
        while (p.next != null) {
            // 将当前节点的下一个节点插入到链表头部
            ListNode q = p.next;
            p.next = q.next;
            q.next = h.next;
            h.next = q;
        }
        return h.next;
    }
}
```
