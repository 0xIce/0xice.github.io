---
title: RxSwift  Filtering Operators 实战
excerpt: 结合项目应用 Filtering Operators
categories:
  - RxSwift
tags:
  - RxSwift
toc: true
author_profile: false
sidebar:
  nav: RxSwift-docs
---

RxSwift 的操作符要么是 `Observable<E>` 中定义的方法，要么是 ObservableType 协议中定义的方法，`Observable<E>` 已经遵守了 ObservableType 协议

操作符对序列的处理会生成新的 observable 序列，因此我们可以**链式**调用操作符来执行多个操作

## 提炼照片序列



## 共享订阅

### 忽略所有元素

### 实现基本的独特的 filter

## 优化照片选择器

### PHPotoLibrary 授权 observable 序列

### 授权之后重新加载照片

### 用户未授权展示错误提示

## 尝试基于时间的 filter 操作符

### 在给定的时间间隔后结束订阅

### 在高负荷时使用 throttle 减少工作



