---
title: 点菜展示表
date: 2020-04-19 12:55:07
tags: 算法
---

# 点菜展示表

给你一个数组 orders，表示客户在餐厅中完成的订单，确切地说， orders[i]=[customerNamei,tableNumberi,foodItemi] ，其中 customerNamei 是客户的姓名，tableNumberi 是客户所在餐桌的桌号，而 foodItemi 是客户点的餐品名称。

请你返回该餐厅的 点菜展示表 。在这张表中，表中第一行为标题，其第一列为餐桌桌号 “Table” ，后面每一列都是按字母顺序排列的餐品名称。接下来每一行中的项则表示每张餐桌订购的相应餐品数量，第一列应当填对应的桌号，后面依次填写下单的餐品数量。

注意：客户姓名不是点菜展示表的一部分。此外，表中的数据行应该按餐桌桌号升序排列。

示例 1：

```
输入：orders = [["David","3","Ceviche"],["Corina","10","Beef Burrito"],["David","3","Fried Chicken"],["Carla","5","Water"],["Carla","5","Ceviche"],["Rous","3","Ceviche"]]
输出：[["Table","Beef Burrito","Ceviche","Fried Chicken","Water"],["3","0","2","1","0"],["5","0","1","0","1"],["10","1","0","0","0"]] 
解释：
点菜展示表如下所示：
Table,Beef Burrito,Ceviche,Fried Chicken,Water
3    ,0           ,2      ,1            ,0
5    ,0           ,1      ,0            ,1
10   ,1           ,0      ,0            ,0
对于餐桌 3：David 点了 "Ceviche" 和 "Fried Chicken"，而 Rous 点了 "Ceviche"
而餐桌 5：Carla 点了 "Water" 和 "Ceviche"
餐桌 10：Corina 点了 "Beef Burrito" 
```

示例 2：

```
输入：orders = [["James","12","Fried Chicken"],["Ratesh","12","Fried Chicken"],["Amadeus","12","Fried Chicken"],["Adam","1","Canadian Waffles"],["Brianna","1","Canadian Waffles"]]
输出：[["Table","Canadian Waffles","Fried Chicken"],["1","2","0"],["12","0","3"]] 
解释：
对于餐桌 1：Adam 和 Brianna 都点了 "Canadian Waffles"
而餐桌 12：James, Ratesh 和 Amadeus 都点了 "Fried Chicken"
```

示例 3：

```
输入：orders = [["Laura","2","Bean Burrito"],["Jhon","2","Beef Burrito"],["Melissa","2","Soda"]]
输出：[["Table","Bean Burrito","Beef Burrito","Soda"],["2","1","1","1"]]
```

提示：

- 1 <= orders.length <= 5 * 10^4
- orders[i].length == 3
- 1 <= customerNamei.length, foodItemi.length <= 20
- customerNamei 和 foodItemi 由大小写英文字母及空格字符 ' ' 组成。
- tableNumberi 是 1 到 500 范围内的整数。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/display-table-of-food-orders-in-a-restaurant
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

按照题意，构造一个“桌号-->桌号对应的菜单”的 Map，打印菜单时按序打印。

## 解答

```java
class Solution {
    // 保存所有的菜，[3, [['Beef Burrito', 0], ['Ceviche', 2], ...]]
    private HashMap<Integer, HashMap<String, Integer>> orderMap = new HashMap<>();
    // 保存所有的菜名
    private HashSet<String> foodSet = new HashSet<>();
    // 保存所有的桌子编号
    private HashSet<Integer> tableSet = new HashSet<>();

    public List<List<String>> displayTable(List<List<String>> orders) {
        for (List<String> order: orders) {
            if (order.size() != 3) {
                continue;
            } else {
                String customerName = order.get(0);
                int tableNumber = Integer.valueOf(order.get(1));
                String foodItem = order.get(2);
                tableSet.add(tableNumber);
                foodSet.add(foodItem);
                HashMap<String, Integer> tableOrderMap = orderMap.get(tableNumber);
                if (tableOrderMap == null) {
                    tableOrderMap = new HashMap<>();
                    orderMap.put(tableNumber, tableOrderMap);
                }
                Integer foodCount = tableOrderMap.get(foodItem);
                if (foodCount == null) {
                    foodCount = 0;
                    tableOrderMap.put(foodItem, foodCount);
                }
                tableOrderMap.put(foodItem, foodCount + 1);
            }
        }
        
        List<List<String>> resultList = new ArrayList<>();
        ArrayList<String> header = new ArrayList<String>(foodSet);
        Collections.sort(header);
        header.add(0, "Table");
        resultList.add(header);
        
        List<Integer> tableList = new ArrayList<Integer>(tableSet);
        Collections.sort(tableList);
        
        for (int i = 0; i < tableList.size(); ++i) {
            ArrayList<String> foodOfTable = new ArrayList<String>();
            int table = tableList.get(i);
            foodOfTable.add(String.valueOf(table));
            HashMap<String, Integer> foods = orderMap.get(table);
            for (int j = 1; j < header.size(); ++j) {
                Integer count = foods.get(header.get(j));
                if (count == null) {
                    count = 0;
                }
                foodOfTable.add(String.valueOf(count));
            }
            resultList.add(foodOfTable);
        }
        
        return resultList;
    }
}
```
