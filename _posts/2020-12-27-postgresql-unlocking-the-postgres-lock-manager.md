---
layout: post
title: Unlocking the Postgres Lock Manager
tags:
- postgresql
- lock
---


## Unlocking the Postgres Lock Manager

https://momjian.us/main/writings/pgsql/locking.pdf

## メモ

- vxid: 2/13, transactionid: 675
  - 2: backend id（最大でmax connections)
  - 13: backend内で実施されたtx番号（シリアルに増加?）
  - 675: データベースクラスタのグローバルなxid

- read only qeuryは、virtual xidだけ、インクリメント

- RowExclusiveLockは、何個のrowを変更しても1つだけ
  - row自身にlock情報は保存される。共有メモリの消費を抑えるため

- 同一行を更新しようとしたとき
  - 最初のプロセス: 特別なロックなどなし
  - 2番目のロック
    - tupleのロックを取得し、最初のプロセスのxidをロック待ちする  // rowから判断?
  - 3番目以降のロック
    - tupleロックで待たされる // tupleのロックが取れないのでxidのロックに進まない
