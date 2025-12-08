---
layout: post
title: 虚拟DOM的简单实现
date: 2019-09-09 18:29:10
tags: [React, 虚拟DOM]
categories:
 - web-framework
---
# 1. 什么是虚拟DOM
虚拟DOM简而言之就是，用JS去按照DOM结构来实现的树形结构对象，你也可以叫做DOM对象

<!-- more -->
# 2. 实现思路
 1. 用JS对象模拟DOM（虚拟DOM）
 2. 把此虚拟DOM转成真实DOM并插入页面中（render）
 3. 如果有事件发生修改了虚拟DOM，比较两棵虚拟DOM树的差异，得到差异对象（补丁数组）（diff）
 4. 把差异对象（补丁数组）应用到真正的DOM树上（patch）

# 3. 代码仓库
参考[React官网](https://zh-hans.reactjs.org/docs/introducing-jsx.html)，实现`createElement`等方法。为了省去热更新等配置，使用了`yarn create react-app my-app --typescript`构建项目。
详细直接看代码及注释
[github](https://github.com/seminelee/virtual-dom)
