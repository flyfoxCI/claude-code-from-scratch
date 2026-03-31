# 第二部分：架构设计

## 章节

- [第 2 章：核心循环](ch02-query-loop.md)
- [第 3 章：消息系统](ch03-message-system.md)
- [第 19 章：状态管理架构](ch19-state-management.md)
- [第 20 章：CLI 与 REPL](ch20-cli-repl.md)

## 内容概述

深入解析 Claude Code 的核心架构，包括：
- Query Engine 核心循环
- 消息系统设计
- 状态管理模式
- CLI/REPL 实现

## 核心概念

- AsyncGenerator 异步循环模式
- 消息类型与流转
- 状态序列化与恢复
- 流式响应处理

## 相关图表

- [Agent 核心循环](../diagrams/agent-loop.svg)
- [架构概览](../diagrams/architecture-overview.svg)
- [状态管理](../diagrams/state-management.svg)
