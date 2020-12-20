---
layout: post
title: An Evaluation of Concurrency Control with One Thousand Cores
tags:
- paper
- in-memory database
- OLTP
- cmu
---

## 概要

CMUのAdvanced Database Systemsで紹介されていた論文を読んだメモ
- X. Yu, et al., [Staring into the Abyss: An Evaluation of Concurrency Control with One Thousand Cores](https://15721.courses.cs.cmu.edu/spring2020/papers/02-inmemory/p209-yu.pdf), in VLDB, 2014
- Staring into the Abyssは「底知れぬ絶望を味わう」という意味。それだけ難しい

## まとめ

- どんな研究か： メニーコア時代に既存の並行処理アルゴリズムでの実現性を確認した研究
- 先行研究との違い： メニーコア時代の並行処理に焦点を絞った点
- 技術や手法のキモ： 既存の有名な7つの並行処理アルゴリズムを様々なworkloadで横比較
- 有効性の確認方法： 1024コアまでシュミレーション。YSCB/TPC-Cを動作させて、スケール性を特定
- 次に読むべき論文： Opportunities for Optimism in Contended Main-Memory Multicore Transactions
- 個人的メモ
  - 完璧な並行処理アルゴリズムはないが、workloadごとの得意・不得意を意識すれば、まだ良い。ただ、TCP-Cは全てでダメ
  - 主なボトルネック -> 2PL: lock thrasing, T/O: timestamp allocation, private spaceへのlocal copy

## abstract

- many core時代。10〜100コア/チップ
- concurrency controlに関して、dbmsのscalabilityを落とさないデザインが必要
- 7つのアルゴリズムを実装して、on-memoryなDBMS上で1024コアをシュミレーションして検証
- どのパターンでも1024コアまでスケールしなかった。ただし、それぞれ理由が異なる
- HW(CPU)と組み合わせたDBMSの抜本的なアーキテクチャの再構築が必要


## introduction

- many-coreの時代になってきているが、DBMSは上手く扱えるように適用されていない
- 本論文では、1000コアになったときにOLTPにどのようなことが起きるのかを確認する
- 7つのアルゴリズムで検証を実施したが、1000コアまでのスケール性は実現できなかった
- bottleneckを特定して、新たなconcurrency controlの方法について議論したい
- ソフトウェアだけでは解決できないので、HWを含めた解決策を検討していく必要がある


## concurrency control schemes

- concurrency control schemesとは何かという話

- OLTPのtxの特徴は3つ
  - short-lived, 少量のデータを扱う, (パラメータは違うが)同一クエリが多い
  - ACID特性を担保する必要あり

- アルゴリズムは7つ // 今回の評価対象
  - Two-Phase Locking(2PL)ベース
    - DL_DETECT: 2PL with deadlock detection
    - NO_WAIT: 2PL with non-waiting deadlock prevention
    - WAIT_DIE: 2PL with wait-and-die deadlock prevention
  - Timestamp Ordering(T/O)ベース
    - TIMESTAMP: Basic T/O algorithm
    - MVCC: Multi-vertion T/O
    - OCC: Optimistic concurrency control
    - H-STORE: T/O with partition-level locking

- 2PL: 悲観的アプローチ
  - rouwing phaseとshrinking phaseでロック取得・リリースしてconflictを発生させない
  - DL_DETECT: waits-forグラフを作成して、循環(deadlock)をチェック
  - NO_WAIT: deadlockっぽければabortする
  - WAIT_DIE: deadlockが起きないようにabortさせる
    - 同一lockを取得した場合はtx実行時間の長さが短い方がabort

- T/O: 単調増加していくtimestampを使って、conflictを適切な順番で解消していく
  - 上記4つの大きな違いは
    - (1)conflictをチェックする粒度
    - (2)conflictをチェックするタイミング
  - TIMESTAMP: (1)tuple, (2)read/write
    - readは、repeatable readを保証するため、ローカルにコピー
  - MVCC: (1)tuple, (2)write // (2)については明記されていないのでたぶん
  - OCC: (1)tuple, (2)tx終了時
    - tx終了時まではprivate workspaceで処理
    - 競合時間が短いので、インメモリ系のDBMSで使われている(Silo, Hekaton)
  - H-STORE: (1)partition, (2)tx開始前


## many-core dbms test-bed

- 1024コア/チップの環境は存在しないので、CPUシュミレーション(Graphite)で検証
  - アーキテクチャは、[こちら](http://groups.csail.mit.edu/carbon/wordpress/wp-content/uploads/2012/07/graphite_intel_tutorial-intro_and_overview.pdf)が詳しい
  - 複数ホストを組み合わせて、論理的に１つのチップに見せる
  - CPU命令の同期を取るため、cpu cycleの精度を落としている
    - // パフォーマンスは落としているけど、1024コア/単一チップを模擬できるという理解

- DBMSは、独自のインメモリDBMSを実装
  - コア数分だけ並列で処理, 行指向, hash tableインデックス
  - 比較対象のconcurrency controlを変更できるようにpluggable lockマネージャを実装
  - NW通信オーバヘッドを無視するため、それぞれのworkerごとのキューでtxを保持

- 処理状況に関する実行時間は下記の6分類で計測
  - USEFUL WORK: 業務ロジックの実行やtuple操作時間
  - ABORT: roll backにより、全ての変更を元に戻すときのオーバヘッド時間
    - // roll backしたtxがabort前までに処理していた時間も含まれる?
  - TS ALLOCATION: concurrency control用にuniqueなtimestampを付与するための時間
  - INDEX: hash indexの作成にかかった時間
  - WAIT: lock(2PC)やtuple更新状況の確認待ち(T/O)にかかった時間
  - MANAGER: lock managerやtimestamp managerの処理にかかった時間

- Workloadは、YCSBとTCP-Cの2種類を利用
  - YCSB: workloadのパラメータ（read/writeの割合, 競合のしやすいさなど）を変更しやすい
  - TPC-C: 倉庫数が与える影響を確認できる

## design choices && optimizations

- 検証対象の2PL, T/Oのボトルネックをできるだけ取り除いた上で検証する

- 共通する最適化
  - それぞれのスレッドがメモリプールからメモリ領域を取得するような独自macllocを実装
    - // PostgreSQLのmemory context的なものだと思う
  - Lock Tableを実装し、lockが集中することを防ぐ
    - 個々のtupleごとにlock状況を管理するため、メモリオーバヘッドは大きくなる

- 2PLの最適化
  - Deadlock検知(DL_DETECTの場合)をパーティション単位で実施し、lockフリーに
  - 2PLはロックにより、並行して処理しているtxが待たされることがボトルネック
    - theata=0.8だと16コアで限界  // 最適化できないということ?
  - NO_WAITの場合は、abortさせるまでの待ち時間によって、abort rate-スループットが変わる
    - YCSBの結果より、最もTPSが高かった100μsの待ち時間とする

- T/Oの最適化
  - timestamp割当部分がmutexesを使うとボトルネックになる。他の方法としては、
    - (1) baches atomic addition: リクエストにまとめて応答を返す
    - (2) clock-base: ローカルのコアが持つ論理時間とthread_idの組み合わせを利用
      - スケーラビリティは高いが、コア数が増えるとsyncの時間かかる
      - 2014年時点でサポートしているコアも少ない
    - (3) hardware counter: HW側で物理的にCPUの中心に配置しておく
      - ただ、（2014年時点で?）サポートしているCPUない
    - シュミレーション結果では、(2)が最もスケーラビリティあり
  - OCCのときにtupleのバージョンチェック時のロック粒度を短くするため、tuple単位で実施
  - H-STOREのときに他partitionへのクエリは、他のスレッドにクエリを飛ばすのではなく、直接データを読みに行く


## experimental analysis

### read-only workload

- OCC, TIMESTAMPは、性能悪い

- ボトルネックは
  - OCC: timestamp allocation。tx開始前とvalidation前の2回割当が必要
  - OCC/TIMESTAMP: read-onlyでもローカルへのコピー

### write-intensive workload

- medium contention
  - 2PL
    - NO_WAIT, WAIT_DIEだけが512core以上でもスケールしている
      - ただ、Abort rateが高い点には注意。roll back対象のテーブル・インデックスなどが多い場合はより影響が大きい
    - DL_DETECTはダメダメ。deadlock
  - T/O
    - 512coreまでは概ね良好。それ以上はスケールしない
    - OCCはabort rateが高い

- high contention
  - abort処理に時間を使っていて、どのアルゴリズムでもスケール性ない
    - YCSBでtheta=0.7以上
  - many-coreの恩恵が受けられないworkload

### アクセスするtuple数が与える影響

- アクセスするtuple数が増えると、自然と競合も発生しやすくなる

- tuple数が少ないと2PLが有利。多くなるとT/Oが有利
  - tuple数が少ないとき
    - 競合が発生しないので、2PLのdeadlock対応の影響少ない
    - T/Oのtimestamp割当コストが大きく見える

### read/writeの混在が与える影響

- どのパターンでもreadが多いほうが性能良い
- TIMESTAMP/OCCは、やはりreadのlocal copyの影響でread割合が多くなっても性能向上の割合低い

### databaseのpartitioningが与える影響

- read-onlyだと、コア数が少ないときは、他のアルゴリズムよりもかなり性能
  - ただ、T/Oなので、コア数が多いときはtimestamp allocationがbottlneckになるのは避けられない

- txが操作するpartitionが増えると、性能が大幅に劣化する
  - 〜2個のパーティションまでは1000コアでもスケールしている。それより上はダメ
  - パーティション間でのやり取りが増えるため

### TPC-C

- 倉庫数が少ないと競合が発生しやすいのでPayment only(upodate)のスケール性は低い
- 1024にしても、どのアルゴリズムでもスケールしていない
  - 特にNewOrderのリクエストのスケール性が低い


## discussion

- どのアルゴリズムでも、全workloadにおけるscalabilityを確保出来ない。主な理由は5つ
  - (1) lock thrashing
  - (2) preemptive aborts
  - (3) deadlocks
  - (4) timestamp allocation
  - (5) memory-to-memory copy

- (1) lock thrasing
  - waiting-basedのアルゴリズム(deadlock detectionなど)はスケールしない
  - NO_WAITなどは、aborts rateとパフォーマンスのバランスが大事

- アルゴリズムごとのボトルネック
  - 2PL
    - DL_DETECT: 競合が少ない場合はスケールするが、lock thrashingが起きやすい
    - NO_WAIT: スケールさせるようにするとabort rateが上がる
    - WAIT_DIE: lock thrasingとtimestampがボトルネック
      - // timetstampを見るのは、txの実行時間を比較するため?
  - T/O: 共通して、timestamp allocationはボトルネックになりやすい
    - TIMESTAMP: 各スレッドのprivate spaceにメモリコピーするオーバヘッドが大きい
    - MVCC: timestamp allocation
    - OCC: private spaceへのメモリコピー, abort costが高い
    - H-STORE: partitioned workloadには最適

- ただ、想定するworkloadによっては、scalabilityが確保できる場合がある
  - DBMS側で複数のアルゴリズムを切り替えられるようにするという対策も一案
    - 競合が少ない場合 -> DL_DETECT
    - 競合が多い場合 -> NO_WAIT, T/O-based
  - MySQLは、DL_DETECT + MVCCのhybrid approach  // PostgreSQLも?

- CPU側を改善することで解決できないか
  - T/Oは、timestamp allocationがボトルネック => HW clock
  - TIMESTAMP/OCCは、memory copyがボトルネック => アクセラレータ, Memory architecture
    - ソフトウェア的にmallocを独自に改善して、memory poolを作成して、対応した
    - ただ、コア数が巨大になると、扱いが困難になるので、decentralized or hierarchicalなメモリコントローラが必要になるのではないか
      - // イメージがついていない。共有メモリ上でのロックがボトルネックになるってこと?

- distributed DBMSはスケールするが、transactionを実現する場合には、single nodeの方が良い
  - node間のコミュニケーションより、スレッド間のコミュニケーションの方が速い
  - CPU処理時間と比較して、NW通信はかなり遅い
  - 理想的には、distributed DBMSとsingle node with multi-threadedの2階層になるとよい


## future work

- アルゴリズムの改善だけでは対応できない可能性があるため、HWを含めて改善したい
- concurrency control以外のloggingやindexについてもscalabilityの実現を目指したい

## acknowledgements

- Intel ScienceとTechnology Center for Big Dataで実施  // IntelだからHWレベルに提言している?

## conclusion

- concurrency controlに焦点を当てて、many core/CPUのときのスケーラビリティについて確認
  - 全workloadでscaleするアルゴリズムはない
    - 2PC: core数が少ない, short transaction, 競合が少ない場合は良い
    - T/O: 2PCを比較するとlong transaction, 競合が多少多くなってもよい

