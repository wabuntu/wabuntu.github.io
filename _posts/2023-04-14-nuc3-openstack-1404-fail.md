---
layout: post
title: "NUC3台でOpenStack構築にちゃれんじ(14.04 失敗)"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

## 初期セットアップ

KVMとネットワーク設定から始めます。

```bash
$ sudo apt-get install kvm virt-manager libvirt-bin bridge-utils
```

ネットワークインターフェースの設定:

```
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.1.201
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 192.168.1.1
```

ブリッジ設定の調整:

```bash
ubuntu@nuc1:~$ sudo vi /etc/sysctl.conf
```

```
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
```

## MAASインストール時の問題と解決

MAASをインストール中にPostgreSQLの問題が発生しました。

```
psql: could not connect to server: No such file or directory
Is the server running locally and accepting
connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
```

解決方法は以下の通りです:

```bash
export LC_ALL="en_US.UTF-8"
sudo pg_createcluster 9.3 main --start
```

その後、再度MAASをインストール:

```bash
ubuntu@nuc1:~$ sudo apt install maas
```

## MAASクラスタコントローラーの設定

```bash
ubuntu@nuc1:~$ sudo dpkg-reconfigure maas-cluster-controller
```

ネットワークアドレスの確認:

```bash
ubuntu@nuc1:~$ sudo ip a
```

出力例:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
 link/ether b8:ae:ed:7f:0b:46 brd ff:ff:ff:ff:ff:ff
 inet 192.168.1.201/24 brd 192.168.1.255 scope global eth0
```

## 管理者アカウントの作成とログイン

```bash
ubuntu@nuc1:~$ sudo maas-region-admin createadmin  
Username: ubuntu  
Password:  
Again:  
Email: wabuntu@wabuntu.com
```

APIキーの取得:

```bash
ubuntu@nuc1:~$ sudo maas-region-admin apikey --username=ubuntu > ./apikey
ubuntu@nuc1:~$ maas login ubuntu http://localhost/MAAS
```

ログイン確認:

```bash
ubuntu@nuc1:~$ maas list  
ubuntu http://localhost/MAAS/api/1.0/ tGs5EF6eDbs3ggKE2E:JpGTnB84P4WyV9mdfA:xaqpykeS8t2Xw4y34pVrp4g5dbjYnMnk
```

## KVM環境構築

UbuntuサーバーISOをダウンロード:

```bash
ubuntu@nuc1:/var/lib/libvirt/images$ sudo wget http://ftp.riken.jp/Linux/ubuntu-releases/14.04/ubuntu-14.04-server-amd64.iso
```

## ブリッジネットワークの設定

外部接続用のブリッジを作成:

```
auto eth1  
iface eth1 inet manual  
auto br0  
iface br0 inet static  
address 10.0.0.1  
netmask 255.255.255.0  
gateway 10.0.0.1  
bridge_ports eth1
```

ブリッジの確認:

```bash
ubuntu@nuc1:~$ sudo brctl show  
bridge name bridge id STP enabled interfaces  
br0 8000.7403bd7f1c59 no eth1  
virbr0 8000.fe5400307852 yes vnet0
```

## 仮想マシンの作成

virtinstパッケージをインストール:

```bash
apt install virtinst
```

仮想マシン作成スクリプト:

```bash
NAME="nuc1-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--network bridge=virbr0 --pxe --boot network,hd,menu=on --graphics vnc \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=15,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=100,format=qcow2 \
--network bridge=br0
```

nuc2用:

```bash
NAME="nuc2-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--network bridge=virbr0 --pxe --boot network,hd,menu=on --graphics vnc \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=15,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=100,format=qcow2
```

nuc3用:

```bash
NAME="nuc3-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--network bridge=virbr0 --pxe --boot network,hd,menu=on --graphics vnc \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=15,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=100,format=qcow2
```

## ネットワークドライバの最適化

`~/.tmux_conf` の設定:

```
set -g mouse on
set -g terminal-overrides 'xterm*:smcup@:rmcup@'
```

## PXEブート時の診断

DHCPの状態確認:

```bash
ubuntu@nuc1:/var/lib/maas/dhcp$ sudo tcpdump -i virbr0 src or dst 192.168.122.1 -xX -vvv
```

TCPダンプ出力の抜粋:

```
15:59:36.876589 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
 192.168.122.1.bootps > 192.168.122.104.bootpc: [udp sum ok] BOOTP/DHCP, Reply, length 300, xid 0x458edb57, secs 14, Flags [none] (0x0000)
 Your-IP 192.168.122.104
 Server-IP 192.168.122.1
 Client-Ethernet-Address 52:54:00:1e:dc:87
 file "pxelinux.0"
```

pxelinuxファイルの場所確認:

```bash
ubuntu@nuc1:/var/lib/maas/dhcp$ sudo locate pxelinux.0  
/usr/lib/syslinux/gpxelinux.0  
/usr/lib/syslinux/pxelinux.0  
/var/lib/maas/boot-resources/snapshot-20170508-102541/pxelinux.0
```

MAAS設定ファイルの確認:

```bash
ubuntu@nuc1:/var/lib/maas/dhcp$ sudo cat /etc/maas/*.conf  
cluster_uuid: 3fcaed4d-93b1-4ffa-ad78-9e3a792b11fb  
maas_url: http://192.168.1.201/MAAS  
database_host: localhost  
database_name: maasdb  
database_pass: 3zCD7zxY0HHR  
database_user: maas
```

## OpenStack要件

Juju CharmスタックでOpenStackを構築するには、以下の要件があります:

- MAASでデプロイされた最低4台の物理マシンまたはVM
- メモリ最低8GB
- 2つのディスク（sda：OS、sdb：Ceph）
- eth0（プライベート）とeth1（パブリック）インターフェース

### ハードウェア仕様

- **nuc1**: i7-5557U CPU @ 3.10GHz x 4, 8GB, 256GB
- **nuc2**: i5-3427U CPU @ 1.80GHz x 4, 8GB, 256GB
- **nuc3**: i5-4250U CPU @ 1.30GHz x 4, 8GB, 256GB

## トラブルシューティング

### ロケール関連の問題

ロケール設定がPostgreSQLの初期化に影響していました:

```bash
export LC_ALL="en_US.UTF-8"
sudo pg_createcluster 9.3 main --start
```

### ネットワーク設定の再検討

重要な発見として、「KVM環境のサブネットとMAASのサブネットは同じである必要がある」ことが判明しました。DHCPとTFTPが正しく動作するためには、これらが同じネットワーク上に存在する必要があります。

ただし、MAASサーバー自体がKVMホスト上で実行されている場合、同じホスト内のサブネットの重複を避ける必要があります。推奨構成は:

1. ゲートウェイ・ルーターのIPアドレス（外部接続用）
2. KVMホスト専用のIPアドレス（DHCP配信用）
3. 別途のVM内にMAASサーバーを配置
4. VM管理用の独立したブリッジネットワーク

## 結論

このプロジェクトでは、複数のNUCサーバー上にOpenStack環境を構築することが目標でしたが、初期段階でのPostgreSQLとネットワーク設定の問題により、詳細な実装まで到達できませんでした。主な学習ポイントは、ロケール設定とネットワークトポロジーの重要性でした。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/nuc3%e5%8f%b0%e3%81%a7openstack%e6%a7%8b%e7%af%89%e3%81%ab%e3%81%a1%e3%82%83%e3%82%8c%e3%82%93%e3%81%9814-04-%e5%a4%b1%e6%95%97/).*
