---
layout: post
title: "gdbで0x0000000000000000 in ?? () が出るとき"
date: 2017-03-08 00:00:00 +0900
lang: ja
---

大抵のケースでは、\*-dbgというパッケージが足りない。探してもわからない時は、

```
(gdb) info sharedlibrary  
From To Syms Read Shared Object Library  
0x00007f0883620f00 0x00007f0883636723 Yes /usr/lib/libtotem_pg.so.5  
0x00007f0883417610 0x00007f0883417978 Yes /usr/lib/libcorosync_common.so.4  
0x00007f08831bb430 0x00007f08831cb956 Yes (*) /usr/lib/libqb.so.0
```

上記の(\*)がdebug infoが欠けてるライブラリらしい


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2017/03/08/gdbで0x0000000000000000-in-が出るとき/).*
