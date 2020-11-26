# Flutter 网络请求

使用 http 库进行网络请求。

## 添加 http 依赖

在 pub.dartlang.org 网站上查找。

在 pubspec.yaml 文件添加 http：

```
dependencies:
  flutter:
    sdk: flutter

  http: ^0.12.0+2
```

使用命令下载依赖：

```
flutter packages get
```

## 使用 http 发起请求

为了不阻塞 UI，使用 async 异步函数：

```
void _getData() aysnc {
    ...
}
```

使用 await 等待 http.get 执行完后，再解析数据。

```
var response = await http.get(url);
if (response.statusCode == 200) {
    ...
}
```