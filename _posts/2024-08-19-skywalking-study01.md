---
layout:     post
title:      "SkyWalking初探&浅析"
subtitle:   "SkyWalking初探&浅析"
date:       2024-08-19
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - SkyWalking
---

## APM系统概述

### 什么是APM系统？

目前主流的产品借助Google的Dapper论文实现的，一下是Dapper的翻译版本：

[Dapper，大规模分布式系统的跟踪系统](https://bigbully.github.io/Dapper-translation/ )

- 日志Logs
- 指标 Metrics
- 链路追踪 Traces

### 主流的APM系统

- 日志 ELK Stack
- 指标 Prometheus
- 链路追踪 SkyWalking

### OpenTracing 标准

[OpenTracing 标准](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/README.md)



## SkyWalking概述

### 基本概念

#### 为什么使用SkyWalking？

- 核心功能
- 特点

#### 关键概念

- 服务
- 服务实例
- 端点

### 架构

#### 组件

- 探针 Agent
- 服务端 OAP
- 存储 Storage
- 用户界面 UI

#### 设计目标

- 保持可观测性
- 拓扑结构
- 轻量级
- 可插拔
- 可移植

## 快速入门

### 部署OAP服务



### UI可视化



### 基于Agent监控SpringBoot应用

- 配置文件
- 启动脚本
- IDE开发工具



## 分布式调用链标准 OpenTracing

### 数据模型

- Trace
- Span
- SpanContext

## SkyWalking原理与架构设计

### 架构设计

#### 自动采集 & 无侵入

#### 跨进程传递Context

#### TraceId唯一性

#### 性能影响

### 实现原理

- Agent 与 Plugin

- TraceSegment

  > 一个TraceSegment是一个Trace在一个进程内所有Span的集合，如果多个线程协同产生1个Trace（例如多次RPC调用不同的方法），他们只会共同创建1个TraceSegment

  - EntrySpan
  - LocalSpan
  - ExitSpan

- TraceId设计

## Java Agent

### 基本概念

#### 自启动方式

##### 静态启动

- JVM参数 -javaagent 挂载Agent
- 入口方法：premain()
- 字节码操作限制
  - 符合字节码规范，对字节码种任意修改
- 适用场景：需要对字节码进行大量修改 APM
- SkyWalking目前仅支持该种方式

##### 动态附加

- JVM运行时使用Attach API挂载Agent
- 入口方法：agentmain()
- 字节码操作限制
  - 不能增减父类，不能增加接口，不能增减字段
- 适用场景：系统诊断 阿里Arthas

