---
layout:     post
title:      "CSN snapshot"
date:       2025-04-13 12:00:00
author:     "Dazzle"
header-style: text
catalog: true
tags:
    - database
---

# 1. Postgres中的传统快照

## 1.1 快照组成

快照的核心包含三个部分：xmin，xmax，xip_list(活跃事务列表)

- xmin：最早仍然活跃的事务的txid。所有比它更早的事务txid < xmin要么已经提交并可见，要么已经回滚并生成死元组
- xmax：第一个尚未分配的txid。所有txid ≥ xmax的事务在获取快照时尚未启动，因而其结果对当前事务不可见
- xip_list：获取快照时活跃事务的txid列表。该列表仅包括xmin与xmax之间的txid

## 1.2 获取快照

1. 获取ProcArrayLock的shared锁
2. 从共享内存中获取（latestCompletedXid + 1）作为xmax
3. 遍历procArray，获取每个连接的pgxact（记录事务相关信息），把pgxact→xid记录在xip_list中
4. 最小的xid当做xmin
5. 释放ProcArrayLock

## 1.3 可见性检查

postgres中mvcc可见性检查在`HeapTupleSatisfiesMVCC` 中执行，根据Tuple头部的t_xmin、t_xmax、t_cid、clog和事务快照来判断tuple的可见性，规则复杂，下面简单讨论

### 1.3.1 t_xmin的状态为ABORTED

```sql
            /* 创建元组的事务已经中止 */
Rule 1:     IF t_xmin status is ABORTED THEN
                RETURN Invisible
            END IF
```

### 1.3.2 t_xmin的状态为IN_PROGRESS

```sql
            /* 创建元组的事务正在进行中 */
            IF t_xmin status is IN_PROGRESS THEN
                /* 当前事务自己创建了本元组 */
                IF t_xmin = current_txid THEN
                    /* 该元组没有被标记删除，则应当看见本事务自己创建的元组 */
Rule 2:             IF t_xmax = INVALID THEN 
                        RETURN Visible /* 例外，被自己创建的未删元组可见 */
Rule 3:             ELSE  
                    /* 这条元组被当前事务自己创建后又删除掉了，故不可见 */
                        RETURN Invisible
                    END IF
Rule 4:         ELSE   /* t_xmin ≠ current_txid */
                    /* 其他运行中的事务创建了本元组 */
                    RETURN Invisible
                END IF
            END IF
```

### 1.3.3 t_xmin的状态为COMMITTED

```sql
            /* 创建元组的事务已经提交 */
            IF t_xmin status is COMMITTED THEN
                /* 创建元组的事务在获取的事务快照中处于活跃状态，创建无效，不可见 */
Rule 5:         IF t_xmin is active in the obtained transaction snapshot THEN
                      RETURN Invisible
                /* 元组被删除，但删除元组的事务中止了，删除无效，可见 */
                /* 创建元组的事务已提交，且非活跃，元组也没有被标记为删除，则可见 */
Rule 6:         ELSE IF t_xmax = INVALID OR status of t_xmax is ABORTED THEN
                      RETURN Visible
                /* 元组被删除，但删除元组的事务正在进行中，分情况 */
                ELSE IF t_xmax status is IN_PROGRESS THEN
                    /* 如果恰好是被本事务自己删除的，删除有效，不可见 */
Rule 7:             IF t_xmax =  current_txid THEN
                        RETURN Invisible
                    /* 如果是被其他事务删除的，删除无效，可见 */
Rule 8:             ELSE  /* t_xmax ≠ current_txid */
                        RETURN Visible
                    END IF
                /* 元组被删除，且删除元组的事务已经提交 */
                ELSE IF t_xmax status is COMMITTED THEN
                    /* 删除元组的事务在获取的事务快照中处于活跃状态，删除无效，不可见 */
Rule 9:             IF t_xmax is active in the obtained transaction snapshot THEN
                        RETURN Visible
Rule 10:            ELSE /* 删除有效，可见 */
                        RETURN Invisible
                    END IF
                 END IF
            END IF
```

规则很多，核心就是依赖snapshot、clog等判断t_xmin, t_xmax对应的事务是否已经提交

## 1.4 存在问题

快照获取慢，快照获取需要持有ProcArrayLock锁，而ProcArrayLock又是一个非常重要的锁，很多时候都需要持有LW_EXCLUSIVE级别的锁，比如

- commit/abort
- analyze
- vacuum
- 新、旧链接
- ……

这会影响事务的并发，于是不需要ProcArrayLock的csn快照应运而生

# 2. CSN 快照

csn快照的性能相对于普通快照来说会有一定的提升，与传统快照相比有以下优势：

| 传统快照 | CSN 快照 |
| --- | --- |
| 需遍历 ProcArray 收集所有活跃事务的 XID，构建 xmin/xmax 范围 | 仅需记录当前全局 CSN 值（单次原子读取） |
| 时间复杂度：O(n)（n=并发事务数） | 时间复杂度：O(1) |
| 高并发时可能成为瓶颈（如数千连接） |  |

**场景示例**：

在 1000 个并发事务中，传统快照需扫描所有事务状态，而 CSN 快照只需读取一个全局计数器。

## 2.1 快照结构

与传统快照一样，csn快照也包含3个核心部分，xmin，xmax，csn

- xmin：最早仍然活跃的事务的txid。
- xmax：第一个尚未分配的txid。
- csn：提交序列号（核心优化点），表示快照创建时的全局提交序列号

共享内存中记录一个int64的next_csn值，事务提交时next_csn作为该事务的提交csn，并把xid与csn的映射记录在csnlog中，同时使next_csn++。事务abort时直接分配一个特殊值标记（如csn=-1）标记该事务为abort。另外还需要新增一个csnlog，用于记录xid与csn的映射关系。

## 2.2 快照获取流程

- csn：与传统快照获取需要遍历procArray获取活跃事务列表不同，csn快照直接使用原子操作从共享内存中获取next_csn作为快照中的csn值，避免了加锁遍历procArray。
- xmax：xmax仍然可以类似于传统快照的xmax一样从共享内存中获取（latestCompletedXid + 1）
- xmin：为了避免遍历procArray，xmin获取需要做一些额外处理，在共享内存中新增一个变量 —— oldestActiveXid，更新逻辑如下：
    
    在事务commit/abort时，对比本事无的xid与oldestActiveXid是否相等
    
    1. 不相等：什么都不做
    2. 相等：根据clog，获取大于oldestActiveXid的最老的activeXid，并以此来更新oldestActiveXid

## 2.2 mvcc判断逻辑

如上所述，每个事务在启动时会取得一个xid，提交时会取得一个提交csn，同时推进全局next_csn值，xid和csn的映射关系保存在csnlog文件中。

简化后判断xid的可见性逻辑如下：

```sql
/* 规则1：超出快照范围的事务绝对不可见 */
    IF xid >= snapshot.xmax THEN
        RETURN False;
    END IF;

    /* 规则2：早于所有活跃事务的XID */
    IF xid < snapshot.xmin THEN
        /* 需检查clog确认提交状态 */
        IF GetTransactionStatus(xid) = COMMITTED THEN
            RETURN True;  /* 已提交的旧事务可见 */
        ELSE
            RETURN False; /* 未提交/中止的旧事务不可见 */
        END IF;
    END IF;

    /* 规则3：xid在[xmin, xmax)区间内 */
    /* 获取该xid对应的CSN（可能需等待提交完成） */
    xid_csn = GetCSN(xid, wait_for_commit:=True);

    /* 规则3.1：事务未提交（CSN无效或为未来值） */
    IF xid_csn == INVALID_CSN OR xid_csn >= snapshot.csn THEN
        RETURN False;
    END IF;

    /* 规则3.2：事务已提交且CSN<快照CSN */
    RETURN True;
```

整体判断逻辑如上，可以结合clog与csnlog，csn快照的作用主要发挥在活跃事务列表的判断逻辑中（规则3），通过GetCSN获取xid对应的csn，如果对应的csn真在提交，则需要等待事务提交结束

## 2.3 其他

1. 数据库崩溃恢复：由上面的判断流程可见，csn的作用是替代了活跃事务列表（规则3），由于数据库崩溃后就不会有活跃事务了，所以csn可以从头开始，恢复过程回放的commit日志对应的xid的csn标记为frozen（都可见）
2. RO节点的csn：RO节点的csn与RW节点同一个xid对应的csn不一定相同，RO结点在回放RW的commit日志时维护自己的next_csn值

# 3. reference

1. https://www.postgresql.org/message-id/CA%2BCSw_tEpJ%3Dmd1zgxPkjH6CWDnTDft4gBi%3D%2BP9SnoC%2BWy3pKdA%40mail.gmail.com
2. https://pg-internal.vonng.com/#/ch5?id=_55-%e4%ba%8b%e5%8a%a1%e5%bf%ab%e7%85%a7
3. https://docs.polardbpg.com/1653230754878/PolarDB-for-PostgreSQL/CSN.html