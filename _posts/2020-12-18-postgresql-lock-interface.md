---
layout: post
title: PostgreSQLのheavy weight lock
tags:
- postgresql
- lock
---

## 同時実行制御を実現するためのロック

PostgreSQLは、MVCCのアーキテクチャを採用している。
でも、ロックしないと同時実行制御が満たせない場合がある

- 例えば
  - TRUNCATEコマンドでテーブル全体の操作を実施したい場合など
  - write-writeのconflictを防ぐため、SELECT * FOR UPDATEなどでロックを取得する

## ロックマネージャ

- 通常のロックマネージャー
  - 通常ロック
  - 近道ロック(fastpath=true) -> 各バックエンドからひとつひとつ収集
    - // 本当にロックと言えるの? その時点でロックが競合することがないことを分かっている場合にのみ使える -> それなのに必要なの?
- 述部ロックマネージャー   //SERIALIZE用? https://qiita.com/yuba/items/89496dda291edb2e558c

## Lock/LWLock/Latch


## ロックのモード

Oxford Languagesの定義によると、ロック(lock)は「かぎをかけて、戸やとびらが開かないようにすること。」とのこと。ただ、同時にアクセスできたりする


## ロック状況の確認方法

システムカタログの[pg_locks](https://www.postgresql.org/docs/13/view-pg-locks.html) viewを使って、データベースクラスタ内のアクティブなロックの状況を確認することができる。

ロックが取得されているロック対象オブジェクトごとに1行づつ、関連情報が出力される。
PostgreSQL v13では、全部で15項目の情報が出力される。

各項目の詳細は以下の通り。

### locktype

ロック対象オブジェクトの種類のこと。種類は全11種類。
pg_stat_activityの[Lockの待機イベント](https://www.postgresql.org/docs/13/monitoring-stats.html#WAIT-EVENT-LOCK-TABLE)でも出力されるもの。

- relation: リレーション上のロック //TODO
- extend: リレーション拡張時のロック //TODO
- frozenid: pg_database.datfrozenxid and pg_database.datminmxidの更新用のロック //TODO
- page: リレーション上のページのロック
- tuple: タプル上のロック
- transactionid: トランザクションが終了するのを待つロック  //TODO: sawada-san blog
- virtualxid: 仮想xidロック //TODO
- spectoken: 投機的挿入ロック  //TODO
- object: 非リレーショナルデータベースオブジェクト上のロック
- userlock: ユーザロック
- advisory: 勧告的ロック  //TODO: 助言ユーザロック?

#### advisory

アプリケーション側で意味付けしたロック。
高速で肥大化を防げるらしい  //TODO: why? -> テーブルなどではなくて、単純に共有メモリにキーをおいておくだけだから?
https://www.postgresql.org/docs/devel/explicit-locking.html#ADVISORY-LOCKS

セッションレベルか、トランザクションレベルで取得可能

トランザクションをまたいだ排他制御なども可能

### database

ロック対象が存在するdatabsaeのOID。
対象が共有オブジェクトであれば0, transaction idであればnull //TODO

### relation

ロック対象のリレーションのOID。
ターゲットがリレーション出ない場合やリレーションの一部であればnull //TODO: 一部って何?

### page

ロック対象のリレーションのページ番号。
リレーションのページやタプルでない場合はnull

### tuple

ロック対象のタプル番号。タプルでない場合はnull

ロック情報は、共有メモリ上ではなく、ディスク上に格納している。
したがって、共有メモリ上のデータを参照しているpg_locks viewには出力されない。
その行ロックを保持している永続トランザクションIDを待つ状態として、ビューに現れる

### virtualxid

ロック対象のトランザクションの仮想ID  // TODO: 仮想IDって何?
仮想IDでない場合はnull。

トランザクションの実行中は常に、仮想トランザクションIDに排他ロックをかける //TODO:誰が?
トランザクションがデータベースの状態を変化させるときに永続IDが割り当てられる場合もある //TODO: いつ?

トランザクションは、終了するまで永続トランザクションIDに対して、トランザクション流量時まで排他ロックを保持する

### transactionid

ロック対象のトランザクションID。
対象がトランザクションIDでない場合はnull。   //TODO: targetって、locktypeのこと?

### classid

ロック対象を含むシステムカタログのOID。
対象が一般的なデータベースでない場合はnull。  //TODO: 一般的なデータベースでないって何?

### objid

システムカタログ内のロック対象のOID。
対象が一般的なデータベースでない場合はnull。  //TODO: 一般的なデータベースでないって何?

### objsubid

ロック対象の列番号。
その他の一般的なデータベースオブジェクトの場合は0。
対象が一般的なデータベースでない場合はnull。

### virtualtransaction

このロックを保持している、もしくは、待機しているトランザクションのvirutal ID

### pid

このロックを保持している、もしくは、待機しているpid。
Prepared Transactionの場合はnull  // TODO: どういうこと?

### mode

ロックモード

### granted

ロックを保持していたらtrue。待機していたらfalse。
falseのプロセスが1つでもいる場合は、何かしらのオブジェクトのロック取得で
競合が起きているということ。

### fastpath

fast path経由でロックを取得していたらtrue。  // TODO: そもそも"fast path経由"とは
メインのロックテーブル経由で取得していたらfalse。


## １つのクエリでも複数対象のロックを取得する

## ロックに関連する有用そうなパラメータ

- https://www.postgresql.org/docs/devel/runtime-config-locks.html


## その他

- 各プロセスは、一度に多くても、1つのロックしか待たない
- pg_locksは、database cluster全体の情報を出力しているので、relation名などとのjoinが必要な場合は、そのrelationの情報を保持しているデータベースに接続して、pg_classとjoinする必要がある
- ロック待ちキューのロック獲得の前後関係を確認するためには、pg_blocking_pids()
- テーブルレベルのロックでも "ROW EXCLUSIVE" modeなど誤解を招く表現があるので注意

## 今後調べてみたいこと

- TODO: ロック対象のオブジェクトが示しているもの
  - Lock TypeのWait Events

- もろもろTODO

- lock implementation
