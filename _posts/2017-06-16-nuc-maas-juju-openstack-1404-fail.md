---
layout: post
title: "NUCでMAAS+juju+OpenStack(14.04 失敗)"
date: 2017-06-16 00:00:00 +0900
lang: ja
---

## 概要

このポストは、NUC（Next Unit of Computing）マシン上でMAAS、Juju、OpenStackを設定しようとした試みについて記録しています。著者は14.04のUbuntu Serverを使用し、最終的には失敗に終わったことを記述しています。

## ネットワーク設定

物理マシンは2つのネットワークインターフェースを使用します:

```
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.1.201
netmask 255.255.255.0
broadcast 192.168.1.255
gateway 192.168.1.1
dns-nameserver 192.168.1.1

# router
auto eth1
iface eth1 inet static
address 192.168.100.1
netmask 255.255.255.0
broadcast 192.168.100.255
```

## サブネット100（VM用）とのフォワーディング設定

IP フォワーディングを有効化:

```bash
# /etc/sysctl.conf での設定
net.ipv4.ip_forward=1

# NATルールの設定
sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/255.255.255.0 -j MASQUERADE
sudo iptables-save -c | sudo tee /etc/iptables.rules
```

iptablesルールを起動時に復元するスクリプト:

```bash
#!/bin/bash
iptables-restore < /etc/iptables.rules
exit 0
```

ファイルを実行可能にします:

```bash
sudo chmod +x /etc/network/if-pre-up.d/iptables
```

現在のiptablesルール確認:

```bash
iptables -nL -t nat
```

ルーティングテーブル確認:

```bash
route
```

**注記**: ホームゲートウェイのLAN側DHCPを無効化してKVMやMAAS側へのDHCPオファーを防止する必要があります。

## MAASインストール

### ロケール設定

MAASインストール前にロケール環境変数を設定:

```bash
sudo vi .bash_profile
export LC_ALL="en_US.UTF-8"

source .bash_profile
locale
```

### パッケージソース修正

aptエラーが発生する場合、ソースを変更:

```bash
sudo apt update
sudo vi /etc/apt/sources.list
# jp.archive.ubuntu.com => us.archive.ubuntu.com に変更
sudo apt install maas
```

### PostgreSQL接続エラー解決

以下のエラーが発生時の解決方法:

```
psql: could not connect to server: No such file or directory
Is the server running locally and accepting connections on Unix domain socket 
"/var/run/postgresql/.s.PGSQL.5432"?
```

解決策:

```bash
export LC_ALL="en_US.UTF-8"
sudo pg_createcluster 9.3 main --start
```

### MAASのインストールと設定

```bash
sudo apt install maas
sudo dpkg-reconfigure maas-cluster-controller
# http://192.168.100.1/MAAS
sudo dpkg-reconfigure maas-region-controller
# 192.168.100.1
```

管理者アカウント作成とAPIキー取得:

```bash
sudo maas-region-admin createadmin
# Username: ubuntu
# Password: (入力)
# Email: wabuntu@wabuntu.com

sudo maas-region-admin apikey --username=ubuntu > ./apikey
cat ./apikey
# yp4zkRq29PeRJNX23R:mzEMs2ngCC6e7VG3GU:gGhhsHxxxxxxx

maas login ubuntu http://localhost/MAAS
maas list
# ubuntu http://localhost/MAAS/api/1.0/ yp4zkRq29PeRJNX23R:mzEMs2nxxxx
```

### リモートアクセス設定

別マシンからMAAS WEBにアクセス:

```bash
wabuntu@nuc:~$ sshuttle -r ubuntu@192.168.1.201 192.168.100.0/24
```

ブラウザでアクセス:

```
http://192.168.100.1/MAAS/
```

### MAAS WEB設定

- imageタブでLinuxイメージをインポート（14）
- Clusters -> Cluster Master -> eth1 -> edit ボタン
- IP、GWを192.168.100.1に、DHCPをオン、範囲を設定、名前をbr0に変更
- Zoneタブでmaasを追加

## KVM設定

### パッケージインストール

```bash
sudo apt install qemu-kvm libvirt0 libvirt-bin virt-manager bridge-utils
sudo wget http://ftp.riken.jp/Linux/ubuntu-releases/14.04/ubuntu-14.04-server-amd64.iso
```

### ブリッジ設定

```bash
sudo vi /etc/network/interfaces
```

```
auto eth1
iface eth1 inet manual

auto br0
iface br0 inet static
address 192.168.100.1
netmask 255.255.255.0
broadcast 192.168.100.255
gateway 192.168.100.1
dns-nameservers 192.168.100.1 192.168.1.1
bridge_ports eth1
bridge_stp off
```

ブリッジを起動:

```bash
sudo ifup br0
```

### テストVM作成

virt-managerでテストVM作成時のポイント:
- PXEブートを選択
- ブート順序でNICを最初に設定
- 画面をSpiceからVNCに変更
- MACアドレスをメモ

VM確認:

```bash
ubuntu@nuc1:/var/lib/libvirt/images$ virsh -c qemu+ssh://ubuntu@192.168.122.1/system list
```

## Commission試行

Nodesタブ、Add HardwareでMachineを選択して設定:
- Power type: Virsh
- Power ID: ubuntu14.04
- Power Address: qemu+ssh://ubuntu@192.168.1.201/system
- Password: ubuntu@192.168.1.201のパスワード
- MAC: ゲストVMのMAC

Commission実行後、MAASのWEB側でNodeがReadyになります。

## Juju用VM作成

virt-installを使用（virt-managerではデプロイ時に失敗):

```bash
NAME="juju"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=15,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=100,format=qcow2 \
--network bridge=br0,mac=52:54:00:63:7e:7a,model=virtio
```

## タグ設定

タグ情報確認:

```bash
ubuntu@nuc1:~$ maas ubuntu tags list
```

新しいタグ作成:

```bash
ubuntu@nuc1:~$ maas ubuntu tags new name=bootstrap
ubuntu@nuc1:~$ maas ubuntu tag nodes virtual | grep hostname
```

ホスト名からシステムIDを取得:

```bash
ubuntu@nuc1:~$ maas ubuntu nodes list hostname=juju | grep system.id
# "system_id": "node-ac931ffe-509f-11e7-9e60-b8aeed7f0b46"
```

ブートストラップタグを付与:

```bash
ubuntu@nuc1:~$ maas ubuntu tag update-nodes bootstrap add=node-ac931ffe-509f-11e7-9e60-b8aeed7f0b46
```

## SSH公開鍵設定

デプロイに必要な公開鍵を生成:

```bash
ubuntu@nuc1:~$ ssh-keygen -t rsa
ubuntu@nuc1:~$ cat .ssh/id_rsa.pub
```

MAAS WEBの右上のユーザーアイコンからAdd SSH keyにペーストします。

## Jujuのブートストラップ

### Jujuパッケージインストール

```bash
ubuntu@nuc1:~$ sudo apt install juju juju-deployer charm-tools
```

### 初期設定

```bash
ubuntu@nuc1:~$ juju generate-config
ubuntu@nuc1:~$ ls -l ~/.juju/
# -rw------- 1 ubuntu ubuntu 20690 Jun 16 09:35 environments.yaml
# drwx------ 2 ubuntu ubuntu 4096 Jun 16 09:35 ssh
```

設定ファイルの編集:

```bash
ubuntu@nuc1:~$ vi ~/.juju/environments.yaml
```

```yaml
default: maas
maas:
 type: maas
 maas-server: 'http://192.168.100.1/MAAS/'
 maas-oauth: 'h5b3LpQFMVz3XhauQ2:aLyqbRbnJZgQbxxxxxx'
```

### ブートストラップ試行

```bash
ubuntu@nuc1:~$ juju status
# ERROR Unable to connect to environment "maas".
```

タグ指定でブートストラップ:

```bash
ubuntu@nuc1:~$ juju bootstrap --show-log --constraints tags=bootstrap
# 400 BAD REQUEST ({"distro_series": ["'xenial' is not a valid distro_series. 
# It should be one of: '', 'ubuntu/trusty'."]})
```

このバグは未解決のようです:

```
https://bugs.launchpad.net/maas/+bug/1537095
```

### ワークアラウンド試行

```bash
ubuntu@nuc1:~$ sudo apt install distro-info-data=0.18
ubuntu@nuc1:~$ distro-info --lts
# trusty
ubuntu@nuc1:~$ juju bootstrap --show-log --constraints tags=bootstrap --upload-tools --series trusty
```

MAAS バージョン:

```
ii maas 1.9.5+bzr4599-0ubuntu1~14.04.1 all
```

## 発生した問題

### イメージの不整合

MAASに16.04のイメージを追加してブートストラップしたところ、すべてが16.04に置き換わりました。Commissionは14で、Deployは16という不整合が発生。

### 環境削除の試行

```bash
ubuntu@nuc1:~$ juju remove-unit juju/0
ubuntu@nuc1:~$ juju destroy-environment maas
ubuntu@nuc1:~$ juju resolved juju/0
ubuntu@nuc1:~$ juju destroy-machine --force 0
ubuntu@nuc1:~$ juju destroy-environment maas --force
```

これらすべてが失敗し、最後のコマンドでようやく環境を削除できました。

### デプロイメントの試行

```bash
ubuntu@nuc1:~$ juju bootstrap --show-log --constraints tags=bootstrap --upload-tools --series trusty
ubuntu@nuc1:~$ juju deploy --to=0 trusty/juju-gui
```

ブートストラップは成功（5-10分）しましたが、deployはallocatingのまま2時間以上進まず。

## 注記

### DHCP

DHCPが予期せず192.168.1.xサブネットに向かい、再起動後SSH接続が失敗。ホームゲートウェイのDHCPを無効化することで解決。

### イメージ同期エラー

ClustersのImages一覧がOut-of-Syncになり、Settingsで「Commission するものが何もない」エラーが発生。

ログエラー:

```
Jun 12 11:37:28 nuc1 maas.boot_image_download_service: [ERROR] Failed to download images: [Errno 1] Operation not permitted
```

このエラーは解決できず、再インストールが必要でした。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2017/06/16/nucでmaasjujuopenstack14-04-失敗/).*
