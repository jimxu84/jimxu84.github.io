---
layout: post
title: "PostgreSQL代码分析（3） - 事务管理"
date: 2025-04-02 12:11:28
tags: Database, PostgreSQL
---

# 前言
上一篇我们介绍了SQL语句执行的流程，里面有两个start\_xact\_command和finish\_xact\_command两个函数，用于开始和结束事务，这两个函数是对底层事务处理函数的包装，实际事务相关代码是在xact.c中实现的。本文我们将详细介绍一下语句执行过程中的事务管理。

# SQL命令
默认情况下，我们每执行一个SQL语句，就会自动开启一个事务，语句成功执行则提交或者因为某个错误而终止。我们也可以通过BEGIN/START TRANSACTION/COMMIT/ABORT/SAVEPOINT等SQL命令来显示的控制事务。例如：
```sql
/* 开启事务并提交 */
BEGIN;
insert into t1 values (1,1);
update t1 set c2=2 where c1=1;
COMMIT;

/* 回滚 */
START TRANSACTION;
delete from t1 where c1=1;
ROLLBACK;

/* 嵌套事务 */
BEGIN;
insert into t1 values (2,2);
SAVEPOINT sp1;
insert into t1 values (3,3);
ROLLBACK to sp1;
COMMIT;
```

# 数据结构和变量
事务控制的主要结构是TransactionStateData, 默认的事务状态由TopTransactionStateData变量维护，当子事务创建时，会新创建一个TransactionStateData对象并赋给而CurrentTransactionState，其parent执行上层事务状态。

## 事务状态结构

```c++
typedef struct TransactionStateData
{
	FullTransactionId fullTransactionId;	/* my FullTransactionId */
	SubTransactionId subTransactionId;	/* my subxact ID */
	char	   *name;			/* savepoint name, if any */
	int			savepointLevel; /* savepoint level */
	TransState	state;			/* low-level state */
	TBlockState blockState;		/* high-level state */
	int			nestingLevel;	/* transaction nesting depth */
	int			gucNestLevel;	/* GUC context nesting depth */
	MemoryContext curTransactionContext;	/* my xact-lifetime context */
	ResourceOwner curTransactionOwner;	/* my query resources */
	MemoryContext priorContext; /* CurrentMemoryContext before xact started */
	TransactionId *childXids;	/* subcommitted child XIDs, in XID order */
	int			nChildXids;		/* # of subcommitted child XIDs */
	int			maxChildXids;	/* allocated size of childXids[] */
	Oid			prevUser;		/* previous CurrentUserId setting */
	int			prevSecContext; /* previous SecurityRestrictionContext */
	bool		prevXactReadOnly;	/* entry-time xact r/o state */
	bool		startedInRecovery;	/* did we start in recovery? */
	bool		didLogXid;		/* has xid been included in WAL record? */
	int			parallelModeLevel;	/* Enter/ExitParallelMode counter */
	bool		parallelChildXact;	/* is any parent transaction parallel? */
	bool		chain;			/* start a new block after this one */
	bool		topXidLogged;	/* for a subxact: is top-level XID logged? */
	struct TransactionStateData *parent;	/* back link to parent */
} TransactionStateData;

typedef TransactionStateData *TransactionState;

static TransactionStateData TopTransactionStateData = {
	.state = TRANS_DEFAULT,
	.blockState = TBLOCK_DEFAULT,
	.topXidLogged = false,
};

static TransactionState CurrentTransactionState = &TopTransactionStateData;
```
## 事务块状态
事务块状态用于保证上层事务管理SQL正常执行，当用户执行BEGIN/COMMIT等命令时，会改变该状态

```c++
typedef enum TBlockState
{
	/* not-in-transaction-block states */
	TBLOCK_DEFAULT,				/* idle */
	TBLOCK_STARTED,				/* running single-query transaction */

	/* transaction block states */
	TBLOCK_BEGIN,				/* starting transaction block */
	TBLOCK_INPROGRESS,			/* live transaction */
	TBLOCK_IMPLICIT_INPROGRESS, /* live transaction after implicit BEGIN */
	TBLOCK_PARALLEL_INPROGRESS, /* live transaction inside parallel worker */
	TBLOCK_END,					/* COMMIT received */
	TBLOCK_ABORT,				/* failed xact, awaiting ROLLBACK */
	TBLOCK_ABORT_END,			/* failed xact, ROLLBACK received */
	TBLOCK_ABORT_PENDING,		/* live xact, ROLLBACK received */
	TBLOCK_PREPARE,				/* live xact, PREPARE received */

	/* subtransaction states */
	TBLOCK_SUBBEGIN,			/* starting a subtransaction */
	TBLOCK_SUBINPROGRESS,		/* live subtransaction */
	TBLOCK_SUBRELEASE,			/* RELEASE received */
	TBLOCK_SUBCOMMIT,			/* COMMIT received while TBLOCK_SUBINPROGRESS */
	TBLOCK_SUBABORT,			/* failed subxact, awaiting ROLLBACK */
	TBLOCK_SUBABORT_END,		/* failed subxact, ROLLBACK received */
	TBLOCK_SUBABORT_PENDING,	/* live subxact, ROLLBACK received */
	TBLOCK_SUBRESTART,			/* live subxact, ROLLBACK TO received */
	TBLOCK_SUBABORT_RESTART,	/* failed subxact, ROLLBACK TO received */
} TBlockState;

```

## 事务状态

```c++
typedef enum TransState
{
	TRANS_DEFAULT,				/* idle */
	TRANS_START,				/* transaction starting */
	TRANS_INPROGRESS,			/* inside a valid transaction */
	TRANS_COMMIT,				/* commit in progress */
	TRANS_ABORT,				/* abort in progress */
	TRANS_PREPARE,				/* prepare in progress */
} TransState;
```

# 执行流程
不管是事务控制语句还是DML语句，都会调用start\_xact\_command和finish\_xact\_command, 其内部会调用StartTransactionCommand/CommitTransactionCommand设置相关事务状态，事务处理函数会根据当前事务状态来做相应处理，这两个函数会调用多次，首次调用start\_xact\_command时会将xact\_tarted变量设置成true, 并其在首次调用finish\_xact\_command时设置成false. 下面介绍执行不同SQL时调用这些函数时对事务状态的影响. User在客户端执行相关SQL语句，Postgres为内部处理函数，TransState表示事务相关幻术以及实际状态。

## COMMIT
![COMMIT](/images/2025/commit.png)

## ROLLBACK

![ROLLBACK](/images/2025/abort.png)

## SAVEPOINT
通过savepoint可以在一个事务内开启子事务，子事务开启后，能够回滚到某个savepoint,但是提交就只能对整个事务有效。相关调用函数以及状态转化如下图 
![SAVEPOINT](/images/2025/sp.png)
