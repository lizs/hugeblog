---
title: "IO"
date: 2018-04-18T15:34:35+08:00
draft: true
---

# **IO概述**

## 架构概述

[架构图](io_1.png)

IO是海外项目服务器框架，由大厅和游戏组成。大厅集群内TCP通信，大厅与游戏间HTTP通信。

* TCP网络层基于[mom](https://github.com/lizs/mom)/[netmsg](https://github.com/lizs/netmsg)/[gomsg](https://github.com/lizs/netmsg)
* 业务协议采用google protobuf
* 数据落地采用redis/mssql

## 说明

*无状态、有状态服务器：*

* 无状态服务器
<br>表明该服务器实例重启之后能继续为集群提供正确服务
* 有状态服务器
<br>相对于无状态服务器

*单点、多点服务器：*

* 单点，表明该服务器在集群中仅存在一个实例
* 多点，表明该服务器在及集群中可存在多个实例

本文档仅阐述IO的当前状态，对设计优化等不在本文说明。

## **大厅**

大厅由Gate/Auth/Cache/Chat/Exchanger组成。集群内通过Gate交互。

### **Gate(C++)**

网关，交互枢纽(基于[mom](https://github.com/lizs/mom))

* 转发客户端上行数据到后端
* 推送后端数据到客户端
* 转发集群内请求和推送

### **Auth(C#，单点无状态)**

账号身份验证服务器

* 账号注册
* 身份验证

Auth提供HTTP访问接口，拥有独立的数据库，提供账号注册和身份验证功能。客户端无论是采取IO账号登录，还是第三方账号登录，Auth都会为该次登录请求创建唯一Token，客户端携带该Token才能与大厅、游戏服务器进行正确交互。Token在每次有效登陆之后或所在Redis重启之后刷新。

### **Cache(C#，多点有状态)**

缓存，大厅业务服务器。

* 缓存大厅数据
* 处理大厅业务
* 数据落地(Redis+MSSQL)

Cache处理了所有非中心化（中心化业务放在Chat处理）的大厅业务，并驱动数据落地。数据根据落地方式分两大类：redis类、sql类。ID种子、排行榜数据落地在redis的rdb文件中，其它数据全部落地在mysql/mssql中。采用servicestack.redis、servicestack.ormlite插件与Redis、mssql交互。
<br>
有两类Redis实例：一类是无需落地，仅用作缓存；一类是需要落地rdb文件。排行榜、ID种子属于前者，其它高频修改的数据（如任务、道具）都会缓存在后者实例中，并设置为过期移除。

### **Chat(C#，单点有状态)**

中心服务器（名称待修改），处理中心化业务，如在线统计、聊天、好友。

### **Exchanger(Golang，单点无状态)**

交换服务器，对游戏集群和其它运维需求暴露Http接口，是游戏服务器与大厅集群交互的唯一途径。

## **游戏**

玩法业务服务器，通过Http与大厅低频交互。可以使用C++/C#/gomsg来开发游戏，因为三者的网络层mom/netmsg/gomsg兼容。

### **GameService**
游戏服务，负责游戏服务器分配

### **Game**
游戏玩法服务器