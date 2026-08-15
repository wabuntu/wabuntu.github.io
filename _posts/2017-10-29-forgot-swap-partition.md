---
layout: post
title: "swap領域作り忘れた"
date: 2017-10-29 00:00:00 +0900
lang: ja
---

もうメモリサイズ十分だし、ディスクもM.2だから速いしSWAPいらないだろうと思ったけど、なぜかこれないとChromeが固まったりするので作ることにしました。パーティションで無くてもファイルとして作成可能です。

```shell
# dd if=/dev/zero of=/swap bs=1M count=8000
8000+0 レコード入力
8000+0 レコード出力
8388608000 bytes (8.4 GB, 7.8 GiB) copied, 17.822 s, 471 MB/s
```

```shell
# mkswap /swap
Setting up swapspace version 1, size = 7.8 GiB (8388603904 bytes)
ラベルはありません, UUID=dd5a00e0-d3d6-4754-aeef-xxxx
```

```shell
# swapon /swap 
swapon: /swap: パーミッション 0644 は安全な値ではありません。 0600 をお勧めします。
```

```shell
# chmod 600 /swap
```

```
# tail /etc/fstab 
/swap swap swap defaults 0 0
/swap swap swap defaults 0 0
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2017/10/29/swap%e9%a0%98%e5%9f%9f%e4%bd%9c%e3%82%8a%e5%bf%98%e3%82%8c%e3%81%9f/).*
