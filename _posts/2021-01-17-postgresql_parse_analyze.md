---
layout: post
title: postgresqlのparse/analyze/rewrite/planに関する処理メモ
tags:
- postgresql
- parse
- analyze
---

# postgresqlのクエリ処理の流れ

parse -> analyze -> rewrite -> plan -> executor という流れで処理される。
[公式ドキュメント](https://www.postgresql.org/docs/current/rules.html)や[こちら](https://www.interdb.jp/pg/pgsql03.html)や[こちら](http://www.nminoru.jp/~nminoru/postgresql/pg-plan-tree.html)を参考にしつつ、planまでの処理をざっくり調べてみる

# 調査方法

ざっくり把握のため、SQL文を固定して、デバックログで各種変数などを追いながら処理を確認する。
便利なdebugオプションが存在しているのでそちらを指定して、下記のSQLを実行したときの出力を各処理の節に記載する。

```
# vim postgresql.conf
debug_print_parse = on
debug_print_rewritten = on
debug_print_plan = on
```

```
postgres=# SELECT * FROM pgbench_branches b
          JOIN pgbench_accounts a ON b.bid = a.bid ORDER BY a.aid LIMIT 10;
```

# parse/analyze処理の流れ

デバックログに出力されたparse/analyze結果は下記の通り。
parsenode.hで定義されているQuery構造体を出力している。

```
2021-01-17 19:40:52 JST [client backend] LOG:  parse tree:
2021-01-17 19:40:52 JST [client backend] DETAIL:     {QUERY
           :commandType 1
           :querySource 0
           :canSetTag true
           :utilityStmt <>
           :resultRelation 0
           :hasAggs false
           :hasWindowFuncs false
           :hasTargetSRFs false
           :hasSubLinks false
           :hasDistinctOn false
           :hasRecursive false
           :hasModifyingCTE false
           :hasForUpdate false
           :hasRowSecurity false
           :cteList <>
           :rtable (
              {RTE
              :alias
                 {ALIAS
                 :aliasname b
                 :colnames <>
                 }
              :eref
                 {ALIAS
                 :aliasname b
                 :colnames ("bid" "bbalance" "filler")
                 }
              :rtekind 0
              :relid 16441
              :relkind r
              :rellockmode 1
              :tablesample <>
              :lateral false
              :inh true
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b 8)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
              {RTE
              :alias
                 {ALIAS
                 :aliasname a
                 :colnames <>
                 }
              :eref
                 {ALIAS
                 :aliasname a
                 :colnames ("aid" "bid" "abalance" "filler")
                 }
              :rtekind 0
              :relid 16438
              :relkind r
              :rellockmode 1
              :tablesample <>
              :lateral false
              :inh true
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b 8 9)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
              {RTE
              :alias <>
              :eref
                 {ALIAS
                 :aliasname unnamed_join
                 :colnames ("bid" "bbalance" "filler" "aid" "bid" "abalance" "filler")
                 }
              :rtekind 2
              :jointype 0
              :joinmergedcols 0
              :joinaliasvars (
                 {VAR
                 :varno 1
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 1
                 :location -1
                 }
                 {VAR
                 :varno 1
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 2
                 :location -1
                 }
                 {VAR
                 :varno 1
                 :varattno 3
                 :vartype 1042
                 :vartypmod 92
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 3
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 1
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 2
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 3
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 3
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 4
                 :vartype 1042
                 :vartypmod 88
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 4
                 :location -1
                 }
              )
              :joinleftcols (i 1 2 3)
              :joinrightcols (i 1 2 3 4)
              :lateral false
              :inh false
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
           )
           :jointree
              {FROMEXPR
              :fromlist (
                 {JOINEXPR
                 :jointype 0
                 :isNatural false
                 :larg
                    {RANGETBLREF
                    :rtindex 1
                    }
                 :rarg
                    {RANGETBLREF
                    :rtindex 2
                    }
                 :usingClause <>
                 :quals
                    {OPEXPR
                    :opno 96
                    :opfuncid 65
                    :opresulttype 16
                    :opretset false
                    :opcollid 0
                    :inputcollid 0
                    :args (
                       {VAR
                       :varno 1
                       :varattno 1
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 1
                       :varattnosyn 1
                       :location 70
                       }
                       {VAR
                       :varno 2
                       :varattno 2
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 2
                       :location 78
                       }
                    )
                    :location 76
                    }
                 :alias <>
                 :rtindex 3
                 }
              )
              :quals <>
              }
           :targetList (
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 1
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 1
                 :location 7
                 }
              :resno 1
              :resname bid
              :ressortgroupref 0
              :resorigtbl 16441
              :resorigcol 1
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 1
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 2
                 :location 7
                 }
              :resno 2
              :resname bbalance
              :ressortgroupref 0
              :resorigtbl 16441
              :resorigcol 2
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 1
                 :varattno 3
                 :vartype 1042
                 :vartypmod 92
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 3
                 :location 7
                 }
              :resno 3
              :resname filler
              :ressortgroupref 0
              :resorigtbl 16441
              :resorigcol 3
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 1
                 :location 7
                 }
              :resno 4
              :resname aid
              :ressortgroupref 1
              :resorigtbl 16438
              :resorigcol 1
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 2
                 :location 7
                 }
              :resno 5
              :resname bid
              :ressortgroupref 0
              :resorigtbl 16438
              :resorigcol 2
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 3
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 3
                 :location 7
                 }
              :resno 6
              :resname abalance
              :ressortgroupref 0
              :resorigtbl 16438
              :resorigcol 3
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 4
                 :vartype 1042
                 :vartypmod 88
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 4
                 :location 7
                 }
              :resno 7
              :resname filler
              :ressortgroupref 0
              :resorigtbl 16438
              :resorigcol 4
              :resjunk false
              }
           )
           :override 0
           :onConflict <>
           :returningList <>
           :groupClause <>
           :groupingSets <>
           :havingQual <>
           :windowClause <>
           :distinctClause <>
           :sortClause (
              {SORTGROUPCLAUSE
              :tleSortGroupRef 1
              :eqop 96
              :sortop 97
              :nulls_first false
              :hashable true
              }
           )
           :limitOffset <>
           :limitCount
              {FUNCEXPR
              :funcid 481
              :funcresulttype 20
              :funcretset false
              :funcvariadic false
              :funcformat 2
              :funccollid 0
              :inputcollid 0
              :args (
                 {CONST
                 :consttype 23
                 :consttypmod -1
                 :constcollid 0
                 :constlen 4
                 :constbyval true
                 :constisnull false
                 :location 105
                 :constvalue 4 [ 10 0 0 0 0 0 0 0 ]
                 }
              )
              :location -1
              }
           :limitOption 0
           :rowMarks <>
           :setOperations <>
           :constraintDeps <>
           :withCheckOptions <>
           :stmt_location 0
           :stmt_len 107
           }
```

コールスタックは、下記の通り。

```
PostgresMain()
  -> exec_simple_query
    -> pg_analyze_and_rewrite()
      -> parse_analyze()
```

parse_analyze()内でQuery Tree(Query構造体)を作成している。
上記のデバックログの通り、様々な情報を保持している。一例では、

- commandType: select/insertなど。1はSELECT
- hasXXX: window関数やsubクエリを保持しているか
- rtable: このクエリが操作するtable一覧。JOIN, subquery, 関数, valueなども含む
  - parsenodes.h内のRTEKindで定義
- jointree: join操作の一覧。join type(INNER/OUTER), left/right subtree, 自分自身やsubtreeのrtableの番号との対応
  - primitive nodeを定義しているprimnodes.h内のJoinExprで定義
- targetList: 出力するタプルのカラム番号, tableのoidなどの一覧。"*"などと記載しても各カラム名に展開される
  - primnodes.h内のTargetEntryで定義
- sortClause: ORDER BYで利用するoperatorのoidなど。
  - parsenodes.h内のSortGroupClauseで定義
- limitCount: 返却するtuple数

## parse処理

### parse treeの生成

SQL文をplain textとして、パースして、parse treeを生成する。

parse treeを管理する構造体は、NodeTagによって異なり、InsertStmt/SelectStmtなどがある。
今回実行しているクエリは、SELECTなのでSelectStmt構造体でparse treeを管理する。

★ SelectStmt構造体は?


### parse中の状態管理の方法

parse/analyze中の状態管理は、ParseState構造体を利用して実施する

★ ParseState構造体は?


### 詳細


## analyze処理の流れ

parse treeを元に意味付けを行い、query treeを生成する。


# rewrite処理の流れ

pg_rewrite catalog


# plan処理の流れ



2021-01-17 19:40:52 JST [client backend] STATEMENT:  SELECT * FROM pgbench_branches b
                  JOIN pgbench_accounts a ON b.bid = a.bid ORDER BY a.aid LIMIT 10;
2021-01-17 19:40:52 JST [client backend] LOG:  rewritten parse tree:
2021-01-17 19:40:52 JST [client backend] DETAIL:  (
           {QUERY
           :commandType 1
           :querySource 0
           :canSetTag true
           :utilityStmt <>
           :resultRelation 0
           :hasAggs false
           :hasWindowFuncs false
           :hasTargetSRFs false
           :hasSubLinks false
           :hasDistinctOn false
           :hasRecursive false
           :hasModifyingCTE false
           :hasForUpdate false
           :hasRowSecurity false
           :cteList <>
           :rtable (
              {RTE
              :alias
                 {ALIAS
                 :aliasname b
                 :colnames <>
                 }
              :eref
                 {ALIAS
                 :aliasname b
                 :colnames ("bid" "bbalance" "filler")
                 }
              :rtekind 0
              :relid 16441
              :relkind r
              :rellockmode 1
              :tablesample <>
              :lateral false
              :inh true
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b 8)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
              {RTE
              :alias
                 {ALIAS
                 :aliasname a
                 :colnames <>
                 }
              :eref
                 {ALIAS
                 :aliasname a
                 :colnames ("aid" "bid" "abalance" "filler")
                 }
              :rtekind 0
              :relid 16438
              :relkind r
              :rellockmode 1
              :tablesample <>
              :lateral false
              :inh true
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b 8 9)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
              {RTE
              :alias <>
              :eref
                 {ALIAS
                 :aliasname unnamed_join
                 :colnames ("bid" "bbalance" "filler" "aid" "bid" "abalance" "filler")
                 }
              :rtekind 2
              :jointype 0
              :joinmergedcols 0
              :joinaliasvars (
                 {VAR
                 :varno 1
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 1
                 :location -1
                 }
                 {VAR
                 :varno 1
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 2
                 :location -1
                 }
                 {VAR
                 :varno 1
                 :varattno 3
                 :vartype 1042
                 :vartypmod 92
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 3
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 1
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 2
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 3
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 3
                 :location -1
                 }
                 {VAR
                 :varno 2
                 :varattno 4
                 :vartype 1042
                 :vartypmod 88
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 4
                 :location -1
                 }
              )
              :joinleftcols (i 1 2 3)
              :joinrightcols (i 1 2 3 4)
              :lateral false
              :inh false
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
           )
           :jointree
              {FROMEXPR
              :fromlist (
                 {JOINEXPR
                 :jointype 0
                 :isNatural false
                 :larg
                    {RANGETBLREF
                    :rtindex 1
                    }
                 :rarg
                    {RANGETBLREF
                    :rtindex 2
                    }
                 :usingClause <>
                 :quals
                    {OPEXPR
                    :opno 96
                    :opfuncid 65
                    :opresulttype 16
                    :opretset false
                    :opcollid 0
                    :inputcollid 0
                    :args (
                       {VAR
                       :varno 1
                       :varattno 1
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 1
                       :varattnosyn 1
                       :location 70
                       }
                       {VAR
                       :varno 2
                       :varattno 2
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 2
                       :location 78
                       }
                    )
                    :location 76
                    }
                 :alias <>
                 :rtindex 3
                 }
              )
              :quals <>
              }
           :targetList (
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 1
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 1
                 :location 7
                 }
              :resno 1
              :resname bid
              :ressortgroupref 0
              :resorigtbl 16441
              :resorigcol 1
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 1
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 2
                 :location 7
                 }
              :resno 2
              :resname bbalance
              :ressortgroupref 0
              :resorigtbl 16441
              :resorigcol 2
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 1
                 :varattno 3
                 :vartype 1042
                 :vartypmod 92
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 1
                 :varattnosyn 3
                 :location 7
                 }
              :resno 3
              :resname filler
              :ressortgroupref 0
              :resorigtbl 16441
              :resorigcol 3
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 1
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 1
                 :location 7
                 }
              :resno 4
              :resname aid
              :ressortgroupref 1
              :resorigtbl 16438
              :resorigcol 1
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 2
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 2
                 :location 7
                 }
              :resno 5
              :resname bid
              :ressortgroupref 0
              :resorigtbl 16438
              :resorigcol 2
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 3
                 :vartype 23
                 :vartypmod -1
                 :varcollid 0
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 3
                 :location 7
                 }
              :resno 6
              :resname abalance
              :ressortgroupref 0
              :resorigtbl 16438
              :resorigcol 3
              :resjunk false
              }
              {TARGETENTRY
              :expr
                 {VAR
                 :varno 2
                 :varattno 4
                 :vartype 1042
                 :vartypmod 88
                 :varcollid 100
                 :varlevelsup 0
                 :varnosyn 2
                 :varattnosyn 4
                 :location 7
                 }
              :resno 7
              :resname filler
              :ressortgroupref 0
              :resorigtbl 16438
              :resorigcol 4
              :resjunk false
              }
           )
           :override 0
           :onConflict <>
           :returningList <>
           :groupClause <>
           :groupingSets <>
           :havingQual <>
           :windowClause <>
           :distinctClause <>
           :sortClause (
              {SORTGROUPCLAUSE
              :tleSortGroupRef 1
              :eqop 96
              :sortop 97
              :nulls_first false
              :hashable true
              }
           )
           :limitOffset <>
           :limitCount
              {FUNCEXPR
              :funcid 481
              :funcresulttype 20
              :funcretset false
              :funcvariadic false
              :funcformat 2
              :funccollid 0
              :inputcollid 0
              :args (
                 {CONST
                 :consttype 23
                 :consttypmod -1
                 :constcollid 0
                 :constlen 4
                 :constbyval true
                 :constisnull false
                 :location 105
                 :constvalue 4 [ 10 0 0 0 0 0 0 0 ]
                 }
              )
              :location -1
              }
           :limitOption 0
           :rowMarks <>
           :setOperations <>
           :constraintDeps <>
           :withCheckOptions <>
           :stmt_location 0
           :stmt_len 107
           }
        )

2021-01-17 19:40:52 JST [client backend] STATEMENT:  SELECT * FROM pgbench_branches b
                  JOIN pgbench_accounts a ON b.bid = a.bid ORDER BY a.aid LIMIT 10;
2021-01-17 19:40:52 JST [client backend] LOG:  plan:
2021-01-17 19:40:52 JST [client backend] DETAIL:     {PLANNEDSTMT
           :commandType 1
           :queryId 9222993614704608748
           :hasReturning false
           :hasModifyingCTE false
           :canSetTag true
           :transientPlan false
           :dependsOnRole false
           :parallelModeNeeded false
           :jitFlags 0
           :planTree
              {LIMIT
              :startup_cost 0.42
              :total_cost 2.23
              :plan_rows 10
              :plan_width 461
              :parallel_aware false
              :parallel_safe true
              :plan_node_id 0
              :targetlist (
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 1
                    :vartype 23
                    :vartypmod -1
                    :varcollid 0
                    :varlevelsup 0
                    :varnosyn 1
                    :varattnosyn 1
                    :location -1
                    }
                 :resno 1
                 :resname bid
                 :ressortgroupref 0
                 :resorigtbl 16441
                 :resorigcol 1
                 :resjunk false
                 }
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 2
                    :vartype 23
                    :vartypmod -1
                    :varcollid 0
                    :varlevelsup 0
                    :varnosyn 1
                    :varattnosyn 2
                    :location -1
                    }
                 :resno 2
                 :resname bbalance
                 :ressortgroupref 0
                 :resorigtbl 16441
                 :resorigcol 2
                 :resjunk false
                 }
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 3
                    :vartype 1042
                    :vartypmod 92
                    :varcollid 100
                    :varlevelsup 0
                    :varnosyn 1
                    :varattnosyn 3
                    :location -1
                    }
                 :resno 3
                 :resname filler
                 :ressortgroupref 0
                 :resorigtbl 16441
                 :resorigcol 3
                 :resjunk false
                 }
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 4
                    :vartype 23
                    :vartypmod -1
                    :varcollid 0
                    :varlevelsup 0
                    :varnosyn 2
                    :varattnosyn 1
                    :location -1
                    }
                 :resno 4
                 :resname aid
                 :ressortgroupref 1
                 :resorigtbl 16438
                 :resorigcol 1
                 :resjunk false
                 }
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 5
                    :vartype 23
                    :vartypmod -1
                    :varcollid 0
                    :varlevelsup 0
                    :varnosyn 2
                    :varattnosyn 2
                    :location -1
                    }
                 :resno 5
                 :resname bid
                 :ressortgroupref 0
                 :resorigtbl 16438
                 :resorigcol 2
                 :resjunk false
                 }
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 6
                    :vartype 23
                    :vartypmod -1
                    :varcollid 0
                    :varlevelsup 0
                    :varnosyn 2
                    :varattnosyn 3
                    :location -1
                    }
                 :resno 6
                 :resname abalance
                 :ressortgroupref 0
                 :resorigtbl 16438
                 :resorigcol 3
                 :resjunk false
                 }
                 {TARGETENTRY
                 :expr
                    {VAR
                    :varno 65001
                    :varattno 7
                    :vartype 1042
                    :vartypmod 88
                    :varcollid 100
                    :varlevelsup 0
                    :varnosyn 2
                    :varattnosyn 4
                    :location -1
                    }
                 :resno 7
                 :resname filler
                 :ressortgroupref 0
                 :resorigtbl 16438
                 :resorigcol 4
                 :resjunk false
                 }
              )
              :qual <>
              :lefttree
                 {NESTLOOP
                 :startup_cost 0.42
                 :total_cost 180105.82
                 :plan_rows 1000000
                 :plan_width 461
                 :parallel_aware false
                 :parallel_safe true
                 :plan_node_id 1
                 :targetlist (
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65000
                       :varattno 1
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 1
                       :varattnosyn 1
                       :location 7
                       }
                    :resno 1
                    :resname bid
                    :ressortgroupref 0
                    :resorigtbl 16441
                    :resorigcol 1
                    :resjunk false
                    }
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65000
                       :varattno 2
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 1
                       :varattnosyn 2
                       :location 7
                       }
                    :resno 2
                    :resname bbalance
                    :ressortgroupref 0
                    :resorigtbl 16441
                    :resorigcol 2
                    :resjunk false
                    }
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65000
                       :varattno 3
                       :vartype 1042
                       :vartypmod 92
                       :varcollid 100
                       :varlevelsup 0
                       :varnosyn 1
                       :varattnosyn 3
                       :location 7
                       }
                    :resno 3
                    :resname filler
                    :ressortgroupref 0
                    :resorigtbl 16441
                    :resorigcol 3
                    :resjunk false
                    }
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65001
                       :varattno 1
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 1
                       :location 7
                       }
                    :resno 4
                    :resname aid
                    :ressortgroupref 1
                    :resorigtbl 16438
                    :resorigcol 1
                    :resjunk false
                    }
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65001
                       :varattno 2
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 2
                       :location 7
                       }
                    :resno 5
                    :resname bid
                    :ressortgroupref 0
                    :resorigtbl 16438
                    :resorigcol 2
                    :resjunk false
                    }
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65001
                       :varattno 3
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 3
                       :location 7
                       }
                    :resno 6
                    :resname abalance
                    :ressortgroupref 0
                    :resorigtbl 16438
                    :resorigcol 3
                    :resjunk false
                    }
                    {TARGETENTRY
                    :expr
                       {VAR
                       :varno 65001
                       :varattno 4
                       :vartype 1042
                       :vartypmod 88
                       :varcollid 100
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 4
                       :location 7
                       }
                    :resno 7
                    :resname filler
                    :ressortgroupref 0
                    :resorigtbl 16438
                    :resorigcol 4
                    :resjunk false
                    }
                 )
                 :qual <>
                 :lefttree
                    {INDEXSCAN
                    :startup_cost 0.42
                    :total_cost 42377.43
                    :plan_rows 1000000
                    :plan_width 97
                    :parallel_aware false
                    :parallel_safe true
                    :plan_node_id 2
                    :targetlist (
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 2
                          :varattno 1
                          :vartype 23
                          :vartypmod -1
                          :varcollid 0
                          :varlevelsup 0
                          :varnosyn 2
                          :varattnosyn 1
                          :location -1
                          }
                       :resno 1
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 2
                          :varattno 2
                          :vartype 23
                          :vartypmod -1
                          :varcollid 0
                          :varlevelsup 0
                          :varnosyn 2
                          :varattnosyn 2
                          :location -1
                          }
                       :resno 2
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 2
                          :varattno 3
                          :vartype 23
                          :vartypmod -1
                          :varcollid 0
                          :varlevelsup 0
                          :varnosyn 2
                          :varattnosyn 3
                          :location -1
                          }
                       :resno 3
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 2
                          :varattno 4
                          :vartype 1042
                          :vartypmod 88
                          :varcollid 100
                          :varlevelsup 0
                          :varnosyn 2
                          :varattnosyn 4
                          :location -1
                          }
                       :resno 4
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                    )
                    :qual <>
                    :lefttree <>
                    :righttree <>
                    :initPlan <>
                    :extParam (b)
                    :allParam (b)
                    :scanrelid 2
                    :indexid 16452
                    :indexqual <>
                    :indexqualorig <>
                    :indexorderby <>
                    :indexorderbyorig <>
                    :indexorderbyops <>
                    :indexorderdir 1
                    }
                 :righttree
                    {MATERIAL
                    :startup_cost 0.00
                    :total_cost 1.15
                    :plan_rows 10
                    :plan_width 364
                    :parallel_aware false
                    :parallel_safe true
                    :plan_node_id 3
                    :targetlist (
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 65001
                          :varattno 1
                          :vartype 23
                          :vartypmod -1
                          :varcollid 0
                          :varlevelsup 0
                          :varnosyn 1
                          :varattnosyn 1
                          :location -1
                          }
                       :resno 1
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 65001
                          :varattno 2
                          :vartype 23
                          :vartypmod -1
                          :varcollid 0
                          :varlevelsup 0
                          :varnosyn 1
                          :varattnosyn 2
                          :location -1
                          }
                       :resno 2
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                       {TARGETENTRY
                       :expr
                          {VAR
                          :varno 65001
                          :varattno 3
                          :vartype 1042
                          :vartypmod 92
                          :varcollid 100
                          :varlevelsup 0
                          :varnosyn 1
                          :varattnosyn 3
                          :location -1
                          }
                       :resno 3
                       :resname <>
                       :ressortgroupref 0
                       :resorigtbl 0
                       :resorigcol 0
                       :resjunk false
                       }
                    )
                    :qual <>
                    :lefttree
                       {SEQSCAN
                       :startup_cost 0.00
                       :total_cost 1.10
                       :plan_rows 10
                       :plan_width 364
                       :parallel_aware false
                       :parallel_safe true
                       :plan_node_id 4
                       :targetlist (
                          {TARGETENTRY
                          :expr
                             {VAR
                             :varno 1
                             :varattno 1
                             :vartype 23
                             :vartypmod -1
                             :varcollid 0
                             :varlevelsup 0
                             :varnosyn 1
                             :varattnosyn 1
                             :location 7
                             }
                          :resno 1
                          :resname <>
                          :ressortgroupref 0
                          :resorigtbl 0
                          :resorigcol 0
                          :resjunk false
                          }
                          {TARGETENTRY
                          :expr
                             {VAR
                             :varno 1
                             :varattno 2
                             :vartype 23
                             :vartypmod -1
                             :varcollid 0
                             :varlevelsup 0
                             :varnosyn 1
                             :varattnosyn 2
                             :location 7
                             }
                          :resno 2
                          :resname <>
                          :ressortgroupref 0
                          :resorigtbl 0
                          :resorigcol 0
                          :resjunk false
                          }
                          {TARGETENTRY
                          :expr
                             {VAR
                             :varno 1
                             :varattno 3
                             :vartype 1042
                             :vartypmod 92
                             :varcollid 100
                             :varlevelsup 0
                             :varnosyn 1
                             :varattnosyn 3
                             :location 7
                             }
                          :resno 3
                          :resname <>
                          :ressortgroupref 0
                          :resorigtbl 0
                          :resorigcol 0
                          :resjunk false
                          }
                       )
                       :qual <>
                       :lefttree <>
                       :righttree <>
                       :initPlan <>
                       :extParam (b)
                       :allParam (b)
                       :scanrelid 1
                       }
                    :righttree <>
                    :initPlan <>
                    :extParam (b)
                    :allParam (b)
                    }
                 :initPlan <>
                 :extParam (b)
                 :allParam (b)
                 :jointype 0
                 :inner_unique true
                 :joinqual (
                    {OPEXPR
                    :opno 96
                    :opfuncid 65
                    :opresulttype 16
                    :opretset false
                    :opcollid 0
                    :inputcollid 0
                    :args (
                       {VAR
                       :varno 65000
                       :varattno 1
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 1
                       :varattnosyn 1
                       :location 70
                       }
                       {VAR
                       :varno 65001
                       :varattno 2
                       :vartype 23
                       :vartypmod -1
                       :varcollid 0
                       :varlevelsup 0
                       :varnosyn 2
                       :varattnosyn 2
                       :location 78
                       }
                    )
                    :location -1
                    }
                 )
                 :nestParams <>
                 }
              :righttree <>
              :initPlan <>
              :extParam (b)
              :allParam (b)
              :limitOffset <>
              :limitCount
                 {CONST
                 :consttype 20
                 :consttypmod -1
                 :constcollid 0
                 :constlen 8
                 :constbyval true
                 :constisnull false
                 :location -1
                 :constvalue 8 [ 10 0 0 0 0 0 0 0 ]
                 }
              :limitOption 0
              :uniqNumCols 0
              :uniqColIdx
              :uniqOperators
              :uniqCollations
              }
           :rtable (
              {RTE
              :alias
                 {ALIAS
                 :aliasname b
                 :colnames <>
                 }
              :eref
                 {ALIAS
                 :aliasname b
                 :colnames ("bid" "bbalance" "filler")
                 }
              :rtekind 0
              :relid 16441
              :relkind r
              :rellockmode 1
              :tablesample <>
              :lateral false
              :inh false
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b 8)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
              {RTE
              :alias
                 {ALIAS
                 :aliasname a
                 :colnames <>
                 }
              :eref
                 {ALIAS
                 :aliasname a
                 :colnames ("aid" "bid" "abalance" "filler")
                 }
              :rtekind 0
              :relid 16438
              :relkind r
              :rellockmode 1
              :tablesample <>
              :lateral false
              :inh false
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b 8 9)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
              {RTE
              :alias <>
              :eref
                 {ALIAS
                 :aliasname unnamed_join
                 :colnames ("bid" "bbalance" "filler" "aid" "bid" "abalance" "filler")
                 }
              :rtekind 2
              :jointype 0
              :joinmergedcols 0
              :joinaliasvars <>
              :joinleftcols <>
              :joinrightcols <>
              :lateral false
              :inh false
              :inFromCl true
              :requiredPerms 2
              :checkAsUser 0
              :selectedCols (b)
              :insertedCols (b)
              :updatedCols (b)
              :extraUpdatedCols (b)
              :securityQuals <>
              }
           )
           :resultRelations <>
           :appendRelations <>
           :subplans <>
           :rewindPlanIDs (b)
           :rowMarks <>
           :relationOids (o 16441 16438)
           :invalItems <>
           :paramExecTypes <>
           :utilityStmt <>
           :stmt_location 0
           :stmt_len 107
           }

2021-01-17 19:40:52 JST [client backend] STATEMENT:  SELECT * FROM pgbench_branches b
                  JOIN pgbench_accounts a ON b.bid = a.bid ORDER BY a.aid LIMIT 10;
```


