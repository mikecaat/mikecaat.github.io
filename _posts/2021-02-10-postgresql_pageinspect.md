---
layout: post
title: PostgreSQLのpageinspect extension
tags:
- postgresql
- extension
- pageinspect
---

## pageinspect

- デバック時に低レベルなデータベースのページ内容を調べることが可能
- 一般的な関数, ヒープ関数, B-tree関数, BRIN関数, GIN関数, HASH関数
- [pg_pageinspect_plus](https://github.com/harukat/pg_pageinspect_plus)では、データ型も考慮してくれる


## pageinspectの実装

- get_raw_page()の出力結果(byte)をベースに各関数が実装されている
  - get_raw_page()は、コアのbufmgrで実装されているReadBufferExtended()でブロック情報を取得するだけ
  - get_raw_page()以外の情報は、内部で管理されているデータ構造で出力してくれる

- データ構造に関する情報は、ドキュメント、ソースコードのREADME、google検索でも情報は豊富にあるイメージ
  - https://www.postgresql.jp/document/12/html/pageinspect.html
  - https://www.postgresqlinternals.org/chapter4/ 

- 例えば
  - PageHeaderData構造体 in "bufpage.h"
    - lsn, checksum, free/visible flag, free spaceに関するoffset, line pointer array など
  - BTMetaPageData構造体 in "nbtree.h"
    - 1ブロック目がメタデータ管理用ブロック

## その他

- インデックスとテーブルは別ファイル(oid)で管理される
