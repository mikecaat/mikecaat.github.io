---
layout: post
title: MVCC(Design Decisions) CMU 15-721 メモ
tags:
- cmu
- mvcc
- in-memory database
- OLTP
---

## まとめ

- [サマリ](https://15721.courses.cs.cmu.edu/spring2020/notes/03-mvcc1.pdf)が詳しい

- MVCCのデザインポイント
  - concurrency control protocol
    - tuple metadata, timestamp ordering, two-phase locking
  - version storage
  - garbase collection
  - index management
  - transaction id wraparound

## mvccとは

- 1つの論理的なtupleに対して、複数の物理バージョンを保持する
- write-writeのみblock
- ロックなしで一貫したsnapshotが取得可能
  - MySQLでは、read onlyの場合は、ボトルネックを避けるため、txidも付与しない
- time-travel queryが簡単
  - PostgreSQLもサポートしていたことがあったがdisk容量を大量に使うので今はサポートしていない(1999年)
  - ただ、今は時系列DBなど、要望は増えてきている

- isolation levelは、snapshot isolation（が多い?）
  - write skew conflictは起きる

## concurrency control protocol

- in-memoryの場合も、disk-baseとほぼ同じ。違いは下記

- tuple-metadata
  - tuple内のヘッダーにロックを保持しているtxid, begin-tx, end-tx(visible validation用)などの情報も保持しておく
    - // PostgreSQLだと、diskベースだけどtuple内保持していた気がする
    - 個々のtupleでheaderを用意するので、disk容量のオーバヘッドは大きくなりがち
      - バッチ的にメタデータを書き込むなどの対応が考えられるが、その場合、計算リソース(tupleを探す時間?)が増える
      - storage volume vs compute resourceのtrade-off

- timestamp ordering
  - readは、begin-ts - end-tsの間のバージョンのものを読む
  - writeは、varidation -> latch獲得で実施する
    - ヘッダーにread-txを追加して、veridationする(abortする)
      - read-tsが後に開始したtxだったら、write出来ない（過去を書き換えられない）のでabort
    - latchを取得して、次のバージョンを作成、commitするまで離さない

- two-phase locking
  - ヘッダーにread-cntを追加して、shared lockを取得できるようにする
  - read時にread-cntをincrement。write時にはtxn-id, read-cntが0ならexclusive lock

- 32bitでtxn-idを管理していると、周回問題が発生する
  - 過去に遡るような挙動に見える
  - postgresqlの場合、txn id wraparoundを利用
    - "frozen"フィールドを用意して、txn idが何であれ過去のデータであることを見分けられるようにしている

## version storage

- delta storage
  - カラム数が多いとstorage容量の削減可能
  - 容量削減により、キャッシュ使用率の向上も見込める
  - versionが増えると、deltaを保管していくコストが高くなる

## garbage collection

- tuple-level
  - background vacuuming: 他のスレッドが定期的にscanして不要なバージョンを削除
    - // Postgresql方式
    - scanコストをさげるために、dirty block bitmap(visibility map)を利用する
    - バージョン更新があったblockを記録しておく
  - cooperative cleaning: 最も古いtxが不要なバージョンはその場で消していく
    - ただ、readされないデータは削除できない(dusty corner)ので別途対応が必要

## index management

- N2O: Newest to Oldest

- [Why Uber Engineering Switched from Postgres to MySQL](https://eng.uber.com/postgres-to-mysql-migration/)
  - secondary indexが多く、updateも多いワークロードの場合、physical address方式(PostgreSQL)よりも、logical address方式(MySQL)の方が有利
    - PostgreSQLの場合、tuple更新時にsecondary indexも含めて、全てのpointerを変更する必要がある
    - page読み込み/書込みも増えるし、wal書き出しも増えるし

- mvccの場合、index内のduplicate keyを保証する必要がある
  - indexにはバージョン情報は含めない
    - ただし、MYSQLのようなindex-organized tables(InnoDB)の場合は例外
      - leaf nodeにデータを格納しているから(?)
  - indexのkeyに紐づく形でlist形式で各バージョンへのポインタを保持する
  - list形式かは分からないが、PostgreSQLでも同様っぽい([参考](https://en.ppt-online.org/7540))


## zheapの話

- [zheapの話](http://rhaas.blogspot.com/2018/01/do-or-undo-there-is-no-vacuum.html)
  - [実装](https://github.com/EnterpriseDB/zheap)?
  - [内部実装の説明](https://www.postgresql.eu/events/pgconfeu2018/sessions/session/2104/slides/93/zheap-a-new-storage-format-postgresql.pdf)
  - [動かしてみた記事](https://qiita.com/U_ikki/items/7a7aa100842d1061a57a)

- 更新が多いワークロードだと、一時的にheap領域の使用量が増加する
  - 減らそうとしても、VACCUME FULLやpg_repackを利用する必要があるが、table全体を書き換える必要がある
  - 更新を複数のtxに分割して、少しづつ行うことで、autovacuumで細かい単位で削除する工夫はできる
  - 1つでもlong-running transactionが存在すると、そのtxが使う可能性があるtupleを消せないのはどうしようもない

- zheapでは、undo logを新たに導入して、heap領域の肥大化を防ぐ
  - 更新時の不要なテーブル拡張を防ぎ、全heapを対象にしたvacuumを防ぐ
  - UPDATE時に古いバージョンのデータをundo logに移動する  //mysql,oracleもそう
    - heap自体は最新のデータのみを保存しておく
    - heap領域（テーブル）全体を対象に不要なtupleを見る必要がなくなる
      - vacuum不要に
      - xminを記録することで、undo logをシーケンシャルに削除できる?([参考](https://www.postgresql.eu/events/pgconfeu2018/sessions/session/2104/slides/93/zheap-a-new-storage-format-postgresql.pdf)p.23)
  - abortしたら、元に戻す
    - 元に戻すコストは高いが、経験則から一般的にabortされることはほぼない

- メリット
  - in-placeにupdateするので、可能な限り、テーブル領域の肥大化を防ぐ
  - txが終了したタイミングで不要なデータはなるべく削除するので、undo領域の肥大化を防ぐ
  - frozen処理も不要で、関連するundo領域を削除すればOK
  - heap領域が大量に増えるようなワークロードだと、パフォーマンス向上が見込める
    - // 理由は、page書込みが分散しないので、キャッシュヒット率があがるとか?

- デメリット
  - 複数txが同一ページを操作していると、readのコストは増加
    - 競合が発生しやすいということ? でも、heapでも同じでは?
  - abortのコストは増加: 再度table情報の書き直しが必要だから
  - updateのコスト増加
    - 実テーブルとundo log領域を2回書く必要がある
    - ただ、メモリ上の操作なら気にならない?
