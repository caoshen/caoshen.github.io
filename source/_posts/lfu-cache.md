---
title: LFU缓存
date: 2020-04-05 23:58:04
tags: 算法
---

# LFU缓存

请你为 最不经常使用（LFU）缓存算法设计并实现数据结构。它应该支持以下操作：get 和 put。

get(key) - 如果键存在于缓存中，则获取键的值（总是正数），否则返回 -1。
put(key, value) - 如果键不存在，请设置或插入值。当缓存达到其容量时，则应该在插入新项之前，使最不经常使用的项无效。在此问题中，当存在平局（即两个或更多个键具有相同使用频率）时，应该去除 最近 最少使用的键。
「项的使用次数」就是自插入该项以来对其调用 get 和 put 函数的次数之和。使用次数会在对应项被移除后置为 0 。

进阶：
你是否可以在 O(1) 时间复杂度内执行两项操作？

示例：

```
LFUCache cache = new LFUCache( 2 /* capacity (缓存容量) */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回 1
cache.put(3, 3);    // 去除 key 2
cache.get(2);       // 返回 -1 (未找到key 2)
cache.get(3);       // 返回 3
cache.put(4, 4);    // 去除 key 1
cache.get(1);       // 返回 -1 (未找到 key 1)
cache.get(3);       // 返回 3
cache.get(4);       // 返回 4
```

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/lfu-cache
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

除了使用 HashMap 保存每个数据之外，需要使用一个额外的 HashMap 保存每个频率和它对应的所有数据，LinkedHashSet 可以保证每个数据按序添加和返回，因此第二个 HashMap 的数据结构是 `HashMap<Integer, LinkedHashSet<Node>>`。

## 解答

```java
class LFUCache {
    private HashMap<Integer, Node> nodeMap;
    private HashMap<Integer, LinkedHashSet<Node>> freqMap;
    // 集合容量
    private int capacity = 0;
    // freqMap 里面的最小频率
    private int min = 0;

    private class Node {
        int key;
        int value;
        int freq;

        public Node(int key, int value, int freq) {
            this.key = key;
            this.value = value;
            this.freq = freq;
        }
    }

    public LFUCache(int capacity) {
        this.capacity = capacity;
        nodeMap = new HashMap<>();
        freqMap = new HashMap<>();
    }
    
    public int get(int key) {
        Node node = nodeMap.get(key);
        if (node == null) {
            return -1;
        } else {
            increaseFreq(node);
            return node.value;
        }
    }

    private void increaseFreq(Node node) {
        int freq = node.freq;
        // set 不为空，因为只有找到了 node 才会触发 increaseFreq 操作，increase 之前 node 已经被 add 过。
        LinkedHashSet<Node> set = freqMap.get(freq);
        if (set == null) {
            return;
        } else {
            // 从当前频率 set 删除 node
            set.remove(node);
            // set 删空了，而且 set 的 freq 是最小的频率，那么需要更新为当前频率
            if (set.isEmpty() && min == freq) {
                min = freq + 1;
            }
            // 更新 node 的频率
            int nextFreq = freq + 1;
            node.freq = nextFreq;
            LinkedHashSet<Node> nextSet = freqMap.get(nextFreq);
            if (nextSet == null) {
                // 下一个频率的 set 是空的
                nextSet = new LinkedHashSet<>();
                nextSet.add(node);
                freqMap.put(nextFreq, nextSet);
            } else {
                // 下一个频率的 set 非空
                nextSet.add(node);
            }
        }
    }
    
    public void put(int key, int value) {
        // capacity 为 0，直接返回，否则会加入新的 node
        if (capacity == 0) {
            return;
        }
        Node node = nodeMap.get(key);
        if (node == null) {
            // key 不存在 map 中
            // 首先检查是不是需要删除不符合条件的 node
            // 已满
            if (nodeMap.size() == capacity) {
                Node removed = removeNode();
                if (removed != null) {
                    // 注意 Map 需要根据 key 删除，Set 根据 value 删除
                    nodeMap.remove(removed.key);
                }
            }
            // 加入 node
            node = new Node(key, value, 1);
            nodeMap.put(key, node);
            addNode(node);
        } else {
            // key 已经存在 map 中，更新 value 和 freq
            node.value = value;
            increaseFreq(node);
        }
    }

    private void addNode(Node node) {
        int freq = node.freq;
        LinkedHashSet<Node> set = freqMap.get(freq);
        if (set == null) {
            set = new LinkedHashSet<>();
            set.add(node);
            freqMap.put(freq, set);
        } else {
            set.add(node);
        }
        // freqMap 新增加一个 node，导致现在最小的频率变为 1
        min = 1;
    }

    private Node removeNode() {
        LinkedHashSet<Node> minSet = freqMap.get(min);
        if (minSet == null) {
            return null;
        } else {
            // 删掉第一个，也就是最早 add 进来的
            Iterator<Node> it = minSet.iterator();
            if (it.hasNext()) {
                Node node = it.next();
                minSet.remove(node);
                return node;
            }
            return null;
        }
    }
}

/**
 * Your LFUCache object will be instantiated and called as such:
 * LFUCache obj = new LFUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```
