---
layout: post
title: pg_hint_plan extensionに関するメモ
tags:
- postgresql
- extension
- pg_hint_plan
---

# pg_hint_planの概要

[使ってみませんか？pg_hint_plan](https://www.slideshare.net/hadoopxnttdata/pghintplan)が分かりやすい。
モチベーションは、プラン制御。プラン制御の方法は、複数あるが、HINT句を実現するための拡張機能。

- HINT句で制御したい： pg_hint_plan
- 統計情報を固定化したい: pg_dbms_stats

指定できる方法は、

- スキャン方法
- 結合方法
- 結合順序
- 見積もり件数補正
- パラレル実行
- パラメータ設定

クエリごとに指定できるが、予め、テーブルにクエリ文とHINT句の対応方法を保持しておくことが可能

# HINT句の詳細

指定できる[HINT句一覧](http://pghintplan.osdn.jp/hint_list-ja.html)は、

スキャン方法として、指定できそうなものは、

- SeqScan, IndexScan, IndexScanRegexp, BitmapScan, BitmapScanRegexp, TidScan, IndexOnlyScan, IndexOnlyScanRegexp
- NoSeqScan, NoIndeexScan, NoBitmapScan, NoTidScan, NoINdexOnlyScan
  - "No"から始まるスキャン方法は、Noで指定したScan方法以外で最小のコストスキャンを選択

結合方法として、指定できそうなものは、

- NestLoop, MergeJoin, HashJoin
- NoNestLoop, NoMergeJoin, NoHashJoin

結合順序として、指定できそうなものは、

- Leading: テーブルを指定した通りの順番で結合

見積もり件数補正として、指定できそうなものは、

- Rows: 結合結果の見積もり件数を補正

パラレル実行補正として、指定できそうなものは、

- Parallel

パラメータ設定として、指定できそうなもの（[ソースコード](https://github.com/ossc-db/pg_hint_plan/blob/master/pg_hint_plan.c#L96)）は、

- Set: [問い合わせ計画](https://www.postgresql.jp/document/12/html/runtime-config-query.html)に関するGUCパラメータをそのクエリの実行計画を作成している間だけ変更する


# pg_hint_planの実装概要

調査したcommitは、4ffa97e5d239467ee6d0a94d32355e984cfd77a2
hookなどのPostgreSQL本体の挙動を変える箇所は下記。

- post_parse_analyze_hook
- planner_hook
- [join_search_hook](https://wiki.postgresql.org/wiki/Join_Order_Search): joinの順番を指定するために利用
- [set_rel_pathlist_hook](https://www.postgresql.jp/document/12/html/custom-scan-path.html): 本体が生成したパスを修正するために利用する
- ProcessUtility_hook
- "PGpgSQL_plugin"変数へのcallback関数の登録

join周りは未確認。シンプルな条件での挙動や実装を確認。

## デバックログ

SeqScanをIndexScanに変更するシンプルなクエリで動作確認.

```
postgres=# /*+ IndexScan(b) */
        EXPLAIN ANALYSE SELECT * FROM pgbench_branches b WHERE bid = 1;
                                                                 QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using pgbench_branches_pkey on pgbench_branches b  (cost=0.14..8.15 rows=1 width=364) (actual time=0.065..0.075 rows=1 loops=1)
   Index Cond: (bid = 1)
 Planning Time: 1.800 ms
 Execution Time: 0.296 ms
(4 rows)
```

処理の流れとしては、

- 1.ProcessUtility_hook(): EXPLAINを実行しなければ呼ばれない
- 2.post_parse_analyze_hook(): コメント内のHINT句をパース　★1
- 3.planner_hook(): 実行計画を生成 ★2
  - 3.1 standard_plannner内から、set_rel_pathlist_hook()が呼ばれる ★3
    - "enable_seqscan"などのGUC経由でプランを強制する
  - 3.1 standard_plannner内から、join_search_hook()が呼ばれる
  - 利用されたHINT句などの情報を出力 ★4

デバックログ(pg_hint_plan.debug_print = 3)

```
2021-01-16 19:57:38 JST [client backend] LOG:  hints in comment="IndexScan(b) ", query="/*+ IndexScan(b) */    ★1
                EXPLAIN ANALYSE SELECT * FROM pgbench_branches b WHERE bid = 1;", debug_query_string="/*+ IndexScan(b) */
                EXPLAIN ANALYSE SELECT * FROM pgbench_branches b WHERE bid = 1;"
2021-01-16 19:57:38 JST [client backend] LOG:  pg_hint_plan[qno=0x0]: planner ★2
2021-01-16 19:57:38 JST [client backend] STATEMENT:  /*+ IndexScan(b) */
                EXPLAIN ANALYSE SELECT * FROM pgbench_branches b WHERE bid = 1;
2021-01-16 19:57:38 JST [client backend] LOG:  pg_hint_plan[qno=0x0]: setup_hint_enforcement index deletion: relation=16407(pgbench_branches), inhparent=0, current_hint_state=0x564b009c1ee0, hint_inhibit_level=0, scanmask=0x2　★3
2021-01-16 19:57:38 JST [client backend] LOG:  pg_hint_plan[qno=0x0]: HintStateDump: {used hints:IndexScan(b)}, {not used hints:(none)}, {duplicate hints:(none)}, {error hints:(none)} ★4
```

## post_parse_analyze_hook

pg_hint_plan_post_parse_analyze()がエントリーポイントで、処理内容は、

- parse後のParseState/Queryを元に、"current_hint_str"にHINT文を登録しておく（★1）
  - get_hints_from_comment()で、コメント文からHINT文を特定しておく。あくまで英語・数字などのレベル
- hint tableを有効化している場合は、クエリ文とhint table内のクエリ文を突合する
  - get_hints_from_table()で、hint table自身に問い合わせる

## planner_hook

pg_hint_plan_planner()がエントリーポイントで、処理内容は、

- "current_hint_str"を読み取って、HintStateを"hstate"に登録
  - create_hintstate()で、parse treeとpaser hintを元にhstate("current_hint_state")を生成
    - parse_hints()で、"current_hint_str"をparseして、HINT句の意味を持たせる
    - Scan方法(HintState->scan_hint)、JOIN方法(HintState->join_hint->enforce_mask)、順序の指定など

- setup_guc_enforcement()でGCUのupdateを実施

- standard_planner()を呼ぶ
  - 内部でset_rel_pathlist_hook()が呼ばれる
  - その後、利用されたHINT句などのデバック情報を出力（★4）

## set_rel_pathlist_hook

pg_hint_plan_set_rel_pathlist()がエントリーポイント。
ここがGUCなどを経由して、実行計画を指定しており、pg_hint_planのメインの処理。

- HINT句を元にGUCを書き換える（★3）
  - setup_hint_enforcement()で、GUCや実行計画で選択可能なindexを制御する
    - find_scan_hint()で、対象relationに関して、ヒント句内("hstate")にScan方法の指定が存在するかをチェック
      - 同一relationに対して、重複して指定されていたら、最後の1つのみ採用
    - setup_scan_method_enforcement()で、GUCの"enalbe_seqscan"などを書き換える
      - HINT句で指定されたScan方法を特定して、"mask"に格納
      - maskで指定されたscan方法のみtrueにして、他のscan方法はfalseにする
      - // 直接GUCを利用する場合は、クエリ内の表ごとの変更は出来なさそう(?)なので、個々のrelationごとに指定できるpg_hint_planが必要という理解
    - restrict_indexes()で、指定されていないindexの使用を防ぐ
      - 複数インデックスが存在しているときに利用させるインデックスを制御するため
      - SeqScan, TidScanやindex指定がない場合は、利用できるインデックス(rel->indexlist)を削除する
      - HINT句で利用可能なindexが指定されていない場合は、何もしない（削除しない）
      - テーブル内のindexから、HINT句で指定されていないindexは、list_delete_cell()でrel->indexlistから削除する
    - find_parallel_hint(), setup_parallel_plan_enforcement()で、parallel度を指定
      - Scanのときと同様にParallelHint(hint)経由で、GUCの"max_parallel_workers_per_gather"などを指定

- 本体が生成した、実行計画を書き換える
  - set_plain_rel_pathlist()で、rel->pathlistを書き換える
    - 例えば、(Path *)(rel->pathlist->elements[0])->pathtypeを"T_SeqScan"から"T_indexPath"に書き換える
      - 本体と同じコード(core.c)内のcreate_index_paths()でindexPathへの置き換えは実施されている
  - parallel度については、hardで指定している場合、直接relを書き換える


## join_search_hook

pg_hint_plan_join_search()がエントリーポイント。
joinのjoin方法、パラレル度、結合順序を指定する。

- JOIN方法を指定
  - transform_join_hints()でHintStateから、指定されたjoin方法、見積もり件数に変更。順序指定の有無を確認
  - pg_hint_plan_standard_join_search()でGUICの"enable_nestloop"などを変更  // 詳細未調査
   - ->pg_hint_plan_standard_join_search()->pg_hint_plan_join_saerch_one_level()
   - ->make_rels_by_clause_joins()->make_join_rel_wrapper()->set_join_config_options()
  - pg_hint_plan_make_join_rel()で、joinのRelOptInfoを作成

- JOINのパラレル度を指定
  - (Path *)(rel->partial_pathlist[x])->parallel_workersを直接書き換える
  - set_join_config_options()でGUCの"hash_mem_multiplier"を変更



# 気をつけるべき点

[ドキュメント](https://pghintplan.osdn.jp/pg_hint_plan-ja.html)に記載されている点で気になる点を下記に記載。これ以外にもあるとは思う

- スキャン方式を指定できるのは、通常のテーブルや継承テーブルなど。外部テーブル・ビュー・副問合せ結果などは無理
- HINT句のコメントを記載する位置。最初のブロックコメントにのみヒントを記載するのが安全っぽい
- PL/pgSQL内でも指定できるが、指定できないSQL文もある
- ヒント句の構文誤りがあっても、途中までのヒント句は有効として、クエリは実行されてしまう

# その他

## 動作するバージョン

2021/1/16時点で13.0は、サポート済。
masterブランチだとエラーになる

```
> make                                                                        master [4ffa97e]
In file included from pg_hint_plan.c:4951:
core.c: In function ‘set_append_rel_pathlist’:
core.c:139:7: error: ‘RelOptInfo’ {aka ‘struct RelOptInfo’} has no member named ‘partitioned_child_rels’
  139 |    rel->partitioned_child_rels =
      |       ^~
core.c:140:20: error: ‘RelOptInfo’ {aka ‘struct RelOptInfo’} has no member named ‘partitioned_child_rels’
  140 |     list_concat(rel->partitioned_child_rels,
      |                    ^~
core.c:141:16: error: ‘RelOptInfo’ {aka ‘struct RelOptInfo’} has no member named ‘partitioned_child_rels’
  141 |        childrel->partitioned_child_rels);
      |                ^~
make: *** [<ビルトイン>: pg_hint_plan.o] エラー 1
```

## HINT句が本体に取り込まれていない理由

[PostgreSQL wiki](https://wiki.postgresql.org/wiki/OptimizerHintsDiscussion)によると、

- ヒント句をサポートすると、大量のリファクタリングが必要。メンテナンス性が落ちる
- upgrade後の性能劣化の可能性が考えられる
- DBAが真の問題に目を向けず、HINT句によるワークアラウンド対応に頼ってしまう
- データサイズにスケールしない
- 大部分の場合、optimizerが正しい
- ユーザがHINT句に頼り始めると、communityにplanner改善に関するレポートをしてくれなくなる

## Tid Scanが発生するクエリ文

ctidを指定した場合に選択される場合があるscan方法。
ctidは、[システム列](https://www.postgresql.jp/document/12/html/ddl-system-columns.html)の1項目でタプルの物理位置を識別するための組（ブロック番号、ブロック内のタプルインデックス）のこと。

```
postgres=# EXPLAIN SELECT aid,bid,abalance,ctid FROM pgbench_accounts WHERE ctid = '(0,1)';
                           QUERY PLAN
-----------------------------------------------------------------
 Tid Scan on pgbench_accounts  (cost=0.00..4.01 rows=1 width=18)
   TID Cond: (ctid = '(0,1)'::tid)
(2 rows)
```

## IndexScanなどの場合、利用するIndexの指定も可能

```
postgres=# \d pgbench_branches;
              Table "public.pgbench_branches"
  Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
 bid      | integer       |           | not null |
 bbalance | integer       |           |          |
 filler   | character(88) |           |          |
Indexes:
    "pgbench_branches_pkey" PRIMARY KEY, btree (bid)
    "pgbench_branches_bbalance" btree (bbalance)
    "pgbench_branches_bbalance_bid" btree (bbalance, bid)
```

*_bbalance というindexを指定

```
postgres=# /*+ IndexScan(b pgbench_branches_bbalance) */
        EXPLAIN ANALYZE SELECT * FROM pgbench_branches b WHERE bbalance = 0;
                                                                    QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using pgbench_branches_bbalance on pgbench_branches b  (cost=0.14..12.31 rows=10 width=364) (actual time=0.046..0.061 rows=10 loops=1)
   Index Cond: (bbalance = 0)
 Planning Time: 0.514 ms
 Execution Time: 0.134 ms
(4 rows)
```

*_bbalance_bid というindexを指定

```
postgres=# /*+ IndexScan(b pgbench_branches_bbalance_bid) */
        EXPLAIN ANALYZE SELECT * FROM pgbench_branches b WHERE bbalance = 0;
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using pgbench_branches_bbalance_bid on pgbench_branches b  (cost=0.14..12.31 rows=10 width=364) (actual time=0.051..0.068 rows=10 loops=1)
   Index Cond: (bbalance = 0)
 Planning Time: 0.597 ms
 Execution Time: 0.162 ms
(4 rows)
```

debugログには、下記のような出力がされる

```
2021-01-16 21:30:23 JST [client backend] LOG:  available indexes for IndexScan(b): pgbench_branches_bbalance
2021-01-16 21:30:23 JST [client backend] STATEMENT:  /*+ IndexScan(b pgbench_branches_bbalance) */
                EXPLAIN ANALYZE SELECT * FROM pgbench_branches b WHERE bbalance = 0;
2021-01-16 21:30:23 JST [client backend] LOG:  pg_hint_plan[qno=0x0]: setup_hint_enforcement index deletion: relation=16407(pgbench_branches), inhparent=0, current_hint_state=0x5653fd2c2ee0, hint_inhibit_level=0, scanmask=0x2
```

## パラレルクエリのワーカ数も指定可能

hard optionを付与しないと、指定した値にならないこともある

```
postgres=#  /*+ Parallel(pgbench_accounts 4) */
      EXPLAIN ANALYZE  SELECT * FROM pgbench_accounts WHERE filler LIKE '%x%';
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..1000.10 rows=1 width=97) (actual time=1324.630..1324.728 rows=0 loops=1)
   Workers Planned: 3
   Workers Launched: 3
   ->  Parallel Seq Scan on pgbench_accounts  (cost=0.00..0.00 rows=1 width=97) (actual time=494.107..494.108 rows=0 loops=4)
         Filter: (filler ~~ '%x%'::text)
         Rows Removed by Filter: 250000
 Planning Time: 38552.069 ms
 Execution Time: 1324.832 ms
(8 rows)
```

filteringや集約関数を呼ばない場合は、parallel度を変更することは出来なかった
詳細は未調査だけど、Diskアクセスだけだから、parallel化するメリットがないから?

```
postgres=# /*+ Parallel(pgbench_accounts 8 hard) */
postgres-#       EXPLAIN ANALYZE  SELECT * FROM pgbench_accounts;
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Seq Scan on pgbench_accounts  (cost=0.00..26394.00 rows=1000000 width=97) (actual time=0.038..121.540 rows=1000000 loops=1)
 Planning Time: 2908.621 ms
 Execution Time: 156.067 ms
(3 rows)
```

## Regexpって何?

pg_store_plans特有の[機能](https://pghintplan.osdn.jp/hint_list-ja.html)。
利用可能なindexをregex経由で指定する方法。

```
postgres=# /*+ IndexScanRegexp(b pgbench_branches_bbalance_*) */
        EXPLAIN ANALYZE SELECT * FROM pgbench_branches b WHERE bbalance = 0;
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using pgbench_branches_bbalance_bid on pgbench_branches b  (cost=0.14..12.31 rows=10 width=364) (actual time=0.031..0.040 rows=10 loops=1)
   Index Cond: (bbalance = 0)
 Planning Time: 0.586 ms
 Execution Time: 0.084 ms
(4 rows)
```


## 検索不能なHINT句が指定された場合は？

[ドキュメント](https://pghintplan.osdn.jp/pg_hint_plan-ja.html)の機能制限に記載されている通り。
例えば、ctid指定でないクエリなのに、TidScanをHINT句で指定した場合


# まとめ

pg_hint_planでは、クエリやrelationごとにGUCのパラメータを変更することで、HINT句で指定されたScan/Merge方法などを変更しているということは分かった。

ただ、実行計画作成に関する基礎知識が足りていないので、そこから調べないと...

- nodeとrelationの違いは?
- joinもRelOptInfoで管理されているということは、joinのrelation

など
