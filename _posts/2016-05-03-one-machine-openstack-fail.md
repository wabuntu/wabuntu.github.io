---
layout: post
title: "マシン１台でOpenStack（失敗）"
date: 2016-05-03 00:00:00 +0900
lang: ja
---

OpenStackを初心者が試すにはMAASでつまづきやすいため、openstack-installを使うことにしました。公式では、単一マシンでの実行には8CPU、12GBメモリ、100GB以上のSSDが必要とされています。使用したPCの仕様は以下の通りです。

* CPU：i5-4460、4コア4スレッド
* メモリ：16GB
* SSD：128GB
* OS：ubuntu-server 14.04.4
* NIC：オンボード＋USB-LAN（オンボードに固定IP 192.168.1.101を設定、USB-LANは無設定）

Webインターフェース用にラップトップも使用しています。Ubuntu 16.04での試行時はopenstackをインストール後、openstackコマンドが見つからなかったため断念。14.04ではシンボリックリンク構成のようです。

参考サイト：
* https://help.ubuntu.com/lts/clouddocs/installer/en/single-install.html
* https://wiki.ubuntu.com/OpenStack/Installer/debugging/single-install

## インストール

```bash
$ sudo apt-add-repository ppa:cloud-installer/stable
$ sudo apt-get update
$ sudo apt-get install openstack
$ sudo openstack-install
```

非常にシンプルで、上記実行後1時間ほど待つだけです。

![Screenshot from 2016-05-03 11:41:38](https://wabuntu.wordpress.com/wp-content/uploads/2016/05/screenshot-from-2016-05-03-114138.png?w=537&h=412)

OpenStack Dashboardが緑になりURIが表示されれば、Horizon（Webインターフェース）へログイン可能です。サブネットが異なるため、ラップトップから以下を実行してルーティングします：

```bash
laptop$ sshuttle -r wabuntu@192.168.1.101 10.0.4.0/24
```

192.168.1.101はオンボードLANの固定IP、10.0.4.0はダッシュボードのサブネットです。

その後、https://10.0.4.177/horizonに接続し、デフォルトユーザー「ubuntu」でログインできます（パスワードはURIに表示）。インストール後のネットワーク構成は以下の通りです：

```
wabuntu@tt:~$ ip a
1: lo: 
2: em1: inet 192.168.1.101/24 brd 192.168.1.255 scope global em1
3: eth0: 
4: lxcbr0: inet 10.0.3.1/24 brd 10.0.3.255 scope global lxcbr0
5: virbr0: inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
6: uoibr0: inet 10.100.1.1/24 brd 10.100.1.255 scope global uoibr0
7: uoibr0-nic: 
9: vethWSDRNN@if8:
```

コンソールにKeystoneエラーが出ており、そのためWebログインできません。いくつかパッケージが不足しているか、初期段階でubuntuユーザー存在が前提のようです。対処中：

```bash
export JUJU_HOME=~/.cloud-install/juju
juju status
JUJU_HOME=~/.cloud-install/juju juju status
sudo apt-get install juju-local
sudo apt-get install uvtool-libvirt uvtool
grep ubuntu /etc/passwd
sudo mkdir /home/ubuntu
sudo adduser ubuntu
sudo chown ubuntu:ubuntu /home/ubuntu/
su - ubuntu
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/05/03/マシン１台でopenstack/).*
