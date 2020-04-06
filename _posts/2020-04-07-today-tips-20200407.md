---
layout: post
title: "今日碎片"
date: 2020-04-07 00:00:00
description: 2020-04-07
comments: false
tags: 
 - 每日碎片
 - C++
 - CMake
---

# 1. CMake在某一个target中清空include_directory
我们都知道，现代CMake是面向target的。在设置target的头文件搜索路径时，也推荐使用`target_include_directory`单独为某个target设置头文件搜索路径，并同时设置路径对其他目标的可见性（private, interface...）。

但如果你在参与一个有一定历史的大型项目，其CMake可能经历了好“几代”程序员的维护，就有可能出现`target_include_directory`与`include_directory`混用的情况。如果这是又恰巧两个目标里面，有同名的头文件，就会出现诡异的事情了 :(

有没有办法解决这种情况呢？

有，使用CMake的`target_set_property`。

# 2. C++初始化列表与窄化类型转换
