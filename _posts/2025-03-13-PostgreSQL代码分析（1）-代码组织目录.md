---
layout: post
title: "PostgreSQL代码分析（1） - 代码组织目录"
date: 2025-03-13 05:58:05
tags: Database, PostgreSQL
---

# 前言

我初次接触PostgreSQL大概是在2006年左右，那时候在学习数据库伦理这门课程，但是当时教学实验是用的SQL Server, 而我的电脑因为安装的Linux系统，无法安装SQL Server，所以就搜索到MySQL与PostgreSQL等开源数据库，我已经记不清选PostgreSQL的具体原因了。后面课程结束了，后面几年也没有太关注过它。

毕业后，阴差阳错的进入关系数据库内核研发领域，我们在研发某些特性时也会对比一下PostgreSQL和MySQL。但也仅仅是使用一下，没有做代码级别的深入研究。

第一次研究代码大概是在2012年初，那时我已经工作了三年多，正好赶上PostgreSQL中国用户会创立，想着跟大牛多学习学习，就把代码拉下来，做了一个简单了解，并贡献了一篇关于调试环境搭建的文章。但是后来由于各种原因，没有继续下来。

这次闲下来了，有大把时间来研究代码了，所以有了这篇以及后续相关文章。

# 获取代码

因为仓库中某些对象太大，导致git clone失败，所以先修改.gitconfig，在文件末尾添加下面内容

```
[http]
        postBuffer = 5242880000
        version = HTTP/1.1
```

使用下面命令获取代码
```
git clone https://git.postgresql.org/git/postgresql.git
```

下载到本地，进入postgresql目录下，会看到一些脚本和目录，其中src就是代码目录。

# src
| 目录名    | 描述                                                 |
|-----------|------------------------------------------------------|
| backend   | 后端主要代码目录，包括查询，存储，事务等各个模块功能 |
| bin       | 二进制目录，包括initdb, pg_ctl等                     |
| common    | 通用功能如: 加密，归档，压缩等                       |
| include   | 头文件目录                                           |
| interface | libpq/ecpg 接口                                      |
| pl        | plperl/plpgsql/plpython等相关代码                    |
| port      | 跨平台移植相关代码                                   |
| timezone  | 时区相关代码                                         |
| tools     | 构建相关脚本                                         |


# backend
| 目录名       | 描述                                                     |
|--------------|----------------------------------------------------------|
| access       | 各种索引扫描，表扫描，以及事务相关代码                   |
| archive      | wal归档相关代码                                          |
| backup       | 备份相关代码                                             |
| bootstrap    | 用于创建初始模板数据库                                   |
| catalog      | 系统表信息                                               |
| commands     | 系统命令实现代码                                         |
| executor     | 执行器实现代码                                           |
| foreign      | 外部数据                                                 |
| jit          | just-in-time                                             |
| lib          | 通用数据结构代码                                         |
| libpq        | 连接认证相关代码                                         |
| main         | postgres程序入口                                         |
| nodes        | 执行计划节点                                             |
| optimizer    | 优化器                                                   |
| parser       | sql解析器                                                |
| partitioning | 分区表                                                   |
| po           | 国际化相关po文件                                         |
| port         | 跨平台相关代码                                           |
| postmaster   | 后端服务代码，监听客户端连接并创建进程处理连接请求       |
| regex        | 正则表达式代码                                           |
| replication  | 主备复制                                                 |
| rewrite      | 查询重写                                                 |
| statistics   | 统计信息代码                                             |
| storage      | 存储相关代码                                             |
| tcop         | 查询处理执行控制模块                                     |
| tsearch      | 全文搜索                                                 |
| utils        | 通用功能实现: 如函数功能管理，内存上下文管理，缓存管理等 |


# 后续

本文简单介绍一下代码目录及主要功能，方便后面研究代码或调试时能够快速定位到相关文件。后续会针对不同模块做更详细的学习。
