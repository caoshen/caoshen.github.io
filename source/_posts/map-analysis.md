---
title: 地图分析
date: 2020-03-29 22:11:17
tags: 算法
categories: 算法
---

# 地图分析

你现在手里有一份大小为 N x N 的『地图』（网格） grid，上面的每个『区域』（单元格）都用 0 和 1 标记好了。其中 0 代表海洋，1 代表陆地，你知道距离陆地区域最远的海洋区域是是哪一个吗？请返回该海洋区域到离它最近的陆地区域的距离。

我们这里说的距离是『曼哈顿距离』（ Manhattan Distance）：(x0, y0) 和 (x1, y1) 这两个区域之间的距离是 |x0 - x1| + |y0 - y1| 。

如果我们的地图上只有陆地或者海洋，请返回 -1。

示例 1：

```
输入：[[1,0,1],[0,0,0],[1,0,1]]
输出：2
解释： 
海洋区域 (1, 1) 和所有陆地区域之间的距离都达到最大，最大距离为 2。
```

示例 2：

```
输入：[[1,0,0],[0,0,0],[0,0,0]]
输出：4
解释： 
海洋区域 (2, 2) 和所有陆地区域之间的距离都达到最大，最大距离为 4。
```

提示：

1 <= grid.length == grid[0].length <= 100
grid[i][j] 不是 0 就是 1

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/as-far-from-land-as-possible
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

题目要求距离陆地最远的海洋，那么可以用广度优先搜索（BFS），最后搜到的一定层次最深、距离最远。搜索之前将所有的陆地加入队列。

## 解答

```java
class Solution {
    public int maxDistance(int[][] grid) {
        Queue<int[]> queue = new ArrayDeque<>();
        int m = grid.length;
        int n = grid[0].length;
        // 所有的陆地进入队列
        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                if (grid[i][j] == 1) {
                    queue.offer(new int[]{i, j});
                }
            }
        }

        // 没有陆地
        if (queue.isEmpty()) {
            return -1;
        }
        // 最远海洋的坐标
        int[] point = null;
        // 四个方向的偏移
        int[] dx = {0, 1, 0, -1};
        int[] dy = {-1, 0, 1, 0};
        while(!queue.isEmpty()) {
            point = queue.poll();
            int x = point[0];
            int y = point[1];
            for (int i = 0; i < 4; ++i) {
                int tx = x + dx[i];
                int ty = y + dy[i];
                if (tx < 0 || tx >= m || ty < 0 || ty >= m) {
                    continue;
                }
                // 访问过的海洋或者陆地
                if (grid[tx][ty] > 0) {
                    continue;
                }
                // 曼哈顿距离加一
                grid[tx][ty] = grid[x][y] + 1;
                queue.offer(new int[]{tx, ty});
            }
        }
        // 没找到海洋
        if (point == null || grid[point[0]][point[1]] == 1) {
            return -1;
        } else {
            return grid[point[0]][point[1]] - 1;
        }
    }
}
```
