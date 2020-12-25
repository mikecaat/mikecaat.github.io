---
layout: post
title: Citusのdistributed deadlockについて調べたのでメモ
tags:
- citus
- distributed deadlock
---

## 概要

- 分散DBになると、各サーバを跨いだdeadlockが発生する可能性があるので対応が必要

- Citusは、wait-graphを作成して、deadlockを検知する方式を採用している
  - extensionを通じて、グローバルなtxをSQL経由で払い出す仕組みを作成
    - グローバルtx開始時に各ノードにこのSQLを発行しているように見える
  - そのため、分散しているノード間でも共通のtx-id体系でwait-graphを作成できる
  - // pg_locksだけだと、異なるノード間の同一txのロック取得待ち情報をつなげるのが難しい
    - // coordinator側で接続してきたプロセスと各ノード上のbackend_pidの紐付けを管理できれば、pg_locksでもいけそうだけど...
    - // client(≒tx id) -> coordinator上のpid -> 各ノードのbackend pid
      - // ただ、この場合、Clock-SIのようにcoordinator的なアクセス先が複数存在すると破綻する
      - // citusの方式だと、各ノードでグローバルtx idは払い出されるが、アクセス先ノードが固定であれば、一意のglobal tx idが払い出される

## deadlockの対策

- deadlockの対策でよくある方法は下記([参考](https://15721.courses.cs.cmu.edu/spring2020/notes/02-inmemory.pdf))
  - Deadlock Detection(悲観): デットロックが発生していないかを検知して、deadlockを解消する
  - Deadlock Prevention(楽観)
    - wait-die, wound-wait: lockできなければ、デットロックしないように待って or すぐにabortする
      - 複数のcoordinatorがいても動作するメリットあり
    - predicate lock: レコードをロックする代わりに述語ロックする  //楽観なのか???

## Citusのdeadlock検知の方法

- Citusの[アーキテクチャ](http://docs.citusdata.com/en/v9.2/develop/reference_processing.html)

- Citusのdeadlock対策([参考](https://www.citusdata.com/blog/2017/08/31/databases-and-distributed-deadlocks-a-faq/))
  - citusでは、deadlock detection。ロック状況の有効グラフを定期的(デフォルト1秒)に収集して、有向グラフを作成して、検知している
  - ちなみにSpannerは、deadlock prevention
    - Spannerではインタラクティブなtransaction blockのやり取りができない
    - そのため、クエリ実行時に全てのコマンドが分かるので可能な方式
    - // [Spanner](https://cloud.google.com/spanner/docs/transactions)の仕様面白い
      - // 読み込み専用・書込みありのAPIで分けていたり => MVCCでロックのあり/なしに影響?
      - // インタラクティブにやり取りできない => 物理ノードを跨る or 跨がらないもtx開始時に分かる。跨るものだけ分散txに切り替えるなどが可能な仕様のよう

- バックエンド・プロセスが定期的にdeadlockを検知([参考](https://image.slidesharecdn.com/pgconfjapan2018b3citusja20181121-1-181121083621/95/lets-scaleout-postgresql-using-citus-japanese-21-638.jpg?cb=1569912194))
  - 2秒ごとにdump_local_wait_graph関数を実行, 1分ごと   にpg_prepared_xactsビューを検索


- メインの[ソースコード](https://github.com/citusdata/citus/blob/3f8ac527c9380ce792e3d124a52fce393ccf9ad8/src/backend/distributed/transaction/distributed_deadlock_detection.c)


- distributed deadlockに関する[設定](http://docs.citusdata.com/en/v9.5/develop/api_guc.html?highlight=distributed%20deadlock#citus-log-distributed-deadlock-detection-boolean)は2つ
  - citus.log_distributed_deadlock_detection: ログに出力するか
  - citus.distributed_deadlock_detection_factor: deadlockを検知する間隔

## 詳細

※ 動作未確認。ざっと読んだだけなので間違えている可能性高い

### エントリーポイント

- メンテナンス用のプロセスが存在していて、そのプロセスが[CheckForDsitributedDeadlocks](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/utils/maintenanced.c#L565)を呼んでいる
  - 1.各ローカルノードでwait-graph作成
  - 2.グローバルでwait-graph作成
  - 3.検索効率化のため、隣接リストに変換
  - 4.DFSでcycleを検索
  - 5.見つかったら、youngest particiant backendをキャンセル
  - 計算量: O(N). Nは最大backend数   // citusに接続しているクライアント数のよう

### 1. ローカルノードでwait-graph作成

- [BuildLocalWaitGraph()](https://github.com/citusdata/citus/blob/3f8ac527c9380ce792e3d124a52fce393ccf9ad8/src/backend/distributed/transaction/lock_graph.c#L98)が呼ばれる
  - 各backendごとに分散txが発生しているか、ロック待ちしているかなどをチェック
    - グローバルtxかどうかをworker側でも判断できるようになっているようにみえる
      - BackendData.transactionId
  - 発生している場合は、lockを保持している、lock待ちしている状態かを考慮して、wait-graph作成  // なぜ最初からworkerを見にいかないのか分かっていない。coordinatorでもロックするリソースがあるということ?

### 2. グローバルでwait-graphを作成

  [BuildGlobalWaitGraph()](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/lock_graph.c#L91)で実施

- 全workerに対して、"SELECT * FROM dump_local_wait_edges()" を実行
  - citusのextensionで追加されている[関数](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/lock_graph.c#L257)
  - ローカルのwait graphを作成して、返す関数。pg_locksだと付与されていない情報あり
    - // transaction_numがグローバルなtxidと思われる

```
CREATE FUNCTION pg_catalog.dump_local_wait_edges(
                    OUT waiting_pid int4,
                    OUT waiting_node_id int4,
                    OUT waiting_transaction_num int8,
                    OUT waiting_transaction_stamp timestamptz,
                    OUT blocking_pid int4,
                    OUT blocking_node_id int4,
                    OUT blocking_transaction_num int8,
                    OUT blocking_transaction_stamp timestamptz,
                    OUT blocking_transaction_waiting bool)
RETURNS SETOF RECORD
LANGUAGE C STRICT
AS $$MODULE_PATHNAME$$, $$dump_local_wait_edges$$;
COMMENT ON FUNCTION pg_catalog.dump_local_wait_edges()
IS 'returns all local lock wait chains, that start from distributed transactions';
```

- [assign_distributed_transaction_id()](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/backend_data.c#L98)がSQL経由で呼ばれて、分散TXの場合は異なるノードでも同一の分散TX用IDが払い出される仕組みになっている
- 下記のBackendData構造体で各ノードで管理されている
  - extensionをpreloadしたときに[共有メモリ上にBackendのDataを管理する領域が確保](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/backend_data.c)されている

```
typedef struct BackendData
{
	Oid databaseId;
	Oid userId;
	slock_t mutex;
	bool cancelledDueToDeadlock;
	CitusInitiatedBackend citusBackend;
	DistributedTransactionId transactionId;
} BackendData;
```

```
typedef struct DistributedTransactionId
{
	int initiatorNodeIdentifier;
	bool transactionOriginator;
	uint64 transactionNumber;          # wait-graphで返している transaction_num
	TimestampTz timestamp;
} DistributedTransactionId;
```

- ローカルノードで作成したwait-graphに全workerの結果を集約する

### 3. 検索効率化のため、隣接リストに変換

- [BuildAdjacencyListsForWaitGraph()](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/distributed_deadlock_detection.c#L443)を呼ぶ

- トランザクションをキーにして、wait for graphを隣接リストに変換
  - // グローバルなtxの紐付けが行われているっぽい => 共通のtx idを利用

### 4. DFSでcycleを検索

- [CheckDeadlockForTransactionNode()](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/distributed_deadlock_detection.c#L232)を呼ぶ

### 5. 見つかったら、youngest particiant backendをキャンセル

- [CancelTransactionDueToDeadlock()](https://github.com/citusdata/citus/blob/v9.5.1/src/backend/distributed/transaction/distributed_deadlock_detection.c#L208)を呼ぶ


## その他

- [こちら](https://image.slidesharecdn.com/pgconfjapan2018b3citusja20181121-1-181121083621/95/lets-scaleout-postgresql-using-citus-japanese-21-638.jpg?cb=1569912194)に pg_prepared_xacts view が1分おきにアクセスされているとのことなので、ついでに検索してみた
  - PendingWorkerTransactionList()を呼んでいるのは、RecoverWorkerTransactions()で 2PC中にfailureが発生したときしか呼ばれていなさそう
  - // もう少しきちんと調べてみる

- Isolation Levelは、READ COMMITTEDっぽい
