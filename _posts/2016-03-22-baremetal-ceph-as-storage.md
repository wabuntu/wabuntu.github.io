---
layout: post
title: "ベアメタルでCephをただのストレージとして使ってみる"
date: 2016-03-22 00:00:00 +0900
lang: ja
---

## 下準備

今回**Ceph Cookbook**という本を買いまして、それの内容を見ながら進めました。ちなみに本よりも下記のリンクで最新の方法を確認したほうが無難です。

http://docs.ceph.com/docs/hammer/start/quick-rbd/

## マシン

割と一般家庭にあるようなレベルのPCを使いました

### ストレージマシン

- ホスト名: tt
- CPU: Core-i5
- Mem: 8G
- /dev/sdb: 128G SSD(OS用)
- /dev/sda: 2TB HDD(ストレージ用)
- Ubuntu Server 14.04(SSH Serverのみ入っている)

### 作業マシン

- Intel nuc
- 普通のUbuntu Desktop 14.04

## ストレージマシン側(tt)

### SSHのセットアップ

サーバーインストール時にSSH Serverを選択しておけばOK

### NTPのセットアップ

NTPが必要らしい・・・

```bash
sudo apt-get install ntp
sudo vi /etc/ntp.conf
#server ntp.ubuntu.com
sudo ntpq -p
```

### ユーザーのセットアップ

私は自分のUbuntuユーザー(wabuntu)を使いましたが、**要はパスなしSudoerが必要**なようです。両方のマシンに**共通のユーザーが必要**です（Sudoerはストレージ側だけで良い）。

```bash
sudo useradd -d /home/{username} -m {username}
sudo passwd {username}
echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
sudo chmod 0440 /etc/sudoers.d/{username}
```

## 作業マシン側(nuc)

今後操作はすべて作業マシン**から**行います。

なお、ceph-deployは**決してsudoやrootから行わず**、先ほどの専用ユーザーを使うこと。（実際やったけどとんでもないことになった）

```bash
ssh-keygen               #これも作業ユーザーで！(wabuntu)
ssh-copy-id wabuntu@tt   #ttはストレージマシン名
mkdir ceph-cluster
cd ceph-cluster/
ceph-deploy new tt       #/etc/hostsに記入済み
```

```bash
vi ceph.conf 
>>osd pool default size = 1
```

#ここにはCephOSDの数を入れるようだが、DISKが１個なのでこうしてみた（意味なかった）
#http://docs.ceph.com/docs/hammer/start/quick-ceph-deploy/#create-a-cluster

ここからttマシンに対してどんどん作業していきます

```bash
ceph-deploy install tt
ceph-deploy mon create-initial
ceph-deploy mon create tt
ceph-deploy gatherkeys tt      #実行すると自分のcephディレクトリ下にkeyringが落ちてくる
ls -la
>>\*.keyring
```

### ディスクの割り当て

```bash
ceph-deploy disk list tt              #私の場合sdaがそれ用ディスク（普通は違うので注意！）
#\[tt\]\[DEBUG \] /dev/sda :
ceph-deploy disk zap tt:sda            #クリーンアップ
ceph-deploy osd prepare tt:sda　　　　 #prepareをすると勝手にsda1とsda2ができた
ceph-deploy osd activate tt:/dev/sda1  #これで済。資料によってはcreateが必要と
あるが・・
```

## 仕上げ

```bash
ceph-deploy admin nuc
#adminを自分にDeployすることで、簡単にcephコマンドが使えるようになる

ceph -s
    cluster 0750e9bc-be75-44d5-bc06-c1c782d3b78e
     health HEALTH_OK
     monmap e1: 1 mons at {tt=192.168.1.100:6789/0}, election epoch 1, quorum 0 tt
     osdmap e11: 1 osds: 1 up, 1 in
      pgmap v21: 192 pgs, 3 pools, 0 bytes data, 0 objects
            39736 kB used, 1857 GB / 1857 GB avail
                 192 active+clean
```

## いろんなステータスを確認してみる

```bash
wabuntu@nuc:~/ceph-cluster$ ceph df
GLOBAL:
    SIZE      AVAIL     RAW USED     %RAW USED 
    1857G     1857G       39736k             0 
POOLS:
    NAME         ID     USED     %USED     MAX AVAIL     OBJECTS 
    data         0         0         0         1857G           0 
    metadata     1         0         0         1857G           0 
    rbd          2         0         0         1857G           0 

wabuntu@nuc:~/ceph-cluster$ ceph mon stat
e1: 1 mons at {tt=192.168.1.100:6789/0}, election epoch 1, quorum 0 tt

wabuntu@nuc:~/ceph-cluster$ ceph osd stat
     osdmap e11: 1 osds: 1 up, 1 in

wabuntu@nuc:~/ceph-cluster$ ceph pg stat
v21: 192 pgs: 192 active+clean; 0 bytes data, 39736 kB used, 1857 GB / 1857 GB avail

wabuntu@nuc:~/ceph-cluster$ ceph osd lspools
0 data,1 metadata,2 rbd,

wabuntu@nuc:~/ceph-cluster$ ceph osd tree
# id    weight    type name    up/down    reweight
-1    1.81    root default
-2    1.81        host tt
0    1.81            osd.0    up    1
```

## 実際にマウントしてみる（作業マシンから）

本来は"ceph-deploy install user-pc"が必要ですが、すでに"ceph-deploy admin"を実行しているのでここでは不要

```bash
wabuntu@nuc:~/ceph-cluster$ ls   #下記のKeyringを使用して操作します
ceph.bootstrap-mds.keyring  ceph.client.admin.keyring  ceph.log
ceph.bootstrap-osd.keyring  ceph.conf                  ceph.mon.keyring

#名前はfooで作成します
wabuntu@nuc:~/ceph-cluster$ rbd create foo --size 4096   #4GBの領域を作成
wabuntu@nuc:~/ceph-cluster$ sudo rbd map foo --pool rbd --name client.admin

#ファイルシステムを作成
wabuntu@nuc:~/ceph-cluster$ sudo mkfs.ext4 /dev/rbd/rbd/foo 

#マウントすると・・書き込みが可能！
wabuntu@nuc:~/ceph-cluster$ sudo mount /dev/rbd/rbd/foo /ceph/
wabuntu@nuc:~/ceph-cluster$ sudo chmod o+w /ceph
wabuntu@nuc:~/ceph-cluster$ vi /ceph/test.txt

#情報を確認
wabuntu@nuc:~/ceph-cluster$ df -h
/dev/rbd1                3.9G  8.0M  3.6G   1% /ceph

wabuntu@nuc:~/ceph-cluster$ rbd info --image foo
rbd image 'foo':
    size 4096 MB in 1024 objects
    order 22 (4096 kB objects)
    block_name_prefix: rb.0.5e36.74b0dc51
    format: 1

wabuntu@nuc:~/ceph-cluster$ rbd showmapped
id pool image snap device    
1  rbd  foo   -    /dev/rbd1 

#rbdによるベンチマーク
wabuntu@nuc:~/ceph-cluster$ rbd bench-write foo
bench-write  io_size 4096 io_threads 16 bytes 1073741824 pattern seq
......
elapsed:   222  ops:   132574  ops/sec:   595.21  bytes/sec: 4820692.25

#ddによるCephのベンチマーク
@nuc:~/ceph-cluster$ dd if=/dev/zero of=/ceph/bench bs=1M count=1024 oflag=direct
1073741824 バイト (1.1 GB) コピーされました、 46.9435 秒、 22.9 MB/秒

#ddによるSSDのベンチマーク
@nuc:~$ dd if=/dev/zero of=/tmp/bench bs=1M count=1024 oflag=direct
1073741824 バイト (1.1 GB) コピーされました、 6.87052 秒、 156 MB/秒
```

## Ceph余談

Ceph Cookbookに書かれている内容で気になったポイントをいくつか

- プロダクション環境では、仮想化したCephを使うことはおすすめされてない（そうです）。

### ３つのタイプのストレージ

- ブロックストレージ（RBD）
- オブジェクトストレージ（RADOS gateway）
- ファイルストレージ（CephFS）

### Cephのキーアイテム

- **OSD**: １つのOSDが１つの物理ディスクに対応。つまり**ディスク数＝OSD数**
- **MON**: いわゆるモニターでOSD, MON, PG, CRUSHという４つのマップを管理。おそらく１マシンに１個
- **RBD**: （RADOS Block Device）現在はCeph block deviceと呼ばれている
- **RADOS gateway**: オブジェクトストレージをREST APIで提供
- **MDS**: **CephFSのためだけに必要**。つまりブロックデバイスやRADOS gatewayには不要

※結構今まで思っていたのと違いましたが、本にはそう書いてありました・・・よ？

### 今回はブロックストレージを使用

下記の理由と、自宅用のDesktopを有効活用したいという理由から、今回ブロックストレージ（RBD)を使うことにしました。

- **ファイルストレージ**: カーネル２．６．３４からサポートされているものの、fsckや複数MDS、スナップショットなどが**準備中らしい**
- **オブジェクトストレージ**: REST APIで操作するのはキツい

ちなみにNFSマウントやFuse、あとWindowsからマウントするためのソフトなどもあるにはあるそうです。

### Ceph Block Deviceはどんなものか

- ブロックデータをCephクラスター内の複数のOSDに分配できる（つまりウチの１台HDDでは・・・）
- フルとインクリメンタルのスナップショットが可能
- シンプロビジョニングが可能
- Copy on Writeでクローンが可能
- ダイナミックリサイズが可能
- OpenstackのCinderやGlanceにも使える

## まったく無関係の利用者PCで新たにCephをマウントしてみる

利用者PCに新しいユーザーを追加（これも要はパスなしSudoerのためみたい。既存ユーザーをそうしてもいいかも）

```bash
$ sudo useradd -d /home/ceph -m ceph
$ sudo vi /etc/sudoers.d/ceph
$ sudo chmod 0440 /etc/sudoers.d/ceph
```

以下は管理マシンでの作業

```bash
wabuntu@nuc:~/ceph-cluster$ ssh-copy-id ceph@192.168.1.254
wabuntu@nuc:~/ceph-cluster$ ceph-deploy install 192.168.1.254  #まずはインストール
wabuntu@nuc:~/ceph-cluster$ ceph-deploy --username ceph push 192.168.1.254 #ceph.confをコピー
wabuntu@nuc:~/ceph-cluster$ ceph-deploy --username ceph admin 192.168.1.254 #keyringをコピー
```

この作業をすると、利用者PC側にconfとkeyringがコピーされ、利用可能になる

```bash
$ ls /etc/ceph
ceph.client.admin.keyring  ceph.conf  rbdmap
$ ceph -s
    cluster 0750e9bc-be75-44d5-bc06-c1c782d3b78e
     health HEALTH_OK
```

後は前回と同じように、プール作成、ファイルシステム作成、マウント

```bash
$ rbd create cordata --size 500000
$ sudo rbd map cordata --pool rbd --name client.admin
$ sudo mkfs.ext4 /dev/rbd/rbd/cordata
$ sudo mkdir /ceph
$ sudo mount /dev/rbd/rbd/cordata /ceph
$ df -h
/dev/rbd0                   481G   70M  456G   1% /ceph
```

感想：正直NASのようにして使うには手順が複雑・・・・Ceph-fsであれば、Ceph-dokanなんていうWindowsのソフトまであるようだ。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/22/%e3%83%99%e3%82%a2%e3%83%a1%e3%82%bf%e3%83%ab%e3%81%a7ceph%e3%82%92%e3%81%9f%e3%81%a0%e3%81%ae%e3%82%b9%e3%83%88%e3%83%ac%e3%83%bc%e3%82%b8%e3%81%a8%e3%81%97%e3%81%a6%e4%bd%bf%e3%81%a3%e3%81%a6/).*
