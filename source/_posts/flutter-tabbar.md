---
title: Flutter TabBar 标签
date: 2019-08-20 00:38:25
tags: Flutter
categories: Flutter
---

# Flutter TabBar 标签

Flutter 实现标签左右滑动切换，可以使用 TabBar 和 TabBarView。TabBar 和 TabBarView 分别表示标签和标签对应的内容页面。

## TabBar

TabBar 需要指定一个 TabController 才能使用，TabController 用来控制 TabBar 的切换。

unselectedLabelColor： 未选中的标签的颜色。

labelColor：选中的标签的颜色。

indicatorColor：标签下方指示器线条的颜色。

indicatorSize：标签下方指示器线条的长度。如果是 label，表示和标签内容长度一样，如果是 tab，表示和整个标签长度一样长，即上一个标签的右边到下一个标签左边的长度。

tabs：表示每个标签的 widget。

```dart
              Container(
                constraints: BoxConstraints.expand(height: 50),
                child: TabBar(
                    controller: _controller,
                    unselectedLabelColor: Colors.black,
                    labelColor: Colors.blue,
                    indicatorColor: Colors.blue,
                    indicatorSize: TabBarIndicatorSize.label,
                    tabs: [
                      Tab(text: "未结束"),
                      Tab(text: "已结束"),
                      Tab(text: "我的比赛")
                    ]),
              ),
```

## TabBarView

TabBarView 指的是每个标签对应的内容界面，这个设置成 Expanded 可以扩展到整个内容区域。

TabBarView 也需要指定 TabController 和 每个标签下的 widget。

```dart
              Expanded(
                child: TabBarView(controller: _controller, children: [
                  ScoreListWidget(_isLoaded, _unFinishedMatches),
                  ScoreListWidget(_isLoaded, _finishedMatches),
                  Center(child: Text("我的比赛"))
                ]),
              )
```

## TabController

TabController 用来控制内容切换，可以将 TabBar 和 TabBarView 嵌套在一个 DefaultTabController 中。也可以使用自定义的 TabController。

```dart
    _controller = TabController(length: 3, vsync: ScrollableState());
    _controller.addListener(() => {
          setState(() {
            if (_controller.index == 0) {
              // ...
            }
          })
        });
```

TabController 通过 length 指定标签的个数。addListener 可以添加标签的滑动监听。