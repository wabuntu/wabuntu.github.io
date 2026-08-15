---
layout: post
title: "Ubuntuデスクトップにlogwatchを設定する"
date: 2016-04-10 00:00:00 +0900
lang: ja
---

## インストールと初期設定

```bash
$sudo apt-get install logwatch

$ sudo cp /usr/share/logwatch/default.conf/logwatch.conf /etc/logwatch/conf

$sudo mkdir /var/cache/logwatch
```

## メアドの設定

```bash
$ sudo cat /etc/aliases
root: aaa.bbb@ccc.com

$ sudo newaliases
```

## 実行

```bash
$sudo logwatch
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/04/10/ubuntu%e3%83%87%e3%82%b9%e3%82%af%e3%83%88%e3%83%83%e3%83%97%e3%81%ablogwatch%e3%82%92%e8%a8%ad%e5%ae%9a%e3%81%99%e3%82%8b/).*
