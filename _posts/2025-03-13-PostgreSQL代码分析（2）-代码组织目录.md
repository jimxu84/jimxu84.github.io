---
layout: post
title: PostgreSQL代码分析（2） - SQL的执行
date: 2025-03-13 08:21:54
tags: Database, PostgreSQL
---

# 前言

上一篇我们介绍了代码的目录结构，这一篇我们将正式开始PostgreSQL的代码之旅。本文主要介绍SQL执行过程中的主要流程和函数，

# PostgresMain

PostgreSQL服务采用的是进程模型， 当有新的客户连接后，会创建一个独立进程来处理用户请求。而这个进程运行的主要代码就是PostgresMain函数，该函数用于接收用户输入的SQL命令并调用后续函数执行SQL

## 流程
1. 处理相关信号： SIGHUP/SIGINT/SIGTERM/SIGQUIT/SIGPIPE/SIGUSER1/SIGUSER2/SIGFPE
2. 基础初始化（BaseInit）：
3. 初始化Postgres（InitPostgres）:
4. 创建内存上下文
5. 异常处理逻辑
6. 循环读取网络输入信息（ReadCommand），根据消息类型做进一步处理，例如查询消息（firstchar='Q'）, 调用exec\_simple\_query

# exec_simple_query

SQL语句被后端接收后，需要经历SQL词法与语法解析，优化，最后选择最优的执行计划去执行，将结果返回给用户，该函数就是这个流程来完成的。

## 流程
1. 记录状态信息
2. 开启事务命令（start\_xact\_command）
3. 词法、语法分析，生成语法树 (pg\_parse\_query)
4. 循环处理每一颗树，一般情况一个SQL语句就是一颗树 (步骤5~8)
5. 如果需要，设置快照（PushActiveSnapshot）
6. 查询重写(pg\_analyze\_and\_rewrite\_fixedparams)
7. 生成查询计划（pg\_plan\_query）
8. 查询执行（CreatePortal/PortalRun）
9. 关闭事务命令(finish\_xact\_command)

![查询执行流程图](images/exec_simple_query.png)
