---
layout: post
title: Clock-SI Snapshot Isolation for Partitioned Data Stores Using Loosely Synchronized Clocks
tags:
- paper
- distributed database
- snapshot isolation
---

## 概要

PostgreSQL Wikiの[Scaleout Design](https://wiki.postgresql.org/wiki/Scaleout_Design)で紹介されていた論文を読んだメモ

## まとめ

- どんな研究か： localのtimestampのみを利用して、SIを実現するための研究
- 先行研究との違い：
  - centralizedなtimestampを利用していない <= conventional SI
  - localのtimestampのずれを想定していること <= spanner
- 技術や手法のキモ： read/prepare/commitでdelayを挟み、eventのtotal orderを実現
- 有効性の確認方法： kvs, YCSBベンチマーク
- 次に読みたい論文: spannerなどを意識したcockroachdb, yugabytedbの実現方法

## abstract

- Clock-SIは、パーティション化されたデータストアに対して、分散SIを実現するプロトコル
- loosely synchronized clocksを利用することで、centralized timestampを利用しない
  - 可用性向上：SPOFがない, 性能向上: ボトルネックがなくなる, latency/throughput向上
  - read-onlyであれば、50%の性能向上が見込める

## introduction

- SIではupdateもatomicに実施される
  - snaphostは、snaphost timestampで認識される
  - updateのcommitは、commit timestampで認識される

- partitioned data store前提
  - 既存の論文は、timestampが中央管理
  - commit時は2PC
   - Clock-SIでも同じ
  - centralized timestampの払い出しがSPOF、パフォーマンスのボトルネックになりがち
    - Clock-SIでの改善ポイント
    - "loosely synchronized clocks"を利用して、snaphostやcommitのtimestampを払い出す

- Clock-SIを利用して、consistentなsnapshotを作成できる
  - // 過去の論文ではtotally order eventsとの記述があるが、Clock-SIも? => そう

- 難しいポイント
  - loosely synchronized clocks前提
  - clock skew, pending commitによって、まだ適切でないsnapshotが作成されるかも
    - 適切になるまで待つことで解決(Fig1, Fig2)
    - 最適化の方法も1つ考えられる

- 既存のSIと比較したときのtrade-offも評価する
  - kvs, YCSB

- 想定するモデル
  - NTP: clock skewあり
  - kvs: get, put, delete
  - 最初にクライアントが接続したパーティションを"origination partition"
    - snaphost/commit時のtimestampは、そのノードのlocal clockを利用する

- SIの特性
  - (1) 一貫性のあるsnapshotを取得
    - snapshot timestamp前にcommitされたデータは全て見える
    - snapshot timestamp後にcommitされたデータは見えない
    - 未commit/abortも見えない
  - (2) commitは、どのノードから見ても同じ順番になる
    - commit timestampで認識する   // 全く同じ時間だったらどうするのだろう
  - (3) write-write conflictが発生したらabrotする

## Clock-SI

### Read Protocol

- 2つの処理が重要
  - (1) snapshot timestampを割り当て
    - local clockを⊿だけ遅らせた値で取得する  // ⊿は後述。一番遅れてるノード?
  - (2) 特定条件化でread/updateを遅らせることで一貫性を確保する
    - commit中で、snapshot timestamp > commit timestampであれば、commitされるまで待つ
    - prepare中の場合は、commit timestampを確認する
    - updateについても、snapshot time > local timeであれば、snapshot timeになるまで待つ

### Commit Protocol

- readのみのtxは何もしない

- updateのときは、commit timestampの割当が必要
  - (1) 特定パーティションのみの更新
    - write-write conflictをチェック
    - local timeをcommit timeにして、commit完了
  - (2) 複数パーティションの更新
    - 2PCでcommitする
      - prepareでupdateしたpartitionにprepareを実施
        - prepareのリクエストを受けた各partitionは、conflictチェック
        - prepare時間にlocal時刻を設定し、リクエストしたpartitionにも返却
      - prepareで返却された時刻(+ localも?)の最大時刻をcommit timeにする
        - 各ノードのprepare時間よりも必ず後の時刻をcommit timeにする
      - commitを各partitionに伝える
  - commiting/commitedへの状態変更 や logの書き出しなども適切なタイミングで変更

### Chooosing Older Snapshots

- snapshot timestampの⊿の選び方
  - 0にすれば、最新の値がかならず見える => ただし、delayが大きくなるかも
  - ⊿の最大値 => ただし、delayを小さくできるが、古いデータを見たり、abort rateが高くなる
    - (1) commitするために必要な時間 + nw latency
    - (2) 最大値のclock-skew - nw latency
    - 定期的に全partition内で確認する

### Correctness

- (1) txはtotal orderでcommit  // eventually consistent?
- (2) read consistent snapshot: snapshot timestampより先のデータは読まない
- (3) write-write conflictは解決された上でcommitされる: 2PC

- 実際にcomimtされた時間通りにはcommitされないことに注意 // Linearizabilityは満たさない

## Discussion

- Availability向上
  - centralizedなtimestampがないのでSPOFなし
  - 特定のpartitionが使えなくてもそれ以外は影響受けない

- Communication costが減る  // latency向上
  - 既存のSIだと、最初にcentralizedなtimestamp払い出しを聞きにいくコストが発生
  - Clock-SIだと、ローカルのtimestampを利用するため、そのコストが発生しない

- Scalability向上
  - centralized timestampがないので、一箇所がボトルネックにならない

- Session consistency
  - 同一セッション内で書いたものが必ず見える
  - client側で各txの実行時刻を保存しておくことでそれ以降のデータを必ず見えるようにする

- Recovery
  - 既存の2PC, WAL, checkpointの仕組み

## analytical model〜

- ⊿を小さくすること・大きくすることに関する証明

- その他 略

## その他

- Spannerとの違い
  - Clock-SIは、一般的な物理時間(ntp)で同期が取れていないことも想定
  - Spannerは、原子時計と同期が取れているclock前提
