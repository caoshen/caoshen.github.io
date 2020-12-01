---
title: CNAME 配置
date: 2020-05-01 13:22:07
tags: 网络
categories: 网络
---

# CNAME 配置

## CNAME作用

CNAME 即指别名记录，也被称为规范名字。这种记录允许将多个名字映射到同一台计算机。 当需要将域名指向另一个域名，再由另一个域名提供 ip地址，就需要添加 CNAME 记录。

CNAME会解析到另一个域名，之后再对另一个域名继续解析，直到解析出节点。

## 域名网站添加域名解析

以腾讯云为例，找到“云解析”，在“域名解析列表”添加CNAME记录。

| 主机记录 | 记录类型 | 线路类型 | 记录值 | TTL（秒）|
| ------ | ------ | ------ | ------ | ------ |
| xxx | CNAME | 默认 | xxx.github.io. | 600 |

等待解析生效后，xxx.yourdomain.com 就能解析到 xxx.github.io

注意记录值最后需要加上一个点

## 在 github pages 新建 CNAME 文件

新增 CNAME 文件

```
xxx.yourdomain.com
```

## Hexo 博客添加 CNAME

Hexo 博客添加 CNAME 时，需要将 CNAME 文件放入 source 目录中。hexo g -d 部署后，CNAME 文件会被自动放入网站的根目录下。



