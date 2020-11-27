---
title: 岛屿数量
date: 2020-03-13 23:21:40
tags: 算法
categories: 算法
---

# 岛屿数量

给定一个由 '1'（陆地）和 '0'（水）组成的的二维网格，计算岛屿的数量。一个岛被水包围，并且它是通过水平方向或垂直方向上相邻的陆地连接而成的。你可以假设网格的四个边均被水包围。

示例 1:

```
输入:
11110
11010
11000
00000

输出: 1
```

示例 2:

```
输入:
11000
11000
00100
00011

输出: 3
```

## 思路

可以采用深度优先搜索（dfs）或者广度有限搜索（bfs）。开始搜索的次数就是岛屿的数量。

需要注意访问过的节点需要置为0，遇到边界值或者已经访问过的节点需要及时返回。

## 解答

深度优先搜索

```java
class Solution {
    public int numIslands(char[][] grid) {
        if (grid == null || grid.length == 0) {
            return 0;
        }
        int num = 0;
        int m = grid.length;
        int n = grid[0].length;
        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                if (grid[i][j] == '1') {
                    num++;
                    // start dfs
                    dfs(grid, i, j);
                }
            }
        }
        return num;
    }

    private void dfs(char[][] grid, int m, int n) {
        int rowCount = grid.length;
        int columnCount = grid[0].length;

        // check valid
        if (m < 0 || m >= rowCount || n < 0 || n >= columnCount) {
            return;
        }
        // check visited
        if (grid[m][n] == '0') {
            return;
        }
        // mark visited
        grid[m][n] = '0';
        // four direction dfs
        // up
        dfs(grid, m - 1, n);
        // right
        dfs(grid, m, n + 1);
        // bottom
        dfs(grid, m + 1, n);
        // left
        dfs(grid, m, n - 1);
    }

}
```