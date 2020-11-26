---
title: Bluedroid 的代码结构分析
date: 2020-03-31 21:32:44
tags: 蓝牙
categories: 蓝牙
---

# Bluedroid 的代码结构分析

system/bt 的主要文件结构及相应功能介绍如下。

## main

bte_main.cc

该功能涉及BTE核心栈的初始化和卸载。

- bte_main_in_hw_init：负责芯片硬件的初始化
- bte_main_boot_entry：调用 GKI_init，

bte_init.cc

BTE_InitStack：初始化 BTE 控制块，如 RFCOMM、DUN、SPP、HSP2 和 HFP 等。核心 stack 必须在创建 BTU task（任务）前调用。

## bta

bta 用于和 Bluetooth process 层交互，实现蓝牙设备管理、状态管理以及一些 Profile 的 Bluedroid 实现。BTA 的主要组件如下所示。

- AG 实现 BTA 音频网关（audio gateway）
- AR 负责 Audio/Video 注册
- AV 实现 BTA advanced audio/video
- DM 实现 BTA 设备管理
- GATT 实现通用属性配置文件（Generic Attribute Profile），此模块是 Bluetooth 4.0 新增加的核心协议。
- HL 实现 HDP （Health Device Profile）协议，此协议主要用于与健康设备的蓝牙连接，比如心率监护仪、血压测量仪、体温计等。
- PAN 实现 PAN （蓝牙个人局域网）协议，使得设备可以连接以下设备：个人局域网用户（PANU）设备、组式临时网络（GN）设备或网络访问点（NAP）设备。
- HH 实现 HID （Human Interface Device）协议，典型的应用包括蓝牙遥控器、蓝牙鼠标、蓝牙键盘、蓝牙游戏手柄等。
- PBAP 实现 PBAP （Phone Book Access Profile）协议，用于从电话薄交换服务器上获取电话薄内容。
- SYS 主要实现 BTA 系统管理。

## btif

Bluetooth Interface：提供所有 Bluetooth Process 需要的 API。

1. src/bluetooth.cc HAL 层定义数组和函数体的实现。
2. src/btif_av.cc Bluedroid 上 AV 的实现，主要结构和功能函数如下。
3. src/btif_core.cc 该功能包含 HAL 层和 BTE 核心协议栈的核心接口函数。
4. src/btif_dm.cc 该功能实现设备管理（Device Manage）相关的功能。
5. src/btif_gatt.cc 实现 gatt 相关的接口。
6. src/btif_hf.cc 该功能实现 handsfree 协议的接口。
7. src/btif_hh.cc 该功能实现 HID Host 的蓝牙接口。
8. src/btif_hl.cc 该功能实现健康设备（Health Device）的蓝牙接口。
9. src/btif_media_task.cc btif 中的多媒体模块处理，AV（Audio Video）、HS（Headset）、HF（Handsfree）中的 audio 和 video 任务的处理。
10. src/btif_pan.cc 该功能实现 PAN 的蓝牙接口。
11. src/btif_rc.cc AVRCP 的实现，完成蓝牙耳机对音乐播放的控制。
12. src/btif_rc.cc 关于 btif 中状态机的处理。
13. src/btif_sock.cc Socket 相关接口。通过 btsock_listen 和 btsock_connect 来处理 SCO、L2CAP 和 RFCOMM 的监听与连接的建立。

## HCI

HCI library 的实现，主要内容包括 HCI 接口的打开和收/发控制、Vendor 的 so 的打开和回调函数的注册、LPM（Low Power Mode） 的实现、btsnoop 的抓取等。

1. src/bt_hci_bdroid.c 该功能主要处理 Bluedroid 中 Host/Controller 接口（HCI）的实现。
2. src/vendor.c 该功能定义了 vendor 的调用函数，加载 libbt-vendor.so 库（由 vendor 提供的 libbt 文件夹里面的代码生成），初始化 vendor_interface，注册 vendor 需要的回调函数。
3. src/hci_h4.c 该功能包含 HCI 传送/接收处理。
4. src/hci_mct.c 该功能处理多链路的 HCI 发送和接收。
5. src/lpm.c 低功耗模式（Low Power Mode，LPM）用于完成低功耗模式相关的处理。

不同的 Android 版本 hci 实现可能不同，可以在 system/bt/hci/src/ 下查看相关文件。

## stack

stack 主要用于完成各协议在 Bluedroid 中的实现，协议包含 a2dp、avctp、avdtp、avrcp、bnep、gap、gatt、hid、l2cap、pan、rfcomm、sdp、macp（Multi-Channel Adaptation Protocol，多通道适配协议）、smp（用于生成对等协议的加密密钥和身份密钥），还包含几个其他模块。

1. btm 主要涉及 Bluetooth Manger。
2. btu 该功能主要用于核心协议层之间的事件处理与转换。

## 参考资料

《低功耗蓝牙智能硬件开发实战》第2.7节《Bluedroid 的代码结构分析》
