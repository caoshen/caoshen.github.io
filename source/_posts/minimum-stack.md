---
title: 最小栈
date: 2019-12-09 23:26:03
tags: 栈
categories: 算法
---

# 最小栈

设计一个支持 push，pop，top 操作，并能在常数时间内检索到最小元素的栈。

- push(x) -- 将元素 x 推入栈中。
- pop() -- 删除栈顶的元素。
- top() -- 获取栈顶元素。
- getMin() -- 检索栈中的最小元素。

示例:

```
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.getMin();   --> 返回 -3.
minStack.pop();
minStack.top();      --> 返回 0.
minStack.getMin();   --> 返回 -2.
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/min-stack
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 解答

题目明确要求“在常数时间”解决，所以不能使用循环遍历的方法求最小值。

因此，需要额外的空间保存最小值。

这里的方法是使用一个额外的栈来保存每个元素所在位置对应的最小值。

```java
class MinStack {

    Stack<Integer> stack = new Stack<>();
    Stack<Integer> minStack = new Stack<>();

    public MinStack() {

    }

    public void push(int x) {
        stack.push(x);
        if (minStack.isEmpty()) {
            minStack.push(x);
        } else {
            Integer peek = minStack.peek();
            if (x < peek) {
                // 如果push的元素小于上一个位置的最小值，那么它自己就是最小值。
                // 极端情况是降序入栈，每个元素都是当前位置的最小值。
                minStack.push(x);
            } else {
                // 否则把上一个位置的最小值最为当前位置的最小值。
                // 极端情况是升序入栈，最小值是第一个元素。
                minStack.push(peek);
            }
        }
    }

    public void pop() {
        stack.pop();
        minStack.pop();
    }

    public int top() {
        return stack.peek();
    }

    public int getMin() {
        return minStack.peek();
    }
}

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack obj = new MinStack();
 * obj.push(x);
 * obj.pop();
 * int param_3 = obj.top();
 * int param_4 = obj.getMin();
 */
```
