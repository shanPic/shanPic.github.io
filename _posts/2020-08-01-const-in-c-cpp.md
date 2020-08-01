---
layout: post
title: "C/C++中关于const限定符的一些行为"
date: 2020-08-01 00:00:00
description: C/C++中关于const限定符的一些行为
comments: false
tags: 
 - C++
---

## 全局const变量

编译成目标文件时，放在了目标文件的.rodata段（只读数据段）中（PS：有时候编译器会把字符串常量放在.data段中）。以下为无全局const变量与有一个全局const int变量的段结构对比：
有全局const变量
![](/resource/images/2020-08-01-const-in-c-cpp/Image-1.png){:.center-image }
无全局const变量
![](/resource/images/2020-08-01-const-in-c-cpp/Image-2.png){:.center-image }

猜测：elf文件标准在设计的时候，加了一个单独的.rodata段，可能就是为了支持C/C++的const语义

TODO：为何无全局const时，.rodata仍占有4byte 的空间？

导致的结果：段错误导致core，原因是写请求只读内存。

## 栈上的const变量

```
const的语义 in C：const修饰的变量将在编译期检查是否被直接更改了，若被直接更改了，则error。
const的语义 in C++：const修饰的变量将在编译期检查是否被直接更改了，若被直接更改了，则error。
```

既然const的语义在C与C++中完全相同，那为什么会出现表现不一致的情况呢？（当使用指针间接修改const变量的值时，C与C++会出现表现不一致的情况。注意：其实通过指针间接修改栈上const变量的值是个未定义行为，大家最好不要写这种代码。。。）。c++编译器（我做实验使用的是gcc与clang）使用了常量折叠的优化。

参考：
1. 大体概念： https://www.cnblogs.com/danshui/archive/2012/04/06/2434550.html 
2. VC++如何做常量折叠的：https://devblogs.microsoft.com/cppblog/optimizing-c-code-constant-folding/ 

在gcc的文档（ https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html）中发现了两个与常量折叠（constant folding）有关的选项：。但经过尝试，都无法关闭C++中的常量折叠。

so上有位老哥提供了一个思路（方法）disable掉常量折叠： https://stackoverflow.com/questions/22288884/g-turn-off-constant-propagation-for-benchmarking\

结论：关于const的语义，C与C++没有特殊的区别。但c++编译器会进行常量折叠的优化。（TODO：确认c++编译器对于全局const变量是否会做常量折叠？如果做，除非是比较长的数组之类的，某些POD类型岂不是可以不用放在.rodata段中，直接搞到.text段里岂不爽歪歪）

refs: 之前有人问过这个UB的问题： https://www.zhihu.com/question/302390890

PS：大家闲着没事研究下可以，但千万不要写这个UB的代码。。。 

