---
layout: post
title: "Ubuntu 16.04でMAASとjuju（失敗）"
date: 2016-05-01 00:00:00 +0900
lang: ja
---

OpenStack勉強したいのに・・・・

- そもそもインストールでしくじる
- VMWareとかXenとかと概念が全く違う
- コンポーネントが理解しにくい
- PC一台じゃ試して遊ぶこともできないみたい

そんな私のための試行錯誤の記録です。今回下記の３台のマシンを使います。見た通りの貧乏構成ですが、３台使います。

- **ラップトップ**：　ブラウザが入ってれば何でも可。ここから作業する。
  - 10万円ぐらいの普通のラップトップ
- **マシンA(cor)**：　maasを入れる
  - CPU:i5-3330, MEM:8G, SSD:128G
  - USB-LAN
- **マシンB(tt)**：  maasによってコントロールされるマシン
  - i5
- NETGEARの５ポートスイッチ

## 小咄

ここでOpenStackについてちょっと。

- いろんな役割を持ったコンポーネントが２０個以上ある
- 相当数の物理マシンを持っていない限り、結局はVMになる(maas,juju以外)
- 結構VMでできちゃうし、やってる人は多い
- VMは今の所KVMが主流、lxdが次第にメジャーになってるが今回はKVMで
- MAASってのはOpenStackとは別で、Ubuntuが提供してる物理マシン管理ツール。「PC起動！とかVM作成！」とかができる。KVMのインスタンスもニセ物理マシンとして管理できる（今回はこれ）
- jujuってのも上記同様、これは「ApacheとMySQL一発でデプロイ！」とかするためのツール。

## マシンAでの作業

### Ubuntu16.04をインストール

1. 一般的なサーバーとしてインストール、LVMなし。取り急ぎDHCPで、追加パッケージはSSH。
2. ホームディレクトリは暗号化してはいかん、というのが他であったので、そこだけはしない。

### KVMをインストール

```shell
sudo apt-get install kvm virt-manager libvirt-bin bridge-utils
```

インストールするとネットワークは自動で下記のように。２つ追加されている。

```
wabuntu@cor:/var/lib/libvirt/images$ ip a | grep -e "^\[0-9\]"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
3: enx7403bd7f1c59: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
5: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue master virbr0 state DOWN group default qlen 500
```

### VM用のイメージをダウンロードして保存

```shell
cd /var/lib/libvirt/images/
sudo wget http://releases.ubuntu.com/16.04/ubuntu-16.04-server-amd64.iso
```

どうもデフォルトの仮想ネットワークは自動でvirbr0になるようですが、追加したUSB-LAN上にbr0を作成します。家庭内LANにDHCPをばらまきたくない、でもvmから外部接続は必要、iptablesいじってみたらdhcpだけ通らなかったり・・・。「とりあえずOpenStackを試してみたい」という私のような初心者にはこのステップが苦痛。そもそもなんでブリッジが必要なのか分かってない。あと自動追加されたvirbr0*の存在意義が不明。

```
wabuntu@cor:~$ cat /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto enp3s0
iface enp3s0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    network 192.168.1.0
    broadcast 192.168.1.255
    gateway 192.168.1.1
    dns-nameservers 192.168.1.1
    dns-search home

auto enx7403bd7f1c59
iface enx7403bd7f1c59 inet manual

iface br0 inet static
address 10.0.0.1
netmask 255.255.255.0
dns-nameservers 192.168.1.1
bridge_ports enx7403bd7f1c59 
bridge_stp off
auto br0

wabuntu@cor:~$ brctl show
bridge name    bridge id        STP enabled    interfaces
br0        8000.7403bd7f1c59    no        enx7403bd7f1c59
virbr0        8000.52540061ec1c    yes        virbr0-nic
```

### MAASのインストール

```shell
sudo apt-get install maas
#メアドは別に嘘でもいいようです。後、今後maas-region-adminは無くなりmaas-regionになるそうな
sudo maas-region-admin createadmin --username=wabuntu --email=wabuntu@uso.com
sudo maas-region-admin apikey --username=wabuntu > ~/apikey
#上記apikeyをクリップボードにコピー
sudo dpkg-reconfigure maas-rack-controller
sudo dpkg-reconfigure maas-region-controller
#上記２つで10.0.0.1を設定（不要？）

maas login wabuntu http://localhost/MAAS
#apiキーを聞かれるので貼り付け（表示されないがOK）
maas list
#上記コマンドで何か出てくればOK
ssh-keygen -t rsa
cat .ssh/id_rsa.pub
#後ほどWEBで使うのでコピー
```

## ラップトップでの作業MAASのWeb

面倒なら下記でトンネルしてもいい

```shell
$ sshuttle -r wabuntu@192.168.1.100 10.0.0.0/24
```

1. ブラウザでhttp://マシンAのIP/MAASに接続
2. Imagesタブから、Import Imagesをクリック
3. Apply Changesをクリック
4. 右上のユーザー名からAccountを選び、＋Add SSH keyをクリック、先ほどのid_rsa.pubの内容をペースト
5. Networksのfabric0(br0の方)でDHCPを有効に。デフォルトゲートウェイを10.0.0.1にしないとPXE失敗するで。（初期状態は.254）

### 仮想マシン作成とコミッション

1. ラップトップにvirtmanagerをインストール
2. 接続を追加→SSHでマシンAのIPを設定
3. 新しい仮想マシンの作成→PXEブート→共有デバイス名を指定でｂｒ０（MACをコピーしておく）
4. Bootの順序をHDD->PXEにした状態で、電源OFFしておく（以後はMAASが管理するので）
5. Nodeを開いて、Powerタイプにqemu+ssh://wabuntu@10.0.0.1/systemを入れる
6. マシンAで"virsh -c qemu+ssh://wabuntu@10.0.0.1/system list –all"をテスト（せんでもいい）
7. power IDはvirtmanagerで作った仮想マシン名、MACもそこからコピペ
8. Take Action->CommissionでGo
9. virtmanagerで画面を見ていると、IP取得→OSインストール→ログイン画面→cloudinit等が走る、などが見て取れ、その後MAASのWebでノードがReadyになります

## マシンAでの作業

1. 下記のようなコマンドで情報が取れます

```shell
maas wabuntu tags read
maas wabuntu nodes read
```

### Jujuのインストール

下記で失敗・・・。断念。ちなみに16.04はjuju-2.0もついてくるので、下記のようにバージョンを指定すると古い方を使える。

```shell
$ juju-1 bootstrap
 WARNING ignoring environments.yaml: using bootstrap config in file "/home/wabuntu/.juju/environments/maas.jenv"
 ERROR cannot determine if environment is already bootstrapped.: could not access file 'd1989080-c49a-463a-8ad8-6833265dd9f3-provider-state': Requested map, got <nil>.
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/05/01/ubuntu-16-04%e3%81%a7maas%e3%81%a8juju%ef%bc%88%e5%a4%b1%e6%95%97%ef%bc%89/).*
