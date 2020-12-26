---
layout: post
title: Clock-SI Snapshot Isolation for Partitioned Data Stores Using Loosely Synchronized Clocks
tags:
- patent
- distributed database
---

## 概要

- PostgreSQL Wikiの[Scaleout Design](https://wiki.postgresql.org/wiki/Scaleout_Design)によるとClock-SIがMicrosoftによって[特許化](https://patents.google.com/patent/US8356007)されているかもとのこと

- 特許に関して、素人なので、技術的観点の類似度を簡単に調べてみるだけ

## まとめ

- 途中までしか読めていないが、サマリだけで、Clock-SIをそのまま特許化している印象

## abstract

- 特許の概要
  - 分散トランザクション技術
  - 分散データベース内の参加ノード間での同期を保証する
  - それぞれの参加ノードのlocal時刻を利用する
  - 分散tx内の参加者は、同一のcommit timestamp, logical read timeを利用することでそれぞれのlocal clocksの同期をとる
  - 分散commitは、2PCによって実施される
  - 各参加ノードから、commit時刻を収集する
  - heartbeatの仕組みを使って、ノード間でloose synchronizationは実施する
  - 各ノードはリモートtx実行時にその結果に加えて、"types of acess"の情報も付与する

- // preparingやcommitingなどの状態を意識した処理については触れられていない 
- // wal書き出しのタイミングについても触れられていない => logの話はdeteilで出てくる

## Distributed Transcation Management For Database Systems With MultiVersioning

- multiversioning前提
- 事前にlocal or globalのtxかを知らない前提
- all isolation levelをサポート
- global clockにアクセスしない
- distributed deadlockを発生しない

## Summary

- logical clockを用いて、version間で読むべきデータが決まる
  - PostgreSQLのようなmonotonically increasing counterも含む

- 全ノードで同一のcommit timestampを保証する

- 最適化手法
  - (1) batch的にcommitすることでオーバヘッドを削減する
  - (2) (結果を返す)ついでにローカル時刻の同期を取る
  - (3) piggybancking informationをやりとりし、早すぎるGCを防ぐ

## Detailed Description

- 実現すること
  - (1) local transactionはオーバヘッドなし
  - (2) local, globalともに全てのisolation levelをサポートする
  - (3) global clockなどのglobal nodeに頻繁にアクセスしない
  - (4) distributed deadlockによる停止を発生しない

- 実装例
  - local clock(やそれに類するlocal transaction counterなど）のみを用いたcommit protocol
    - local lockをグローバルtxのコミットの一部で同期を取る
    - 一例としては、同じcommit timestampを参加ノード間で保証する
  - heartbeatなどのメカニズムでそれぞれのlocal clockの同期を取る
  - txの結果やread/writeなどのタイプを返すことで、coordinatorがtxを追跡できるようにする

- 挫折) p5 line65まで
  - 次読むときは、embodimentとmethodを分けて考えると良さそう
  - embodimentは、一例っぽい。methodが抽象化した特許の技術と思われる

