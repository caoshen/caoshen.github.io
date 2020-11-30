---
title: 交点
date: 2020-04-12 18:34:08
tags: 算法
categories: 算法
---

# 交点

给定两条线段（表示为起点start = {X1, Y1}和终点end = {X2, Y2}），如果它们有交点，请计算其交点，没有交点则返回空值。

要求浮点型误差不超过10^-6。若有多个交点（线段重叠）则返回 X 值最小的点，X 坐标相同则返回 Y 值最小的点。

示例 1：

```
输入：
line1 = {0, 0}, {1, 0}
line2 = {1, 1}, {0, -1}
输出： {0.5, 0}
```

示例 2：

```
输入：
line1 = {0, 0}, {3, 3}
line2 = {1, 1}, {2, 2}
输出： {1, 1}
```

示例 3：

```
输入：
line1 = {0, 0}, {1, 1}
line2 = {1, 0}, {2, 1}
输出： {}，两条线段没有交点
```

提示：

坐标绝对值不会超过 2^7
输入的坐标均是有效的二维坐标

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/intersection-lcci
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

如果其中一个线段的两个点都在另一个线段的同一边，那么它们不会相交。如果每一个线段的两个点都在另一个线段的不同边或者有一个点在另一个线段上，那么会相交。

针对相交的情况，又可以分为两种情况：

1. 两条线段在一条直线的情况，需要根据线段的端点判断两条线段是否重叠。
2. 两条线段不在一条直线的情况，根据两条直线的方程求交点坐标。

## 解答

```java
class Solution {
    /**
     * 判断两个线段是否相交。
     * 如果其中一个线段的两个点都在另一个线段的同一边，那么不会相交。
     * 如果每一个线段的两个点都在另一个线段的不同边，那么会相交。
     */
    public double[] intersection(int[] start1, int[] end1, int[] start2, int[] end2) {
        int resultStart2OfLine1 = calcSide(start1, end1, start2);
        int resultEnd2OfLine1 = calcSide(start1, end1, end2);
        boolean isLine2CrossLine1 = resultStart2OfLine1 * resultEnd2OfLine1 <= 0;
        int resultStart1OfLine2 = calcSide(start2, end2, start1);
        int resultEnd1OfLine2 = calcSide(start2, end2, end1);
        boolean isLine1CrossLine2 = resultStart1OfLine2 * resultEnd1OfLine2 <= 0;

        // 线段会相交
        if (isLine2CrossLine1 && isLine1CrossLine2) {
            // 重叠情况，同一条直线。其中一个端点一定在另一条线段上。
            if (resultStart2OfLine1 == 0 && resultEnd2OfLine1 == 0) {

                // 将线段端点按序排列
                int[] min1 = calcMinPointer(start1, end1);
                int[] max1 = end1;
                if (min1 != start1) {
                    max1 = start1;
                }
                int[] min2 = calcMinPointer(start2, end2);
                int[] max2 = end2;
                if (min2 != start2) {
                    max2 = start2;
                }

                if (min1[0] > min2[0] || min1[1] > min2[1]) {
                    // 交换线段位置
                    int[] temp = min1;
                    min1 = min2;
                    min2 = temp;

                    temp = max1;
                    max1 = max2;
                    max2 = temp;
                }

                if (comparePoints(max1, min2) < 0) {
                    return new double[]{};
                } else if (comparePoints(max2, min1) < 0) {
                    return new double[]{};
                } else if (comparePoints(max1, min2) >= 0) {
                    return new double[]{min2[0], min2[1]};
                } else if (comparePoints(max2, min1) >= 0) {
                    return new double[]{min1[0], min1[1]};
                } else {
                    return new double[]{};
                }
            }
 
            // 两条直线有交点情况
            // 求解 (y1 - y2) * (x - x1) - (x1 - x2) * (y - y1) = 0 和 (y3 - y4) * (x - x3) - (x3 - x4) * (y - y3) = 0
            //  先求 x 。(y1 - y2) * (x - x1) / (x1 - x2) + y1 = y = (y3 - y4) * (x - x3) / (x3 - x4) + y3 = y
            // (y1-y2)(x-x1)(x3-x4) - (y3-y4)(x-x3)(x1-x2) = (y3 - y1)(x1-x2)(x3-x4);
            //  [(y1-y2)(x3-x4) - (y3-y4)(x1-x2)]x + [(y3-y4)(x1-x2)x3 - (y1-y2)(x3-x4)x1] = (y3 - y1)(x1-x2)(x3-x4)
            // x = {(y3 - y1)(x1-x2)(x3-x4) - [(y3-y4)(x1-x2)x3 - (y1-y2)(x3-x4)x1]} / [(y1-y2)(x3-x4) - (y3-y4)(x1-x2)]
            double p0 = ((start2[1] - start1[1]) * (start1[0] - end1[0]) * (start2[0] - end2[0])
                - ((start2[1] - end2[1]) * (start1[0] - end1[0]) * start2[0] - (start1[1] - end1[1]) * (start2[0] - end2[0]) * start1[0])
                ) * 1.0 / ((start1[1] - end1[1]) * (start2[0] - end2[0]) - (start2[1] - end2[1]) * (start1[0] - end1[0]));
            // 同理 y = {(x3 - x1)(y1-y2)(y3-y4) - [(x3-x4)(y1-y2)y3 - (x1-x2)(y3-y4)y1]} / [(x1-x2)(y3-y4) - (x3-x4)(y1-y2)]
            double p1 = ((start2[0] - start1[0]) * (start1[1] - end1[1]) * (start2[1] - end2[1])
                - ((start2[0] - end2[0]) * (start1[1] - end1[1]) * start2[1] - (start1[0] - end1[0]) * (start2[1] - end2[1]) * start1[1])
                ) * 1.0 / ((start1[0] - end1[0]) * (start2[1] - end2[1]) - (start2[0] - end2[0]) * (start1[1] - end1[1]));
            return new double[]{p0, p1};
        } else {
            return new double[]{};
        }
    }

    // 线段的方程： (y1 - y2) * (x - x1) - (x1 - x2) * (y - y1) = 0
    private int calcSide(int[] start, int[] end, int[] point) {
        return (start[1] - end[1]) * (point[0] - start[0]) - (start[0] - end[0]) * (point[1] - start[1]);
    }

    private int[] calcMinPointer(int[] point1, int[] point2) {
        if (point1[0] < point2[0]) {
            return point1;
        } else if (point1[0] == point2[0]) {
            if (point1[1] < point2[1]) {
                return point1;
            } else {
                return point2;
            }
        } else {
            return point2;
        }
    }

    private int comparePoints(int[] point1, int[] point2) {
        if (point1[0] < point2[0]) {
            return -1;
        } else if (point1[0] == point2[0]) {
            if (point1[1] < point2[1]) {
                return -1;
            } else if (point1[1] == point2[1]) {
                return 0;
            } else {
                return 1;
            }
        } else {
            return 1;
        }
    }
}
```
