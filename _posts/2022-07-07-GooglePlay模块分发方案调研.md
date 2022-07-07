---
layout: post
title: GoolgePlay模块分发方案调研
date: 2022-07-07 18:00:00
categories: 
- Android
tags:
- GooglePlay
---  

## Play Feature Delivery 概览
上架到Google Play的aab包，存在150M大小的限制，当超过的时候，Google Play不能上传提审，必须借助Play Feature Delivery功能实现用户按需下载。

功能：**能够自定义如何以及何时将应用的不同功能下载到搭载Android 5.0或更高版本的设备上**

| 分发选项 | 行为 | 样例 |
| :-----| ----: | :----: |
| 安装时分发 | 默认情况下，未配置上述任何分发选项的功能模块会在安装应用时下载。这是一个重要的行为，因为这意味着您可以逐步采用高级分发选项 | 如果应用包含特定的指导Activity(比如关于如何在购物平台上买卖商品的交互式指南)，可以配置为在应用安装时默认包含该功能。但是为了减少应用的安装大小，应用可在用户完成指导之后请求删除功能 |
| 按需分发 | 允许您的应用根据需要请求和下载功能模块 | 如果使用购物平台的用户中，只有20%的人发布待售商品，有一个不错的策略可以减少大多数用户的初始化下载大小，将拍照，输入商品以及上架商品功能配置按需下载 |  
| 根据条件分发 | 允许您指定特定的用户设备需求(例如硬件特性、区域设置和最低API级别)，以确定是否在安装应用时候下载模块化功能  | 支付注册区域  
| 免安装分发 | Google Play的免安装体验让用户无需再设备上安装APK即可与应用互动。用户通过Google Play商店中的‘立即体验’按钮或者您创建网址体验应用  | 游戏的前几个关卡，放在轻量级功能模块中


## 注意事项
* 按条件分发或者按需分发安装50个以上导致性能问题
* 只有Android5.0以上，才支持按需下载和安装功能
* Play Feature Delivery要求使用app bundle发布应用
* 家长端和学生端可以拆分出来的功能模块


## 参考文档:

[Configure on demand delivery  |  Android Developers](https://developer.android.com/guide/app-bundle/on-demand-delivery) 

[Overview of Play Feature Delivery  |  Android Developers](https://developer.android.com/guide/app-bundle/play-feature-delivery) 