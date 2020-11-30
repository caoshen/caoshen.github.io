---
title: 两棵二叉搜索树中的所有元素
date: 2019-12-30 00:25:50
tags:
  - 算法
  - 树
categories: 算法
---

# 两棵二叉搜索树中的所有元素

给你 root1 和 root2 这两棵二叉搜索树。

请你返回一个列表，其中包含 两棵树 中的所有整数并按 升序 排序。.

 

示例 1：

```
输入：root1 = [2,1,4], root2 = [1,0,3]
输出：[0,1,1,2,3,4]
```

示例 2：

```
输入：root1 = [0,-10,10], root2 = [5,1,7,0,2]
输出：[-10,0,0,1,2,5,7,10]
```

示例 3：

```
输入：root1 = [], root2 = [5,1,7,0,2]
输出：[0,1,2,5,7]
```

示例 4：

```
输入：root1 = [0,-10,10], root2 = []
输出：[-10,0,10]
```

示例 5：


```
输入：root1 = [1,null,8], root2 = [8,1]
输出：[1,1,8,8]
```


提示：

每棵树最多有 5000 个节点。
每个节点的值在 [-10^5, 10^5] 之间。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/all-elements-in-two-binary-search-trees
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

首先对两个树做先序遍历，会得到从小到大排序的 2 个有序数组。然后对两个有序数组进行合并。

## 解答

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public List<Integer> getAllElements(TreeNode root1, TreeNode root2) {
        List<Integer> list1 = preOrderTree(root1);
        List<Integer> list2 = preOrderTree(root2);
        return mergeOrderList(list1, list2);
    }

    public static List<Integer> mergeOrderList(List<Integer> list1, List<Integer> list2) {
        List<Integer> resultList = new ArrayList<>();
        if (list1 == null && list2 == null) {
            return resultList;
        }
        if (list1 == null) {
            return list2;
        }
        if (list2 == null) {
            return list1;
        }
        int index1 = 0;
        int index2 = 0;
        while (index1 < list1.size() && index2 < list2.size()) {
            Integer val1 = list1.get(index1);
            Integer val2 = list2.get(index2);
            if (val1 < val2) {
                resultList.add(val1);
                index1++;
            } else {
                resultList.add(val2);
                index2++;
            }
        }
        while (index1 < list1.size()) {
            resultList.add(list1.get(index1));
            index1++;
        }
        while (index2 < list2.size()) {
            resultList.add(list2.get(index2));
            index2++;
        }
        return resultList;
    }

    public static List<Integer> preOrderTree(TreeNode node) {
        List<Integer> preOrderList = new ArrayList<>();
        // stop condition
        if (node == null) {
            return preOrderList;
        }
        if (node.left == null && node.right == null) {
            preOrderList.add(node.val);
            return preOrderList;
        }
        // recursive get order list
        List<Integer> left = preOrderTree(node.left);
        preOrderList.addAll(left);
        preOrderList.add(node.val);
        List<Integer> right = preOrderTree(node.right);
        preOrderList.addAll(right);
        return preOrderList;
    }

}
```