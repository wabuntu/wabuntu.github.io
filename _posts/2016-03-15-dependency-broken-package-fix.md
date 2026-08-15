---
layout: post
title: "依存関係や壊れたパッケージの対処"
date: 2016-03-15 00:00:00 +0900
lang: ja
---

こんなん昔からあったっけ・・・

```bash
sudo apt-get install -f
```

ちなみに引数なしの-fはどういう意味かというと

> -f, --fix-broken
>        Fix; attempt to correct a system with broken dependencies in
>        place. This option, when used with install/remove, can omit any
>        packages to permit APT to deduce a likely solution.

-fはforceと勘違いしてただけかも。

あとはおなじみのこれ

```bash
sudo dpkg --configure -a
sudo dpkg-reconfigure xxx
```

## 参考サイト

http://qiita.com/ktyubeshi/items/d76e2f46c67c46163760


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/15/%e4%be%9d%e5%ad%98%e9%96%a2%e4%bf%82%e3%82%84%e5%a3%8a%e3%82%8c%e3%81%9f%e3%83%91%e3%83%83%e3%82%b1%e3%83%bc%e3%82%b8%e3%81%ae%e5%af%be%e5%87%a6/).*
