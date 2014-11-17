---
layout: default
title: VS编译时找不到系统库的问题（opengl32.lib）
---

## 前言

最近把家里的一个老项目放上github上，但在公司clone后却发现编译时找不到系统的一些库，如opengl32.lib等。

## 解决办法

通过和其他正常项目进行比较，发现他们项目属性里的VC++目录里的库目录不同。一个指向的是$(WindowsSdkDir_71A)lib，另一个指向的是$(WindowsSDK_LibraryPath_x86)lib。通过[同时使用VS 2012与VS 2010的问题解决方法](http://blog.csdn.net/dyllove98/article/details/9105455)知道是由于老项目是通过vs2010更新到vs2012的导致库路径不对。

这里不采用原文提到的添加搜索目录的办法，因为项目要放在github上，不想在其他环境也要配目录。直接把老项目改为使用VS2012创建的就可以了。

结束