---
layout: post
title: PostgreSQLでよく使うコマンド・SQLまとめ
tags:
- postgresql
- sql
- summary
---

## pgbench関連

- データ格納

```
psql -p 5432 -c "CREATE DATABASE pgbench;"
pgbench -i -s 1 pgbench
pgbench -c 1 -T 30 pgbench
```

- データ更新

```
UPDATE pgbench_accounts SET bid = 10 WHERE aid = 1;
```

## pg_locks関連

```
SELECT a.datname, relation::regclass, a.pid, l.locktype, l.page, l.tuple, l.virtualxid, l.transactionid, l.classid, l.objid, l.objsubid, l.virtualtransaction, l.mode, l.granted, l.fastpath
    FROM pg_stat_activity a
        JOIN pg_locks l ON l.pid = a.pid
        ORDER BY l.pid;
```

## バックエンド関連

```
SELECT pg_backend_pid();
```
