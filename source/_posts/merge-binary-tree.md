---
title: 合并二叉树
date: 2020-04-13 22:42:20
tags: 
  - 算法
  - 树
categories: 算法
---

# 合并二叉树

给定两个二叉树，想象当你将它们中的一个覆盖到另一个上时，两个二叉树的一些节点便会重叠。

你需要将他们合并为一个新的二叉树。合并的规则是如果两个节点重叠，那么将他们的值相加作为节点合并后的新值，否则不为 NULL 的节点将直接作为新二叉树的节点。

示例 1:

```
输入: 
	Tree 1                     Tree 2                  
          1                         2                             
         / \                       / \                            
        3   2                     1   3                        
       /                           \   \                      
      5                             4   7                  
输出: 
合并后的树:
	     3
	    / \
	   4   5
	  / \   \ 
	 5   4   7
```

注意: 合并必须从两个树的根节点开始。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/merge-two-binary-trees
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

用递归的思路，每次加两个节点。

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
    public TreeNode mergeTrees(TreeNode t1, TreeNode t2) {
        if (t1 == null) {
            return t2;
        }
        if (t2 == null) {
            return t1;
        }
        if (t1 != null && t2 != null) {
            int val = t1.val + t2.val;
            TreeNode root = new TreeNode(val);
            TreeNode left = mergeTrees(t1.left, t2.left);
            TreeNode right = mergeTrees(t1.right, t2.right);
            root.left = left;
            root.right = right;
            return root;
        }
        return null;
    }
}
```
