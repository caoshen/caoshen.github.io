---
title: 另一个树的子树
date: 2020-05-12 23:51:19
tags: 算法
---

# 另一个树的子树

给定两个非空二叉树 s 和 t，检验 s 中是否包含和 t 具有相同结构和节点值的子树。s 的一个子树包括 s 的一个节点和这个节点的所有子孙。s 也可以看做它自身的一棵子树。

示例 1:

给定的树 s:

```
     3
    / \
   4   5
  / \
 1   2
```

给定的树 t：

```
   4 
  / \
 1   2
```

返回 true，因为 t 与 s 的一个子树拥有相同的结构和节点值。

示例 2:

给定的树 s：

```
     3
    / \
   4   5
  / \
 1   2
    /
   0
```

给定的树 t：

```
   4
  / \
 1   2
```

返回 false。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/subtree-of-another-tree
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

常规解法是暴力匹配原始树和目标树是否相同。但是这种方法时间复杂度高，因为需要依次比对两棵树上的所有节点。

另一种方法是将两棵树转化为字符串，然后判断是不是子字符串。参考 https://leetcode-cn.com/problems/subtree-of-another-tree/solution/ling-yi-ge-shu-de-zi-shu-by-leetcode-solution/

## 解答

暴力匹配

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    public boolean isSubtree(TreeNode s, TreeNode t) {
        if (s == null && t == null) {
            return true;
        }
        if (s == null && t != null) {
            return false;
        }
        if (s != null && t == null) {
            return true;
        }
        if (isSameTree(s, t)) {
            return true;
        }
        if (isSubtree(s.left, t)) {
            return true;
        }
        if (isSubtree(s.right, t)) {
            return true;
        }
        return false;
    }

    private static boolean isSameTree(TreeNode a, TreeNode b) {
        if (a == null && b == null) {
            return true;
        }
        if (a != null && b == null) {
            return false;
        }
        if (a == null && b != null) {
            return false;
        }
        if (a.val != b.val) {
            return false;
        }
        return isSameTree(a.left, b.left) && isSameTree(a.right, b.right);
    }
}
```
