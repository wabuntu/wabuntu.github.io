---
layout: post
title: "プライマリの GPT テーブルは破損しているようです"
date: 2018-06-13 00:00:00 +0900
lang: ja
---

外付けHDDが突然認識されなくなり、fdiskで変なメッセージが出るようになった。（まさかとは思うが18.04にアップグレードした影響では・・・）

```bash
$ sudo fdisk -l
```

プライマリの GPT テーブルは破損しているようです、しかしバックアップテーブルは大丈夫のようですので、そちらを使用します。

```
ディスク /dev/sdd: 1.8 TiB, 2000398934016 バイト, 3907029168 セクタ
単位: セクタ (1 * 512 = 512 バイト)
セクタサイズ (論理 / 物理): 512 バイト / 512 バイト
I/O サイズ (最小 / 推奨): 512 バイト / 512 バイト
ディスクラベルのタイプ: gpt
ディスク識別子: 7D3342CE-45B7-4BAE-BBF8-43454A48FD93

デバイス 開始位置 最後から セクタ サイズ タイプ
/dev/sdd1 10487808 3907029134 3896541327 1.8T Ceph OSD
/dev/sdd2 2048 10485760 10483713 5G Ceph Journal

パーティション情報の項目がディスクの順序と一致しません。
```

たしかに昔Cephとして使ったような記憶があるが・・・。gdiskを使ってみる。

```bash
$ sudo gdisk /dev/sdd
Command (? for help): p
```

```
Disk /dev/sdd: 3907029168 sectors, 1.8 TiB
Model: 001-1ER164 
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 7D3342CE-45B7-4BAE-BBF8-43454A48FD93
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 253879390758596
Partitions will be aligned on 2048-sector boundaries
Total free space is 253875483733523 sectors (115.4 PiB)

Number Start (sector) End (sector) Size Code Name
 1 10487808 3907029134 1.8 TiB F800 ceph data
 2 2048 10485760 5.0 GiB F802 ceph journal
```

```bash
Recovery/transformation command (? for help): w
```

```
Caution! Secondary header was placed beyond the disk's limits! Moving the
header, but other problems may occur!

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sdd.
The operation has completed successfully.
```

世間ではgdiskで内容確認してwするだけで治った例もあるようだが、さすがにCephは間違いみたいでマウントできない。

```bash
$ sudo fdisk /dev/sdd

コマンド (m でヘルプ): p
```

```
ディスク /dev/sdd: 1.8 TiB, 2000398934016 バイト, 3907029168 セクタ
単位: セクタ (1 * 512 = 512 バイト)
セクタサイズ (論理 / 物理): 512 バイト / 512 バイト
I/O サイズ (最小 / 推奨): 512 バイト / 512 バイト
ディスクラベルのタイプ: gpt
ディスク識別子: 7D3342CE-45B7-4BAE-BBF8-43454A48FD93

デバイス 開始位置 最後から セクタ サイズ タイプ
/dev/sdd1 10487808 3907029134 3896541327 1.8T Ceph OSD
/dev/sdd2 2048 10485760 10483713 5G Ceph Journal

パーティション情報の項目がディスクの順序と一致しません。

コマンド (m でヘルプ): t
```

```
パーティション番号 (1,2, 既定値 2): 1

パーティションのタイプ (L で利用可能なタイプを一覧表示します): 20

パーティションのタイプを 'Ceph OSD' から 'Linux filesystem' に変更しました。

コマンド (m でヘルプ): t
パーティション番号 (1,2, 既定値 2): 
パーティションのタイプ (L で利用可能なタイプを一覧表示します): 20

パーティションのタイプを 'Ceph Journal' から 'Linux filesystem' に変更しました。

コマンド (m でヘルプ): w
パーティション情報が変更されました。
ioctl() を呼び出してパーティション情報を再読み込みします。
ディスクを同期しています。
```

fdisk的にまだ何かおかしいが、無事マウントできてデータも無事なようだ・・・。良かった。

```bash
$ sudo fdisk -l
```

```
ディスク /dev/sdd: 1.8 TiB, 2000398934016 バイト, 3907029168 セクタ
単位: セクタ (1 * 512 = 512 バイト)
セクタサイズ (論理 / 物理): 512 バイト / 512 バイト
I/O サイズ (最小 / 推奨): 512 バイト / 512 バイト
ディスクラベルのタイプ: gpt
ディスク識別子: 7D3342CE-45B7-4BAE-BBF8-43454A48FD93

デバイス 開始位置 最後から セクタ サイズ タイプ
/dev/sdd1 10487808 3907029134 3896541327 1.8T Linux ファイルシステム
/dev/sdd2 2048 10485760 10483713 5G Linux ファイルシステム

パーティション情報の項目がディスクの順序と一致しません。
```

怖いのでデータ退避させて、パーティション作り直そう・・・。

```bash
$ sudo mount /dev/sdd1 /mnt/
$ ls /mnt
archive
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2018/06/13/%e3%83%97%e3%83%a9%e3%82%a4%e3%83%9e%e3%83%aa%e3%81%ae-gpt-%e3%83%86%e3%83%bc%e3%83%96%e3%83%ab%e3%81%af%e7%a0%b4%e6%90%8d%e3%81%97%e3%81%a6%e3%81%84%e3%82%8b%e3%82%88%e3%81%86%e3%81%a7%e3%81%99/).*
