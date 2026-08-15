---
layout: post
title: "NUC一台のみでどれだけMAASで遊べるのか"
date: 2016-03-25 00:00:00 +0900
lang: ja
---

虎の子のNUCがすでに我が家ではDesktopとして確立してしまっているので、そのままUbuntu DesktopでMAAｓを使えるか試してみます。MAASは本来物理マシンを管理するものですが、予算の関係で今回は同じマシンにKVMを入れ、VMをMAASで管理できるか試してみます。

基本的にはここの内容に従います  
http://maas.ubuntu.com/docs/install.html

## 使用したマシン

- Intel NUC　Core i5 3427U　D53427RKE（total 82000円）
  - HDD: mSata 128GB SSD
  - MEM: 8G x 2
  - ミッキーケーブル（NUCに付属しないので別途必要）
  - OS: Ubuntu Desktop 14.04
- BUFFALO　USB3.0　LANアダプターLUA4-U3-AGT　1500円(追加NIC用)
- NETGEAR スイッチ GS105-500JPS 3000円（今回特に必要ではない）

## 事前準備

`ip a`で見ると追加のUSBLANがeth2になってました。なんにしてもデスクトップ右上のNetworkManagerがうるさいので、/network/interfacesに普通のサーバに設定するのと同じように固定IPを書きました。（追加すると黙る。デスクトップとして使ってるマシンなのでできればNetworkManagerは削除したくない）

## KVM

他に管理対象サーバーを持っているなら不要ですが、管理対象マシンとしてVMを作るためにKVMを入れます

参考：　http://qiita.com/TsutomuNakamura/items/e15d2c8c02586a7ae572

### インストール

```bash
sudo apt-get install kvm virt-manager libvirt-bin bridge-utils
lsmod | grep kvm
echo vhost_net >> /etc/modules
sudo service libvirt-bin start
ip a
```

```
#1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
#2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
#3: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
#4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
```

virbr0というのが自動でできます

### ブリッジ作成

KVMはブリッジが無いと使えないそうな。もとからあるeth0は一切いじりたくないので、USBLANであるeth2を使います

```bash
$ cat /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 192.168.1.50
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 192.168.1.1

auto eth2
iface eth2 inet manual #manualに
#ここから追加
iface br2 inet static
address 10.0.0.1
netmask 255.255.255.0
dns-nameservers 192.168.1.1　#これが無いとMAASでout-of-syncが出る・・のかも
bridge_ports eth2
bridge_stp off
auto br2
```

一回NICを再起動

```bash
wabuntu@nuc:~$ NIC="eth2";
wabuntu@nuc:~$ sudo ifdown ${NIC} && sudo ifup ${NIC}
wabuntu@nuc:~$ sudo ifup br2
```

下記をsysctlに追加

```bash
wabuntu@nuc:~$ sudo vi /etc/sysctl.conf 
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
wabuntu@nuc:~$ sudo sysctl -p
```

br2ができてこのようになります

```bash
wabuntu@nuc:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP group default qlen 1000
3: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br2 state UP group default qlen 1000
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
5: br2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 

wabuntu@nuc:~$ brctl show
bridge name    bridge id        STP enabled    interfaces
br2        8000.7403bd7fad84    no        eth2
virbr0        8000.000000000000    yes
```

### KVM用のイメージをダウンロードして下記にコピー

```bash
sudo cp ubuntu-14.04.4-server-amd64.iso /var/lib/libvirt/images/
```

### virt-managerを起動

Ubuntuマークを押して検索するか、コマンドラインから上記を起動して、お試しインスタンス（名前：testvm）を作ります。

- ネットワークブート（PXE)を選ぶ
- 最後の詳細オプションで、ホストデバイスにbr2を選ぶ

あとは普通です。起動して黒画面が出たらシャットダウンして放置。

### NAT

KVMインスタンスはeth2のブリッジ(br2)なので、ufwでNATを設定して、インスタンスからネットにつながるようにします  
参考：　http://d.hatena.ne.jp/ubuntu-nikki/20100921/1285077768

```bash
nuc:~$cat /etc/default/ufw
DEFAULT_FORWARD_POLICY="ACCEPT"

nuc:~$cat /etc/ufw/sysctl.conf
net/ipv4/ip_forward=1

nuc:~$head /etc/ufw/before.rules
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
COMMIT

nuc:~$sudo ufw disable && sudo ufw enable
```

**※このNATではシンプルすぎてダメ。インスタンス起動のときPXE失敗する。一度enableした後デプロイ前にdisableしました。**

※後日談 おそらく必要ない。MAASのWebに繋ぎたいときはsshuttle -rを使ったほうが楽。

## MAAS

### インストールと設定

```bash
sudo add-apt-repository ppa:maas/stable
sudo apt-get install maas
#MAAS apache2 bind9 dhcp python squid3 weechatなどが含まれています
```

普通はここでMAASのWEBのIPなど聞かれるはずですが、Desktopだからなのか聞いてきません。なので手動で設定します。

```bash
sudo dpkg-reconfigure maas-cluster-controller
#http://10.0.0.1/MAASを設定
sudo dpkg-reconfigure maas-region-controller
#PXE/Provisioning network => 10.0.0.1　こっちはいらないかも
```

### 管理者アカウントを作成

```bash
sudo maas-region-admin createadmin
#Username: wabuntu
#Password: 
#Again: 
#Email: wabuntu@wabuntu.com
```

### コマンドラインのアクセスを設定する

- http://10.0.0.1/MAASにアクセス
- Imagesタブになんか出るので、Imageを一つ登録してあげる
- 右上のユーザー名▼、Preferenceから、一番上のMAAS keysをコピーする。それを下記のコマンドで使用。

```bash
#方法はいろいろありますが、/api/2.0だと何故かできなかったので注意。/api/x.0を省略も可
wabuntu@nuc:~$ maas login wabuntu http://localhost/MAAS/api/1.0
#You are now logged in to the MAAS server at 成功するとこのように出ます
```

下記のコマンドで自分のユーザーが出ていればOK

```bash
wabuntu@nuc:~$ maas list
#wabuntu http://localhost/MAAS/api/1.0/ #z6QKkpafQZBmj2tFyV:6GkyM7Gze2Dcw2avrT:32xEqUPZpkxjxxxxxxxxxxx3
```

※後日談 maasユーザーのsshのpubkeyもこのページに追加（cluster masterがconnectedにならない場合はこれ）

### DHCP

公式にはmaas-rack-controllerが載っていますが、これはUbuntu Xenialからのようですので、普通のDHCPでいきます  
br2にDHCP+DNS（management=2）を設定します。

参考：http://maas.ubuntu.com/docs/maascli.html#cli-dhcp

```bash
wabuntu@nuc:~$ uuid=$(maas wabuntu node-groups list | grep uuid | cut -d\" -f4)
wabuntu@nuc:~$ maas wabuntu node-group-interface update $uuid eth2 \
 ip_range_high=10.0.0.200 \
 ip_range_low=10.0.0.100 \
 management=2 \
 broadcast_ip=10.0.0.255 \
 router_ip=10.0.0.1 #Success.
```

正直この作業はClusterタブのCluster Masterから、WEBでできるような気がします・・・

### 管理対象マシン（KVMのインスタンス）の登録

ここからは下記のガイダンスに従います  
http://maas.ubuntu.com/docs/nodes.html

maasユーザーというのが自動で作られますが、そいつが管理者ユーザーにパス無しでなれるようにします。最初「いらねーよそんなの。管理者アカウントはwabuntuだよ！」と思ったのですが、システムの自動処理上（Commission）必要なようです。

```bash
wabuntu@nuc:/home/maas$ sudo chsh -s /bin/bash maas
wabuntu@nuc:/home/maas$ sudo su - maas
maas@nuc:~$ ls
boot-resources    dhcp  dhcpd-interfaces    dhcpd6-interfaces  gnupg  media  secret
maas@nuc:~$ pwd
/var/lib/maas
maas@nuc:~$ ssh-keygen -f ~/.ssh/id_rsa -N ''
maas@nuc:~$ ssh-copy-id -i ~/.ssh/id_rsa wabuntu@10.0.0.1
wabuntu@nuc:~$ ls -l .ssh/
-rw------- 1 wabuntu wabuntu 390 May 1 23:57 authorized_keys
```

下記コマンドで、仮想マシンの状態が見えます。ssh-copy-idしたのでパス無しでできます。このインスタンスが、ニセの物理PCになります。ベースのKVMは透明人間です。

```bash
maas@nuc:~$ virsh -c qemu+ssh://wabuntu@10.0.0.1/system list --all
 Id    名前                         状態
----------------------------------------------------
 6     testvm                      実行中
```

次にWeb上でNodeを追加してみます。

- Nodeタブから＋Add　Nodeをクリック
- Power typeをVirshにします
- Power ID :　testvm（上のコマンドで取得したやつ）
- Power Address: qemu+ssh://wabuntu@10.0.0.1/system
- Password:
- MAC: VirtMangerからコピーするのが吉

こんなコマンドもリモートでできます

```bash
virsh -c qemu+ssh://wabuntu@10.0.0.2/system list --all
virsh -c qemu+ssh://wabuntu@10.0.0.2/system iface-dumpxml eth0
virsh -c qemu+ssh://wabuntu@10.0.0.2/system hostname
virsh -c qemu+ssh://wabuntu@10.0.0.2/system sysinfo
virsh -c qemu+ssh://wabuntu@10.0.0.2/system nodeinfo
virsh -c qemu+ssh://wabuntu@10.0.0.2/system net-list
virsh -c qemu+ssh://wabuntu@10.0.0.2/system net-info default
```

out-of-syncが消えないので、ClusterのタブのNICの所と、Networkのタブの内容を正しくする（自動で入ってるけど少し違う）。

メモ：  
ログを見る限り変更、セーブしても内容がすぐ変わらない。しばらくすると、いつの間にかトップページの大きな丸が消えて、リスト型に変わっていた。out-of-syncの解消のサインなのか。これはネットワーク系の問題を解消しするとパスできるようだ。

### Failed to Commisionになる

Power ID(domainうんたら)が取れないそうな。調べてみるとDNS関連の問題でそうなるとのこと。下記のエリアをちゃんとしたら直った。

- Setting=>NetworkConfiguration=>UpstreamDNS：192.168.1.1
- Zones=>Add Zone : maas
- Clusters=>Cluster Master=>Interfaces
  - eth2 : Unmanaged
  - br2 :DHCP and DNS、10.0.0.1…等

### イメージが無いと出る

```
Finished importing boot images, the region does not have any new images.
```

Imageのタブでamd64じゃなくて、i386が必要だったみたい。（VMの設定によるかも）。現在は14.04を使ってMAASを作ってても、pxeが勝手にxenialを落とそうとするので、そっちのimageも必要。

### 特に理由もなくComissioningがFailする

- VirtManagerの該当インスタンスの詳細のページで、Bootの欄が1:HDD 2:PXEになってなかった
- Comissionのトラブルシューティングは、VirtManagerの画面を開いて実際にBootすると良く分かります

流れとしてはこのニセ物理PC（kvmインスタンス）が起動、PXEでDHCPからIPをもらう、DNSを使うのか使わないのか、とにかくMAASに登録されたインストールイメージをダウンロードしてインストールが始まる、という事になります。以上でうまくいけば、１分もせずにNodeがReadyになります。

### スタティックレンジが無いのでDeployできないと言われる

- NodeタブからStaticという設定をeth0にしようとしてもできない。
- ClusterタブからCluster MasterのInterfacesで、**br2のStatick IPが入ってなかった**ので入れる。

以下確認

まずMaasClusterのUUIDをゲット

```bash
wabuntu@nuc:/var/log/maas$ maas wabuntu node-groups list
[
    {
        "name": "maas", 
        "uuid": "e1b6eee7-b574-480c-a5e7-15441eed0ec2"
    }
]
```

そのUUIDを元に、interfacesを出してみる

```bash
wabuntu@nuc:/var/log/maas$ maas wabuntu node-group-interfaces list e1b6eee7-b574-480c-a5e7-15441eed0ec2
[
    {
        "name": "eth0-ipv6-16409", 
        "interface": "eth0"
    }, 
    {
        "name": "eth0", 
        "ip": "192.168.1.50", 
        "interface": "eth0"
    }, 
    {
        "ip_range_high": "10.0.0.150", 
        "ip_range_low": "10.0.0.100", 
        "management": 2, 
        "static_ip_range_low": "10.0.0.151", 
        "name": "br2", 
        "ip": "10.0.0.1", 
        "subnet_mask": "255.255.255.0", 
        "broadcast_ip": "10.0.0.255", 
        "static_ip_range_high": "10.0.0.200", 
        "interface": "br2"
    }
]
```

デプロイできました！  
Nodeタブからいろんな情報がでて、Network欄で10.0.0.151がアサインされたことがわかります。

### インスタンスにログインしてみる

なんでログインできないんだろう、と思ったら、ubuntuユーザーでログインしないといけないようです（そこ大事じゃね？？？）。

```bash
wabuntu@nuc:/var/log/maas$ ssh ubuntu@10.0.0.151
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-83-generic x86_64)
ubuntu@dear-milk:~$ ip a
    inet 10.0.0.151/24 brd 10.0.0.255 scope global eth0
```

パスワードは聞かれませんでしたが、いつの間にか私のid_rsa.pubが、.ssh/authorized_keysに入れられてました。  
疑問。ClusterのページでNetwork Interfaceをいじると（たとえばIP）、/etc/network/interfaceも書き換えてしまうのか、それともMAAS用に新しいIPを定義しろ、ということなのか


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/25/nuc%e4%b8%80%e5%8f%b0%e3%81%ae%e3%81%bf%e3%81%a7%e3%81%a9%e3%82%8c%e3%81%a0%e3%81%91maas%e3%81%a7%e9%81%8a%e3%81%b9%e3%82%8b%e3%81%ae%e3%81%8b/).*
