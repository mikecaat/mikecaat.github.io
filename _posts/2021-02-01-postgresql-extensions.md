---
layout: post
title: PostgreSQLで有名なcontribまとめ
tags:
- postgresql
- extension
- contrib
- summary
---

## 本体同梱

- adminpack
  - サーバの"ログファイル"の書き込み・名前変更・削除などを実施するための関数を提供
  - pgAdminなどで使われているらしい

- amcheck
  - リレーション構造の"一貫性を検査"する機能を提供
  - B-Treeインデックスのみのよう。順序が入れ替わっていないかなどをチェックできるよう

- auth_delay
  - 認証に失敗したら、指定したミリ秒間だけ待機する機能を提供
  - "総当り攻撃"にかかる時間を増やすことができる

- auto_explain
  - 実行が遅い文の"実行計画を自動的に"ログに記録する機能を提供
  - スーパユーザであれば、セッション単位で読み込ませて利用可能

- bloom
  - "bloom filter"によるアクセスメソッドを提供

- bool_plperl  # ドキュメント化が漏れている
  - perlの真偽値(bool)の仕様の分かりにくさをwrapするための関数を提供
  - perlでは、'f'が渡されても、trueと判断されてしまうらしい

- btree_gin
  - "B-treeと同等の動作をするGIN"演算子クラス
  - 素のB-treeでビットマップよりも効率的らしい
  - デバック用らしいが使いみちが思いつかない。正しさだけなら、普通のB-Treeでいいはず

- btree_gist
  - "B-treeと同等の動作をするGiST"インデックス演算子クラス

- citext
  - "大文字小文字の区別がない"文字列型を提供
  - lower関数を使うと、SQLが冗長・共通のインデックスが使えないなどの課題を解決できる

- cube
  - "多次元立方体"を表すためのcubeデータ型を提供
  - 立方体間の演算（&&, <=, <->など多数）を提供

- dblink
  - "リモートデータベース"での操作を提供
  - postgres_fdwがより新しく互換性高い

- dict_int
  - 整数向けの辞書テンプレートを追加

- dict_xsyn
  - 単語を"類義語の集まり"に置き換えて、検索できるようにする辞書テンプレートを追加

- earthdistance
  - 地表面上の"大圏距離を計算"する。ただし、地球は完全な球という前提
  - postgis使うほうが良いと思う

- file_fdw
  - サーバ上の"ファイルをテーブル"として見えるようにする機能などを提供
  - programの実行も可能 // pg_waldumpをダンプして結果をviewに表示もできそう? フォーマットをcsvとかにできると良さそう

- fuzzystrmatch
  - 文字列間の類似度や相関度を決める関数を提供
  - 音声・文字に対応しているよう。類似の文字を生成することも可能

- hstore
  - KVSとして利用するためのhstore型を提供。インデックスも使える
  - 事前にテーブル構造などを定義する必要がない

- hstore_plperl
  - PL/Perlでhstore型を利用するためのextension

- hstore_plpython
  - PL/Pythonでhstore型を利用するためのextension

- intagg
  - 整数型の集約子と列挙子を提供。もう使われていない
  - 組み込み関数 array_agg, unnest関数が存在する([利用例](https://qiita.com/choplin/items/9d5e2ff8721fb9509bf8))

```
CREATE FUNCTION int_array_sort_uniq(int[]) RETURNS int[] AS $$
    SELECT
        array_agg(DISTINCT i ORDER BY i)
    FROM
        unnest($1) AS t(i)
$$ LANGUAGE SQL;

$ SELECT int_array_sort_uniq(ARRAY[2,4,3,2,1]);
 int_array_uniq_sort
---------------------
 {1,2,3,4}
(1 row)
```

- intarray
  - 整数の配列操作用の便利関数を提供。インデックスも提供
  - ソートや重複排除、重なりや包含検索など

- isn
  - ISBNなどの国際的な標準製品番号に従うデータ型を提供
  - 一部、変換もサポート

- jsonb_plperl
  - jsonb型をPerlのarray, hash, scalarなどに適切に変換する機能を提供

- jsonb_plpython
  - jsonb型をPythonのdictionary, list, scalarなどに適切に変換する機能を提供

- lo
  - ラージオブジェクトの保守作業サポートを提供
  - PostgreSQLでは、ラージオブジェクトを更新しても削除されない仕様になっている
    - このextensionを利用することでトリガ経由でlarge objectを削除してくれる
    - `vacuumlo` ユーティリティコマンドで不要になったlarge objectを削除することも可能

- ltree
  - 階層ツリーを模擬したデータラベルを表現できるltree型を提供
    - LDAP的な感じで使えそう
  - 各種便利な関数やインデックス機能も提供

- ltree_plpython
  - PL/Python向けでltree型を提供。listに変換される

- oid2name
  - OIDとPostgreSQLデータディレクトリ内のファイルノード解決
  - contribディレクトリに入っているけど、クライアントアプリケーション
  - 引数を指定しなければ、全データベースのOIDを出力させたり、-sオプションだけつければ、全リレーションのOIDを出力させたりも出来て、便利そう

- old_snapshot
  - v14で追加。GUCの `old_snapshot_threshold` の正しさを検査するためのモジュール
  - `old_snapshot_threashold` によって、スナップショットを取得しているクエリが存在したら、エラーを返すようになる
    - vacuumでデータを削除することができる

- pageinspect
  - データベースのページ内容を調べる関数などを提供
  - [pg_pageinspect_plus](https://qiita.com/harukat1232000/items/4df4637ba6228ae8ada6)も利用することでbyte列の型変換も可能

- passwordcheck
  - 弱いパスワードを弾くための機能を提供
  - ただ、簡易なチェックのみのよう（8文字未満, user名を含む, 大文字小文字を含む）

- pg_buffercache
  - 共有バッファキャッシュ上で起きていることをリアルタイムに確認するビューを提供
  - 例えば、テーブルごとのバッファ上のページ数やダーティページ数などを確認可能
    - ピン留めしているバックエンド数も分かる

- pgcrypto
  - md5, sha1, sha256, crypt, pgpなどの暗号関数を提供

- pg_freespacemap
  - Free Space Mapを検査するための方法を提供
  - ブロック（ページ）内の空き領域サイズを確認できる

- pg_prewarm
  - バッファにデータをロードするための方法を提供
  - 手動 or 自動で実行可能

- pgrowlocks
  - 行ロックの情報を示す関数を提供
  - coreで確認出来ないのは、各行ごとにロック情報をロックマネージャに保持していないため([参考](https://masahikosawada.github.io/2018/09/04/Row-locking-and-locking-on-transactionid/))と思われる

- pg_stat_statements
  - サーバで実行されたSQLの実行時の統計情報を記録する手段を提供

- pgstattuple
  - タプルレベルの統計情報を提供
  - 例えば、タプル数、無効なタプル数など

- pg_trgm
  - トライグラム一致に基づく英数字の類似度を元にした高速検索用のインデックス、関数、演算子を提供

- pg_visibility
  - visibility mapとページレベルでの可視性を検査するための関数を提供
  - visibility mapを再構築することも可能

- postgres_fdw
  - 外部のPostgreSQLサーバに格納されたデータへのアクセス方法を提供
  - ただし、ON CONFLICT時などの制約もあるので事前にドキュメントを読むこと

- pg_surgery //dev
  - 破損したリレーションを復旧するための関数群を提供
  - tupleをdead/freezeさせることができる

- seg
  - 浮動小数点区間を表現するsegデータ型を提供
  - 重なりや包含などを検索する関数も合わせて提供

- sepgsql
  - SELinuxのセキュリティポリシーに基づいたアクセス制御を提供

- spi
  - Server Programming Interface(SPI)、トリガを使用した動作例を提供

- sslinfo
  - 接続時のSSL証明書に関する情報を提供

- tablefunc
  - 正規乱数値など便利なテーブルを返す関数を提供

- tcn
  - テーブル上の変更を監視者に通知するトリガ関数を提供
  - 簡単に動作確認した感じだと、同一セッション内でしか通知されない様子
  - ユースケースがイマイチ分からない

- test_decoding
  - ロジカルレプリケーション用(?)WALの論理デコード用のプラグイン

- tsm_system_rows
  - table sampleing method を提供
  - ORDER BY RANK() でも同じようなことはできるはず

- tsm_system_time
  - table sampling methodを提供。時間で指定

- unaccent
  - アクセントを取り除く全文検索用の辞書
  - À -> A にする感じ

- uuid-ossp
  - UUIDを生成するライブラリ。ただし、pgcryptoモジュールを水晶

- vacuumlo
  - 参照されていないラージオブジェクトを削除するコマンド
  - PostgreSQLでは、ラージオブジェクトを更新しても削除されない仕様になっている
    - `vacuumlo` ユーティリティコマンドで不要になったlarge objectを削除することも可能

- xml2
  - XPath問い合わせとXSLT機能を提供
    - XPath問い合わせは、XMLの情報の一部（要素や属性など）にアクセスするための指定記述
    - XSLT機能は、XMLからXMLへ, HTMLへの変換などができる機能
  - PostgreSQLのコアでXML型をサポートしているので現在は後方互換性のために存在

## その他

沢山あるので気になるものを一部だけ抜粋
- https://github.com/dhamaniasad/awesome-postgres
- https://github.com/devton/awesome-postgresql

- [citus](https://github.com/citusdata/citus)
  - 複数ノードで分散処理させる機能を提供
  - 2PCによるatomic commit, グローバルデッドロック検知などを有する
  - atomic visibilityは、ドキュメント上には見当たらず、(たぶん)ない

- [pg_stat_kcache](https://github.com/powa-team/pg_stat_kcache)
  - クエリ単位でのCPU使用状況、Disk I/Oなどを提供
  - [getrusage](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/getrusage.2.html)を使って実装されている

- [pg_store_plans](http://ossc-db.github.io/pg_store_plans/)
  - 実行された各クエリごとの実行計画を提供
  - pg_stat_statementsの実行計画版

- [pg_hint_plan](http://pghintplan.osdn.jp/pg_hint_plan-ja.html)
  - 実行計画をユーザが制御するためのヒント句を提供
  - 安定運用のためなどに導入される
  - 議論はされているが、[実行計画の改善が出来なくなるなど](https://wiki.postgresql.org/wiki/OptimizerHintsDiscussion)でコアには導入されていない

- [pg_statsinfo, pg_stat_reporter](http://pgstatsinfo.sourceforge.net/index_ja.html)
  - PostgreSQLの統計情報やOSのリソース情報などを時系列で取得・保存するツール
  - 保存したデータを活用した可視化も提供

- [Barman](https://www.pgbarman.org/)
  - PostgreSQLのバックアップを実現するためのツールの１つ([PostgreSQL wiki](https://wiki.postgresql.org/wiki/Ecosystem:Backup))
  - 2020年にも更新されており、メンテされており、第一候補(?)

- [Postgis](https://postgis.net/docs/manual-2.4/postgis-ja.html)
  - GistベースのR木空間インデックス、GISデータ型、各種関数を多数サポート
    - ベクタ、ラスタ、トポロジ、OpenGIS(OGC規格)対応
  - 地理情報を扱うシステムでよく使われている印象

- [cstore_fdw](https://github.com/citusdata/cstore_fdw)
  - columnar storeとしてPostgreSQLを使えるようにするための機能を提供
