---
layout: post
title: "Googleに特定の検索結果が出てきたときにポップアップ"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

このブログ投稿では、Google検索の結果をモニタリングし、特定のフレーズが見つかったときにデスクトップ通知を表示する方法を紹介しています。

## コード例

```shell
$ res=`googler --np -n 1 "フレーズ"`
$ if [ -s "${res}" ]; then notify-send "Google" ${res}; fi;
```

## ツール説明

**googler** — Google検索を実行するコマンドラインツール
- `--np` : 入力を待たずに終了する
- `-n 1` : 最初の1件の検索結果のみ表示

**notify-send** — デスクトップにポップアップ通知を表示するツール

## Cronでの定期実行例

```shell
0 3 * * * res=`googler --np -n 1 "フレーズ"`; if [ -s "${res}" ]; then notify-send "Google" ${res}; fi;
```

上記は3時間ごとに実行されます。

## HTMLフォーマットについての注記

HTMLで色付けも可能ですが、Ubuntuではシステムカラーが優先されるため機能しないとのこと：

```shell
notify-send -u critical "title" "<font color="red">contents of message</font>"
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/google%e3%81%ab%e7%89%b9%e5%ae%9a%e3%81%ae%e6%a4%9c%e7%b4%a2%e7%b5%90%e6%9e%9c%e3%81%8c%e5%87%ba%e3%81%a6%e3%81%8d%e3%81%9f%e3%81%a8%e3%81%8d%e3%81%ab%e3%83%9d%e3%83%83%e3%83%97%e3%82%a2%e3%83%83/).*
