---
date: 2023-11-14 08:00:00
layout: post
title: Android 系统稳定性优化实践 - 锁是一把双刃剑
subtitle: Android 系统稳定性优化实践 - 锁是一把双刃剑
description: Android 系统稳定性优化实践 - 锁是一把双刃剑
image: /assets/img/markdown_img/image-title-android-system-optimize-lock.jpg
optimized_image: /assets/img/markdown_img/image-title-android-system-optimize-lock.jpg
category: Android system stability optimization
tags:
    - Android System
    - Android system stability optimization
author: etoilexx
paginate: false
---


## Sec1. Problem Description

突然接到测试同学报告的一个问题，设备在打开自研应用护眼助手后整机卡死重启，并且无法进入系统，卡开机动画。

## Sec2. Problem Analysis Process
由于是必现问题，所以直接先打开logcat看当前设备卡在什么地方：
![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-stability-optimization/lock-is-a-double-edged-sword/1.png)

通过logcat的信息发现，整机已经触发了SWT，`android.hardware.sensor@2.0-service-mediatek`一直在报错，错误是`“Timeout 头 write and wait sensorservice ack”`，也就是说`sensoreservice`卡住没有响应了，这也难怪系统会卡住，如果是sensorservice的问题的话，上层开启了自研护眼助手应用后会注册SensorEventListener用于接收底层光感（ALS）的数据进行展示，如果这个时候sensorservice不通，那么肯定应用ANR。

于是我们通过`dumpsys sensorservice`查看当前sensorservice的状态，并通过`dmesg -Tw | grep -iE` "sensor"查看kernel的log：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-stability-optimization/lock-is-a-double-edged-sword/2.png)

发现dumpsys sensorservice同样超时了，说明问题还出在更底层，根据kernel的log，发现sensor的事件一直报不上去，提示input buf已满（这就很奇怪，也没几个传感器怎么会满呢？），由于系统已经触发了SWT，通过MTK-AEE的机制可以查看触发SWT时的backtrace：
![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-stability-optimization/lock-is-a-double-edged-sword//3.png)

但是！！！`ALS_ENABLE_AFTER_RESUME`这个宏并未定义，这就导致了mutex_lock后一直没有unlock，后续的光感数据均无法继续上报，继而蝴蝶效应引发上层与整机的一系列崩溃问题。

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-stability-optimization/lock-is-a-double-edged-sword//4.png)

## Sec3. Solve the problem
定位到问题后就很简单了，只需要把mutex_lock包在宏定义里面就行了:

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-stability-optimization/lock-is-a-double-edged-sword//5.png)
