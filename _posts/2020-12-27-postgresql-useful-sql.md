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


## oid関連

```
SELECT * FROM pg_class WHERE relname = 'pg_cast_oid_index'
```

## psql関連

- [よく使うpsqlの便利技10選](https://masahikosawada.github.io/2018/03/16/%E3%82%88%E3%81%8F%E4%BD%BF%E3%81%86psql%E3%81%AE%E4%BE%BF%E5%88%A9%E6%8A%8010%E9%81%B8/)

