---
title: HTTP-基本概念和术语
tags:
  - http
toc: true
---

## 1. MIME

- 表示媒体类型

- 客户端可以针对不同类型做不同处理。

- Content-type: image/jpeg

- 最初设计MIME(Multipurpose Internet Mail Extension)是为了解决在不同的电子邮件系统之间搬移报文时存在的问题。

## 2. URI

- 统一资源标识符(Uniform Resource Identifier)
- URI有两种形式
  - URL(统一资源定位符)
    - 描述了一台特定服务器上某资源的特定位置
    - 第一部分为scheme，如: `http://`
    - 第二部分为host，如: `www.baidu.com`
    - 第三部分为path，如: `images/jingyefu.jpg`
  - URN(统一资源名)
    - URN是作为特定内容的唯一名称使用，与目前的资源所在地无关
    - URN还处于试验阶段
    - `RFC 2141`用URN表示为`urn:ietf:rfc:2141`

## 3. 报文

包括以下三个部分：

1. 起始行
   1. 
2. 首部字段
3. 主体

