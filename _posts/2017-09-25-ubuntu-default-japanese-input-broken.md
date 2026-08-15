---
layout: post
title: "なんかubuntuデフォの日本語効かなくなってね？"
date: 2017-09-25 00:00:00 +0900
lang: ja
---

よく分からないのですが、普段はできていた気がする日本語入力が新規インストール（Desktop１６）でできなくなってました。

基本ibus-mozcを入れれば直るようです。

```shell
sudo apt install ibus-mozc
killall ibus-daemon
ibus-daemon -d -x &
```

あとは言語サポート、テキスト入力などをお好みで。

インストール後は再起動でもいいし、上記のようにkillしてもいいです。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2017/09/25/なんかubuntuデフォの日本語効かなくなってね？/).*
