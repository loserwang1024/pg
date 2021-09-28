---
title: "空中换引擎 —— PostgreSQL不停机迁移数据"
linkTitle: "PgSQL不停机迁移"
date: 2018-02-06
author: |
  [冯若航](http://vonng.com)（[@Vonng](http://vonng.com/en/)）
description: >
  通常涉及到数据迁移，常规操作都是停服务更新。不停机迁移数据是相对比较高级的操作。
---

# 应用层逻辑复制迁移表


通常涉及到数据迁移，常规操作都是停服务更新。不停机迁移数据是相对比较高级的操作。

不停机数据迁移在本质上，可以视作由三个操作组成：

* 复制：将目标表从源库**逻辑复制**到宿库。
* 改读：将应用**读取路径**由源库迁移到宿库上。
* 改写：将应用**写入路径**由源库迁移到宿库上。

但在实际执行中，这三个步骤可能会有不一样的表现形式。



## 逻辑复制

使用逻辑复制是比较稳妥的做法，也有几种不同的做法：应用层逻辑复制，数据库自带的逻辑复制（PostgreSQL 10 之后的逻辑订阅），使用第三方逻辑复制插件（例如pglogical）。

几种逻辑复制的方法各有优劣，我们采用了应用层逻辑复制的方式。具体包括四个步骤：

#### 一、复制

- 在新库中fork老库目标表的模式，以及所有依赖的函数、序列、权限、属主等对象。
- 应用添加双写逻辑，同时向新库与老库中写入同样数据。
  - 同时向新库与老库写入
- 保证增量数据正确写入两个一样的库中。
- 应用需要正确处理全量数据不存在下的删改逻辑。例如改`UPDATE`为`UPSERT`，忽略`DELETE`。
- 应用读取仍然走老库。
- 出现问题时，回滚应用至原来的单写版本。

二、同步

- 老表加上表级排它锁 `LOCK TABLE <xxx> IN EXCLUSIVE MODE`，阻塞所有写入。
- 执行全量同步 `pg_dump | psql`
- 校验数据一致性，判断迁移是否成功。
- 出现问题时，简单清空新库中的对应表。

1. 改读
   - 应用修改为从新库中读取数据。
   - 出现问题时，回滚至从老库中读取的版本。
2. 单写
   - 观察一段时间无误后，应用修改为仅写入新库。
   - 出现问题时，回滚至双写版本。

### 说明

关键在**于阻塞全量同步期间对老表的写入**。这可以通过表级排它锁实现。


在对表进行了分片的情况下，锁表对业务造成的影响非常小。

一张逻辑表拆分成8192个分区，实际上一次只需要处理一个分区。

阻塞对八千分之一的数据写入约几秒到十几秒，业务上通常是可以接受的。

但如果是单张非常大的表，也许就需要特殊处理了。



## ETL函数

以下Bash函数接受三个参数，源库URL，宿库URL，以及待迁移的表名。

假设是源宿库都可连接，且目标表都存在。

```bash
function etl(){
    local src_url=${1}
    local dst_url=${2}
    local table_name=${3}

    rm -rf "/tmp/etl-${table_name}.done"
    
    psql ${src_url} -1qAtc "LOCK TABLE ${table_name} IN EXCLUSIVE MODE;COPY ${table_name} TO STDOUT;" \
    | psql ${dst_url} -1qAtc "LOCK TABLE ${table_name} IN EXCLUSIVE MODE; TRUNCATE ${table_name}; COPY ${table_name} FROM STDIN;"
    
    touch "/tmp/etl-${table_name}.done"
}
```

实际上虽然锁定了源表与宿表，但在实际测试中，管道退出时前后两个psql进程退出的timing并不是完全同步的。管道前面的进程比后面一个进程早了0.1秒退出。在负载很大的情况下，可能会产生数据不一致。

另一种更科学的做法是按照某一唯一约束列进行切分，锁定相应的行，更新后释放



## 物理复制

物理复制是通过回放WAL日志实现的复制，是数据库集簇层面的复制。

基于物理复制的迁移粒度很粗，仅适用于垂直分裂库时使用，会有极短暂的服务不可用。

使用物理复制进行数据迁移的流程如下：

- 复制，从主库拖出一台从库，保持流式复制。
- 改读：将应用读取路径从主库改为从库，但写入仍然写入主库。
  - 如果有问题，将应用回滚至读主库版本。
- 改写：将从库提升为主库，阻塞老库的写入，并立即重启应用，切换写入路径至新主库上。
  - 将不需要的表和库删除。
  - 这一步无法回滚（回滚会损失写入新库的数据）
