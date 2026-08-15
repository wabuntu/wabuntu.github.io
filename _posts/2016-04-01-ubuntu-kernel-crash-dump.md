---
layout: post
title: "ubuntuカーネルのcrash dump"
date: 2016-04-01 00:00:00 +0900
lang: ja
---

参考: https://wiki.ubuntu.com/Kernel/CrashdumpRecipe

## インストール

```bash
sudo apt-get install linux-crashdump
```

vmlinuxファイルも必要です。

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ECDCAD72428D7C01
sudo apt-get update
sudo apt-get install linux-image-$(uname -r)-dbgsym
```

インストール後、デバッグカーネルは以下の場所に配置されます:

```
/usr/lib/debug/boot
```

## crash ツールの使用

```bash
crash <debug kernel> <crash dump>
```

ただし、このツールには相性の問題が存在する可能性があります。

## ダンプの調査

```bash
apport-retrace --stdout --rebuild-package-info /var/crash/linux-image\*.crash
```

## テスト用パニック発生方法

仮のパニックを発生させるには、以下のコマンドを使用します:

```bash
echo 1 > /proc/sys/kernel/hung_task_panic          # panic when hung task is detected
echo 1 > /proc/sys/kernel/panic_on_io_nmi          # panic on NMIs from I/O
echo 1 > /proc/sys/kernel/panic_on_oops            # panic on oops or kernel bug detection
echo 1 > /proc/sys/kernel/panic_on_unrecovered_nmi # panic on NMIs from memory or unknown
echo 1 > /proc/sys/kernel/softlockup_panic         # panic when soft lockups are detected
echo 1 > /proc/sys/vm/panic_on_oom                 # panic when out-of-memory happens
```

まだ実装していないため、実際に試してから詳細を記載予定です。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/04/01/ubuntu%e3%82%ab%e3%83%bc%e3%83%8d%e3%83%ab%e3%81%aecrash-dump/).*
