---
layout: post
title: "一台構成でJujuからCephをデプロイ"
date: 2016-03-14 00:00:00 +0900
lang: ja
---

家で試せてかつ実用的なサービスとなると、やはりストレージではないか、とネットをウロウロしていたところ一台構成でCephをセットアップされた方がいらっしゃるようなので、試してみました。

## 参考サイト

- http://ceph.com/dev-notes/deploying-ceph-with-juju/
- http://todotani.cocolog-nifty.com/blog/2016/02/ceph-e274.html
- https://jujucharms.com/ceph

## Cephのセットアップ

### 下準備

下記コマンドで**fsid**を取得

```bash
sudo apt-get install uuid
uuid
```

下記コマンドで**monitor-secret**を取得

```bash
sudo apt-get install ceph-common
ceph-authtool /dev/stdout --name=mon. --gen-key
```

### Ceph

下記ファイルにそれぞれをコピペ。タブ使えないのでスペースでやること

```bash
vi ~/.juju/cepy.yaml
```

```yaml
ceph:
  fsid: 1ef7d4f4-e990-11e5-801a-ef4zzzzzzzzz
  monitor-secret: AQADKOZW0N1FNBAAxxxxxxxx==
```

```bash
juju deploy -n 3 --config cepy.yaml ceph
```

### Ceph-OSD

OSD用の場所

```bash
tt:~/.juju$ ls /data
osd0 osd1 osd2
```

yamlファイルを編集

```bash
vi ceph.yaml
```

```yaml
ceph-osd:
  osd-devices: /data/osd0 /data/osd1 /data/osd2
```

デプロイ

```bash
juju deploy -n 3 --config ceph.yaml ceph-osd
juju add-relation ceph-osd ceph
```

### Ceph-Rados

```bash
juju deploy ceph-radosgw
juju add-relation ceph-radosgw ceph
juju expose ceph-radosgw
```

## ここで問題が

agent is lostというエラーが出ました

```bash
$juju-status
    units:
      ceph-osd/0:
        workload-status:
          current: unknown
          message: agent is lost, sorry! See 'juju status-history ceph-osd/0'
          since: 15 Mar 2016 09:10:25+09:00
        agent-status:
          current: lost
```

status-historyには特に何も・・・

```bash
$ juju status-history ceph-osd/0
TIME                          TYPE        STATUS       MESSAGE                   
15 Mar 2016 08:45:25+09:00    agent       executing    running update-status hook
15 Mar 2016 08:45:25+09:00    workload    active       Unit is ready (3 OSD)     
15 Mar 2016 08:45:27+09:00    agent       idle
```

health detailで見ると、monclient用のkeyringが無いと出ています

```bash
$ juju ssh ceph-osd/0 sudo ceph health detail
2016-03-15 09:15:51.574749 7f25c5849700 -1 monclient(hunting): ERROR: missing keyring, cannot use cephx for authentication
2016-03-15 09:15:51.574750 7f25c5849700  0 librados: client.admin initialization error (2) No such file or directory
$ juju ssh ceph/0 sudo ls /etc/ceph/
ceph.client.admin.keyring  ceph.conf  rbdmap
```

設定ファイルによるとkeyringの場所は下記

```bash
$ juju ssh ceph/0 sudo cat /etc/ceph/ceph.conf
[mon]
keyring = /var/lib/ceph/mon/$cluster-$id/keyring
[mds]
keyring = /var/lib/ceph/mds/$cluster-$id/keyring
[osd]
keyring = /var/lib/ceph/osd/$cluster-$id/keyring
```

実際おいてあるファイルは名前が違うような？

```bash
$ juju ssh ceph/0 sudo find /var/lib/ceph/
/var/lib/ceph/bootstrap-osd/ceph.keyring
/var/lib/ceph/bootstrap-mds/ceph.keyring
/var/lib/ceph/mon/ceph-wabuntu-local-machine-1/keyring
```

他のCephユニットを見る限り状態は同じなので、Ceph/０だけがエラーになっているのは何か別のところに理由があるのだろうか？

とりあえず再起動してみる

```bash
$ juju ssh ceph/0 sudo service ceph-osd-all restart
```

ceph -wを見るとHEALTH_OKに

```bash
$ juju ssh ceph/0 sudo ceph -w
Warning: Permanently added '10.0.3.223' (ECDSA) to the list of known hosts.
 cluster c0e9ce1e-e9c4-11e5-9d0b-9b717dc9f1f8
 health HEALTH_OK
 monmap e1: 3 mons at {wabuntu-local-machine-1=10.0.3.223:6789/0,wabuntu-local-machine-2=10.0.3.56:6789/0,wabuntu-local-machine-3=10.0.3.172:6789/0}, election epoch 4, quorum 0,1,2 wabuntu-local-machine-2,wabuntu-local-machine-3,wabuntu-local-machine-1
 osdmap e45: 9 osds: 9 up, 9 in
 pgmap v2273: 1020 pgs, 14 pools, 840 bytes data, 43 objects
 192 GB used, 745 GB / 988 GB avail
 1020 active+clean
```

しかしながら、まだ例のメッセージは出たまま

```
ceph-osd/0:
  workload-status:
    current: unknown
    message: agent is lost, sorry! See 'juju status-history ceph-osd/0'
    since: 15 Mar 2016 11:20:25+09:00
```

**結局しばらくほっといたら治ってました・・・**


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/14/%e4%b8%80%e5%8f%b0%e6%a7%8b%e6%88%90%e3%81%a7juju%e3%81%8b%e3%82%89ceph%e3%82%92%e3%83%87%e3%83%97%e3%83%ad%e3%82%a4/).*
