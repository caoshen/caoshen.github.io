---
title: 阿里云镜像配置
date: 2019-08-18 22:15:58
tags: 镜像
categories:
---

# 阿里云镜像配置

由于一些网络原因，下载软件和依赖的速度很慢。切换使用淘宝镜像下载。

## 阿里云 Maven 镜像

gradle

```
repositories {
    maven {url 'http://maven.aliyun.com/nexus/content/groups/public/'}
    mavenCentral()
}
```

maven

```
<repositories>
    <repository>
        <id>aliyunmaven</id>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    </repository>
</repositories>
```

## 淘宝 NPM 镜像

https://npm.taobao.org/

可以使用 cnpm (gzip 压缩支持) 命令行工具代替默认的 npm:

```
$ npm install -g cnpm --registry=https://registry.npm.taobao.org
```

