---
title: 单词接龙 II
date: 2020-06-15 00:26:57
tags: 算法
---

# 单词接龙 II

给定两个单词（beginWord 和 endWord）和一个字典 wordList，找出所有从 beginWord 到 endWord 的最短转换序列。转换需遵循如下规则：

每次转换只能改变一个字母。
转换后得到的单词必须是字典中的单词。
说明:

如果不存在这样的转换序列，返回一个空列表。
所有单词具有相同的长度。
所有单词只由小写字母组成。
字典中不存在重复的单词。
你可以假设 beginWord 和 endWord 是非空的，且二者不相同。
示例 1:

输入:
beginWord = "hit",
endWord = "cog",
wordList = ["hot","dot","dog","lot","log","cog"]

输出:
[
  ["hit","hot","dot","dog","cog"],
  ["hit","hot","lot","log","cog"]
]
示例 2:

输入:
beginWord = "hit"
endWord = "cog"
wordList = ["hot","dot","dog","lot","log"]

输出: []

解释: endWord "cog" 不在字典中，所以不存在符合要求的转换序列。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/word-ladder-ii
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

查找目标单词的过程实际上是一个图的广度优先搜索过程（BFS），如果一个单词可以由另外一个单词变换得到，那么把这个单词加入到备选路径中，然后将路径加入到 BFS 的队列。

可以将每个单词映射到一个 id，方便查找和构建图的边，将单词搜索转换为 id 搜索。

## 解答

```java
class Solution {
    private static final int INF = 1 << 20;
    private Map<String, Integer> wordId;
    private List<String> idWord;
    private List<Integer>[] edges;

    public Solution() {
        wordId = new HashMap<>();
        idWord = new ArrayList<>();
    }

    public List<List<String>> findLadders(String beginWord, String endWord, List<String> wordList) {
        // 初始化 word id map 和 id word list
        int id = 0;
        for (String word : wordList) {
            if (!wordId.containsKey(word)) {
                wordId.put(word, id++);
                idWord.add(word);
            }
        }
        // 提前判断 endWord 在不在列表
        if (!wordId.containsKey(endWord)) {
            return new ArrayList<>();
        }
        // 加入 beginWord 的 id，方便查找
        if (!wordId.containsKey(beginWord)) {
            wordId.put(beginWord, id++);
            idWord.add(beginWord);
        }
        // 初始化图的边
        edges = new ArrayList[idWord.size()];
        for (int i = 0; i < edges.length; ++i) {
            edges[i] = new ArrayList<>();
        }
        for (int i = 0; i < idWord.size(); ++i) {
            for (int j = i + 1; j < idWord.size(); ++j) {
                if (transformCheck(idWord.get(i), idWord.get(j))) {
                    edges[i].add(j);
                    edges[j].add(i);
                }
            }
        }

        // 初始化 beginWord 到每个 word 的花费 cost
        int[] cost = new int[id];
        for (int i = 0; i < id; ++i) {
            cost[i] = INF;
        }
        // 起点 id
        int source = wordId.get(beginWord);
        // beginWord 的 cost 为 0
        cost[source] = 0;
        // 目的 id
        int dest = wordId.get(endWord);
        // 输出的答案
        List<List<String>> res = new ArrayList<>();
        // 将起点加入队列
        Queue<ArrayList<Integer>> q = new LinkedList<>();
        ArrayList<Integer> tmpBegin = new ArrayList<>();
        tmpBegin.add(source);
        q.add(tmpBegin);
        // 开始 BFS
        while(!q.isEmpty()) {
            ArrayList<Integer> now = q.poll();
            // 最近访问的点，也就是当前搜索路径上最靠前的点
            int last = now.get(now.size() - 1);
            if (last == dest) {
                // 找到了终点
                ArrayList<String> tmp = new ArrayList<>();
                for (int i = 0; i < now.size(); ++i) {
                    tmp.add(idWord.get(now.get(i)));
                }
                // 把找到的路径加入到答案，然后继续查找剩余的可能路径。
                res.add(tmp);
            } else {
                // 继续查找最近访问节点的每一条边，找到可以变换单词的路径
                for (int i = 0; i < edges[last].size(); ++i) {
                    int to = edges[last].get(i);
                    // <= 的作用在于把代价相同的其他路径也保存下来
                    if (cost[last] + 1 <= cost[to]) {
                        cost[to] = cost[last] + 1;
                        // 这里构造了一个新的节点，进入队列
                        ArrayList<Integer> tmp = new ArrayList<>(now);
                        // 更新最近访问的点
                        tmp.add(to);
                        // 把符合条件的路径加入队列
                        q.add(tmp);
                    }
                }
            }
        }
        return res;
    }

    /**
     * 是否能够变换一个字母成为相同单词
     */
    private boolean transformCheck(String str1, String str2) {
        int diff = 0;
        for (int i = 0; i < str1.length() && diff < 2; ++i) {
            if (str1.charAt(i) != str2.charAt(i)) {
                ++diff;
            }
        }
        return diff == 1;
    }
}
```
