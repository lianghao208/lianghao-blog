---
title: "混沌工程-Chaos Mesh初探"
date: 2020-12-24T14:24:26+08:00
draft: true
tags:
    - Kubernetes
    - ChaosMesh
---

## 背景
应用上云后带来了各种各样的挑战，我们无法避免生产环境发生故障。无论故障是由于物理机房大面积宕机导致的，还是由于机器资源紧张导致的，都会对业务的正常运行带来严重的影响。我们不妨采用一些手段提前模拟各种可能出现的故障情况，而混沌工程（Chaos Engineering）的出现很好解决了这样一个难题。
（提前暴露问题）
## 基本原则
参考混沌工程的基本原则：[PRINCIPLES OF CHAOS ENGINEERINGt](https://principlesofchaos.org/?lang=ENcontent)
我们可以将实现混沌工程的基本原则分为以下5项：
https://www.jianshu.com/p/11798468da77
https://zhuanlan.zhihu.com/p/100738380
### 一、定义系统稳定状态
### 二、尽可能多地模拟现实中潜在发生的各种故障
### 三、在生产环境模拟实验
### 四、自动化多次执行实验
### 五、最大程度减少实验产生的影响
在实验过程中我们应该尽可能地做到业务方无感知，减少实验带来的负面影响。