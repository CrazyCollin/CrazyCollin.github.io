---
title: "DNS"
date: 2022-04-30T17:18:32+08:00
lastmod: 2022-04-30T17:18:32+08:00
author: ["Collin"]
categories: 
- 计算机网络
tags: 
- 计网
description: ""
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---


> [https://juejin.cn/post/6918894984414363655](https://juejin.cn/post/6918894984414363655)

> [https://juejin.cn/post/6919755385330991112#heading-7](https://juejin.cn/post/6919755385330991112#heading-7)

域名解析协议
<a name="c3Ffb"></a>
## DNS层次结构

- **根域名服务器**
- **顶级域名服务器**
- **权威域名服务器**
- **本地域名服务器**
   - **不在DNS层次结构中**
   - **是电脑解析时的默认域名服务器**
<a name="PsXOf"></a>
## DNS查询步骤
1）首先搜索**浏览器的 DNS 缓存**，缓存中维护一张域名与 IP 地址的对应表；<br />2）若没有命中，则继续搜索**操作系统的 DNS 缓存**；<br />3）若仍然没有命中，则操作系统将域名发送至**本地域名服务器**，本地域名服务器查询自己的 DNS 缓存，查找成功则返回结果（注意：主机和本地域名服务器之间的查询方式是**递归查询**）；<br />4）若本地域名服务器的 DNS 缓存没有命中，则本地域名服务器向上级域名服务器进行查询，通过以下方式进行**迭代查询**（注意：本地域名服务器和其他域名服务器之间的查询方式是迭代查询，防止根域名服务器压力过大）：

- 首先本地域名服务器向**根域名服务器**发起请求，根域名服务器是最高层次的，它并不会直接指明这个域名对应的 IP 地址，而是返回顶级域名服务器的地址，也就是说给本地域名服务器指明一条道路，让他去这里寻找答案
- 本地域名服务器拿到这个**顶级域名服务器**的地址后，就向其发起请求，获取**权限域名服务器**的地址
- 本地域名服务器根据权限域名服务器的地址向其发起请求，最终得到该域名对应的 IP 地址

4）本地域名服务器将得到的 IP 地址返回给操作系统，同时自己将 IP 地址缓存起来<br />5）操作系统将 IP 地址返回给浏览器，同时自己也将 IP 地址缓存起来<br />6）至此，浏览器就得到了域名对应的 IP 地址，并将 IP 地址缓存起来<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/25605701/1649503386128-fb352a57-bdd4-4fcd-9d12-acd98e4defce.png#clientId=u08b34141-8a70-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=477&id=tGXo2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=953&originWidth=1304&originalType=binary&ratio=1&rotation=0&showTitle=false&size=430893&status=done&style=none&taskId=u7008a28a-bf65-4ddf-a3ae-a03c22feb04&title=&width=652)
<a name="yrdlj"></a>
## DNS解析器
进行DNS查询的主机和软件都叫做DNS解析器，用户使用的个人电脑和工作栈都属于DNS解析器。<br />DNS解析器是DNS查询的第一站，其负责与发出初始请求的客户端打交道。<br />解析器启动查询序列，最终将URL转换得到对应的IP地址

<a name="MPbNn"></a>
## DNS缓存
DNS缓存又称DNS解析器缓存，是由**操作系统维护的临时数据库，拥有最近访问网站和其他internet域的访问记录**
<a name="LyL0M"></a>
### DNS缓存工作方式
在浏览器向外部发送请求之前拦截每个请求并在dns缓存中查找域名，该缓存中维护了最近的域名列表，以及dns首次发出请求时dns为其计算的域名
<a name="CGv8O"></a>
### DNS缓存方式
<a name="y4s1L"></a>
#### 浏览器缓存
将dns查询结果存储在浏览器缓存中，chrome://net-internals/#dns 查看
<a name="M6OOo"></a>
#### os内核缓存
<a name="vMrin"></a>
## DNS安全
