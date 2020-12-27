---
layout: post
title: PostgreSQLのheavy weight lockに関する内部構造メモ
tags:
- postgresql
- lock
- internal
---

## TODO

- ロックの仕組みが全然分かっていない
  - このメモも全然ダメ
    - fast path, ProcSleep, localでlock情報を保持している理由など謎だらけ

## PostgreSQLのロックに関するソース

- src/backend/storage/lmgr/README
- src/backend/access/heap/README.tuplock

### ロックの分類

- lock method: DEFUALT or USER
- lock type: read/write or shared/exclusive

### ロック管理の方法

- per-lockable-object LOCK struct
  - 共有メモリ上のlock hash tableで管理
  - lock managerが保持している
  - lockを取得 or 取得予定のlockableなオブジェクトごとに存在
  - それぞれの意味は、src/include/strorage/lock.hが詳しい

```
typedef struct LOCK
{
	/* hash key */
	LOCKTAG		tag;			/* unique identifier of lockable object */

	/* data */
	LOCKMASK	grantMask;		/* bitmask for lock types already granted */
	LOCKMASK	waitMask;		/* bitmask for lock types awaited */
	SHM_QUEUE	procLocks;		/* list of PROCLOCK objects assoc. with lock */
	PROC_QUEUE	waitProcs;		/* list of PGPROC objects waiting on lock */
	int			requested[MAX_LOCKMODES];	/* counts of requested locks */
	int			nRequested;		/* total of requested[] array */
	int			granted[MAX_LOCKMODES]; /* counts of granted locks */
	int			nGranted;		/* total of granted[] array */
} LOCK;
```

- per-lock-and-requestor PROCLOCK struct
  - 共有メモリ上のPROCLOCK hash tableで管理
  - lock managerが保持している

```
typedef struct PROCLOCK
{
	/* tag */
	PROCLOCKTAG tag;			/* unique identifier of proclock object */

	/* data */
	PGPROC	   *groupLeader;	/* proc's lock group leader, or proc itself */
	LOCKMASK	holdMask;		/* bitmask for lock types currently held */
	LOCKMASK	releaseMask;	/* bitmask for lock types to be released */
	SHM_QUEUE	lockLink;		/* list link in LOCK's list of proclocks */
	SHM_QUEUE	procLink;		/* list link in PGPROC's list of proclocks */
} PROCLOCK;
```

- それぞれのbackendが管理するLOCALLOCK struct
  - lockable objectとlock modeごとに保持する
  - lockを保持 or 保持予定のobjectoごと

```
typedef struct LOCALLOCK
{
	/* tag */
	LOCALLOCKTAG tag;			/* unique identifier of locallock entry */

	/* data */
	uint32		hashcode;		/* copy of LOCKTAG's hash value */
	LOCK	   *lock;			/* associated LOCK object, if any */
	PROCLOCK   *proclock;		/* associated PROCLOCK object, if any */
	int64		nLocks;			/* total number of times lock is held */
	int			numLockOwners;	/* # of relevant ResourceOwners */
	int			maxLockOwners;	/* allocated size of array */
	LOCALLOCKOWNER *lockOwners; /* dynamically resizable array */
	bool		holdsStrongLockCount;	/* bumped FastPathStrongRelationLocks */
	bool		lockCleared;	/* we read all sinval msgs for lock */
} LOCALLOCK;
```

### Lock Managerの内部ロック方法

- 8.2までは1つのLWLockで管理していたが、ボトルネックになりがち
  - partitionに分割して、独立したLWLockで管理
    - LOCKTAGのhash値でpartitionを決定


### fast pathロック

- 限られた数のロックを記録する方法
  - DEFAULT lock method
  - database relation単位でロック
  - 競合が発生しにくいweak lock(AccessSharedLock, RowSharedLock, RowExclusiveLock)
  - よくアクセスされるが、競合が少ないロック
  - 簡単に競合が発生しているかどうかを確認できる

- fast pathを使うパターン
  - (1) DDL(CLUSTER/ALTER TABLE/DROP...)を実行した場合
    - SELECT, INSERT, UPDATE, DELETEなど、並列実行されるものは競合しやすいので対象外
  - (2) VXIDをロックする場合
    - CREATE INDEX CONCURRENTLY, Hot Standbyくらいで、vxidの所有者くらいしかロックしない


### ロック周りの処理フロー例

#### 例えば、updateでのRowExclusiveLock

- 1つ目のセッションでupdate

```
pgbench=# BEGIN;
BEGIN
pgbench=*# UPDATE pgbench_accounts SET bid = 10 WHERE aid = 1;
UPDATE 1
```

- 別セッションでロック状況を確認
  - relation, virtualxid, transactionidへのロックが発生している

```
pgbench=# SELECT a.datname, relation::regclass, a.pid, l.locktype, l.page, l.tuple, l.virtualxid, l.transactionid, l.classid, l.objid, l.objsubid, l.virtualtransaction, l.mode, l.granted, l.fastpath
    FROM pg_stat_activity a
        JOIN pg_locks l ON l.pid = a.pid
        ORDER BY l.pid;
 datname |         relation          |  pid  |   locktype    | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |       mode       | granted | fast
path
---------+---------------------------+-------+---------------+------+-------+------------+---------------+---------+-------+----------+--------------------+------------------+---------+-----
-----
 pgbench | pgbench_accounts          | 46601 | relation      |      |       |            |               |         |       |          | 5/72               | RowExclusiveLock | t       | t
 pgbench |                           | 46601 | virtualxid    |      |       | 5/72       |               |         |       |          | 5/72               | ExclusiveLock    | t       | t
 pgbench | pgbench_accounts_pkey     | 46601 | relation      |      |       |            |               |         |       |          | 5/72               | RowExclusiveLock | t       | t
 pgbench |                           | 46601 | transactionid |      |       |            |           762 |         |       |          | 5/72               | ExclusiveLock    | t       | f
```

- 2つ目のセッションでupdate

```
pgbench=# BEGIN;
BEGIN
pgbench=*# UPDATE pgbench_accounts SET bid = 10 WHERE aid = 1;
```

- ロック状況を確認
  - 2つ目のセッションでは、下記の2つのロックが発生している
    - transactionid=762(1つ目のセッション)へのShareLock
    - tupleへのExclusiveLock

```
pgbench=# SELECT a.datname, relation::regclass, a.pid, l.locktype, l.page, l.tuple, l.virtualxid, l.transactionid, l.classid, l.objid, l.objsubid, l.virtualtransaction, l.mode, l.granted, l.fastpath
    FROM pg_stat_activity a
        JOIN pg_locks l ON l.pid = a.pid
        ORDER BY l.pid;
 datname |         relation          |  pid  |   locktype    | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |       mode       | granted | fast
path
---------+---------------------------+-------+---------------+------+-------+------------+---------------+---------+-------+----------+--------------------+------------------+---------+-----
-----
 pgbench | pgbench_accounts_pkey     | 46601 | relation      |      |       |            |               |         |       |          | 5/72               | RowExclusiveLock | t       | t
 pgbench |                           | 46601 | virtualxid    |      |       | 5/72       |               |         |       |          | 5/72               | ExclusiveLock    | t       | t
 pgbench |                           | 46601 | transactionid |      |       |            |           762 |         |       |          | 5/72               | ExclusiveLock    | t       | f
 pgbench | pgbench_accounts          | 46601 | relation      |      |       |            |               |         |       |          | 5/72               | RowExclusiveLock | t       | t

 pgbench | pgbench_accounts          | 49299 | relation      |      |       |            |               |         |       |          | 4/61               | RowExclusiveLock | t       | t
 pgbench |                           | 49299 | transactionid |      |       |            |           762 |         |       |          | 4/61               | ShareLock        | f       | f
 pgbench |                           | 49299 | virtualxid    |      |       | 4/61       |               |         |       |          | 4/61               | ExclusiveLock    | t       | t
 pgbench | pgbench_accounts          | 49299 | tuple         |   21 |     6 |            |               |         |       |          | 4/61               | ExclusiveLock    | t       | f
 pgbench |                           | 49299 | transactionid |      |       |            |           763 |         |       |          | 4/61               | ExclusiveLock    | t       | f
 pgbench | pgbench_accounts_pkey     | 49299 | relation      |      |       |            |               |         |       |          | 4/61               | RowExclusiveLock | t       | t
```

- 1つ目のセッションでCOMMITすると、tupleレベルでは何もない

```
pgbench=# SELECT a.datname, relation::regclass, a.pid, l.locktype, l.page, l.tuple, l.virtualxid, l.transactionid, l.classid, l.objid, l.objsubid, l.virtualtransaction, l.mode, l.granted, l.fastpath
    FROM pg_stat_activity a
        JOIN pg_locks l ON l.pid = a.pid
        ORDER BY l.pid;
 datname |         relation          |  pid  |   locktype    | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |       mode       | granted | fast
path
---------+---------------------------+-------+---------------+------+-------+------------+---------------+---------+-------+----------+--------------------+------------------+---------+-----
-----
 pgbench | pgbench_accounts_pkey     | 49299 | relation      |      |       |            |               |         |       |          | 4/61               | RowExclusiveLock | t       | t
 pgbench | pgbench_accounts          | 49299 | relation      |      |       |            |               |         |       |          | 4/61               | RowExclusiveLock | t       | t
 pgbench |                           | 49299 | virtualxid    |      |       | 4/61       |               |         |       |          | 4/61               | ExclusiveLock    | t       | t
 pgbench |                           | 49299 | transactionid |      |       |            |           763 |         |       |          | 4/61               | ExclusiveLock    | t       | f
```

#### 上記フローのときのLockに関する関数


##### 1.InitLocks()で共有メモリを初期化

- PROCLOCK hash: (最大バックエンド数+最大prepare数) x トランザクションあたりの最大ロック数
- FastPathStrongRelationLocks
- LOCKLOCAL hash: バックエンドごと

##### 2.ロックをリクエスト

- ParseOpenTable() -> LockRelationOid() -> LockAcquireExtended()で表ロックを"RowExclusiveLock"獲得
  - パース処理:ParseOpenTable()のときにテーブルをオープン。オープンするときにモードを指定する。この場合、UPDATEなので"RowExclusiveLock"
  - LockRelationOid()のときにLockTagTypeを `LOCKTAG_RELATION,			/* whole relation */` 変更したり、ロック対象のデータベースoid, リレーションidを付与

```
#define RowExclusiveLock		3	/* INSERT, UPDATE, DELETE */
```

- LockAcquireExtended()内がメインの処理
  - LOCALLOCKのhashを見て、既に同じロックを獲得していなければ、作成する

- fast path経由でロックを取得できるか確認
  - fast pathとれなければ、普通のロックを取得する
    - // よく分からない。バックエンドごとにslotを用意していて、ロックしたい対象を保存している
    - // どこかでまとめて、ロックが取得される？

- virtualxid(?)については、XactLockTableInsert()経由でLockAcquireExtendedが呼ばれていそう
  - "ExclusiveLock"で呼ばれる

- tupleは、LockTuple()経由で呼ばれる
  - tuple自体は、ExclusiveLock取得出来ている

- tupleを操作しているtxのExclusiveLockは、XactLockTableWait()経由で呼ばれる
  - // ヒープを見て、他のtxが実行していたら、そのtx自体をロックする仕組み？
  - WaitOnLock()->ProcSleep()が呼ばれて、ロック待ちを行う
