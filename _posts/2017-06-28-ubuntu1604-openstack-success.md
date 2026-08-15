---
layout: post
title: "Ubuntu16.04でOpenStack(成功)"
date: 2017-06-28 00:00:00 +0900
lang: ja
---

## 概要

初めてやるときにとても混乱したので、どのようなデザインなのかを軽く。

```
               [VM juju]   [VM 1]   [VM2] [VM3]   [VM4] [VM5]
              --------------------  ------------  -----------
|   [MAAS]    |      [KVM]       |  |  [KVM]   |  |  [KVM]  |
----------------------------------  ------------  -----------
|         [物理マシン１]           |  |物理マシン２| |物理マシン３|
```

- Openstackは推奨５台、貧乏なのでVM５台でまかないます
- きょう日はLXDなど新しい方法がありますが、KVMへのこだわりがあるので使用します。（LXDも併用）
- MAASは物理マシン１に直でソフトとして入れます（VMでもいいらしい）。
- 物理マシン１のKVM上にVMを２つ作り、ひとつはjuju専用に、もうひとつはノードとして使います。
- MAASはそれらのVMの起動、停止、OSインストール等を行います（物理マシンもコントロールできるけど割愛）。
- jujuはVMに対して「このソフト入れろ」とか指示を出せます。
- Openstack完成にはVMが4個は必要です(openstack base charmの場合)。

## 接続

鬼ざっくりですがこんな感じです。これは最終形で、インストール時は全部ホームゲートウェイに繋いだほうが楽です。

```
　[インターネット]
  -------------
       ||
  ----------------
　[ホームゲートウェイ]  ---- [操作用マシン]
  ----------------
       || eth0
[nuc-1(メイン)]    [nuc-2]    [nuc-3]
       || eth1      ||          ||
      -----------------------------
　　　[           バカハブ           ]
      -----------------------------
```

## 装備

- **操作用マシン**
  - GUIあれば何でも
- **juju用メインマシン(nuc-1) ：コスト８万ぐらい**
  - NUC5i7RYH
    - CPU i7
    - MEM 8G
    - HDD M.2 240G（経験上、Openstackインストール時はHDDの速さが大事）
    - メインLAN　ホームゲートウェイにつなぐ
    - USB-LAN　KVM/MAASネットワーク用、バカハブに分離しとく
    - OS Ubuntu Server 16.04、ユーザー名ubuntu（以前MAASはこのユーザが存在することを期待していたので）、ホーム暗号化なし(確かMAASか何かが以前暗号化未対応だった)、基本デフォルト
- 上記同様のマシンが２台(nuc-2, nuc-3)
- バカハブ：数千円

## 下準備

### ネットワーク系の設定（仮）

NICの名前付けのルールが変わったようです。

いわゆるeth0 : enp0s25

いわゆるeth1 : enx7403bd7f1c59

```bash
ubuntu@nuc1:~$ sudo vi /etc/network/interfaces

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s25 #ホームルーターにつながってるNIC
iface enp0s25 inet static
 address 192.168.1.201
 netmask 255.255.255.0
 network 192.168.1.0
 broadcast 192.168.1.255
 gateway 192.168.1.1
 # dns-* options are implemented by the resolvconf package, if installed
 dns-nameservers 192.168.1.1

#kvm
 auto enx7403bd7f1c59 #KVM用のNIC
 iface enx7403bd7f1c59 inet static
 address 192.168.100.1
 netmask 255.255.255.0
 broadcast 192.168.100.255

ubuntu@nuc1:~$ sudo ifup enx7403bd7f1c59
```

初期はインターネット接続がめんどくさいので、全部バカハブにつないで、それとホームゲートウェイをつなぐのが楽。

### iptables

iptablesでホームルーター（インターネット側）とKVM側をつなぐ。DHCPだけ通さないようにしてみたが効いてるかどうか自信なし。

```bash
ubuntu@nuc1:~$ sudo cat /etc/sysctl.conf | grep forward net.ipv4.ip_forward=1
net.ipv4.ip_forward=1
ubuntu@nuc1:~$ sudo sysctl -p
ubuntu@nuc1:~$ sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/255.255.255.0 -j MASQUERADE
ubuntu@nuc1:~$ sudo iptables -A FORWARD -p udp --dport 67:68 --sport 67:68 -j DROP
ubuntu@nuc1:~$ sudo iptables -L -t nat
... 
Chain POSTROUTING (policy ACCEPT)
target prot opt source destination 
MASQUERADE all -- 192.168.100.0/24 anywhere

ubuntu@nuc1:~$ sudo iptables -L
...
Chain FORWARD (policy ACCEPT)
target prot opt source destination 
DROP udp -- anywhere anywhere udp spts:bootps:bootpc dpts:bootps:bootpc
```

全部バカハブ繋ぎにしたらDHCPの漏れ（ホームルーターのやつがKVM側に流れちゃう）を解消出来なかった・・・。メインLANをホームゲートウェイに、KVM用のLANだけ別のバカハブに挿して、物理的に切り離したほうが楽。

### iptablesの起動設定

```bash
ubuntu@nuc1:~$ sudo iptables-save -c | sudo tee /etc/iptables.rules
ubuntu@nuc1:~$ sudo vi /etc/network/if-pre-up.d/iptables

#! /bin/bash
iptables-restore < /etc/iptables.rules
exit 0

ubuntu@nuc1:~$ sudo chmod +x /etc/network/if-pre-up.d/iptables
```

## MAAS

### MAASのインストール

ロケールが無いと失敗するので注意。

```bash
ubuntu@nuc1:~$ export LC_ALL="en_US.UTF-8"
ubuntu@nuc1:~$ sudo vi .bash_profile
export LC_ALL="en_US.UTF-8"
ubuntu@nuc1:~$ sudo apt install maas
```

### MAASのIP関係を設定

```bash
ubuntu@nuc1:~$ sudo dpkg-reconfigure maas-rack-controller
http://192.168.100.1/MAAS
ubuntu@nuc1:~$ sudo dpkg-reconfigure maas-region-controller
192.168.100.1
```

### Adminアカウントを作成

メアドは何に必要なのか知らん。

```bash
ubuntu@nuc1:~$ sudo maas-region-admin createadmin
Username: ubuntu
Password: 
Again: 
Email: wabuntu@wabuntu.com
Import SSH keys [] (lp:user-id or gh:user-id):
```

APIキーが後々必要になるので取っておくこと。ちなみにmaas-region-adminは無くなって今後、maas-regionになるそうな

```bash
ubuntu@nuc1:~$ sudo maas-region-admin apikey --username=ubuntu > ./apikey
```

### ネットワークを本番用に書き換える

いわゆるeth1の方（KVM用）をブリッジにします。neutronが入るVMだけは、ブリッジが２つ必要です。（１つはKVM等、もうひとつはインターネット）。何もしない空のブリッジを専用に作っておきます。

```
# The primary network interface
auto enp0s25 #インターネット側
iface enp0s25 inet static
 address 192.168.1.201
 netmask 255.255.255.0
 network 192.168.1.0
 broadcast 192.168.1.255
 gateway 192.168.1.1
 # dns-* options are implemented by the resolvconf package, if installed
 dns-nameservers 192.168.1.1

auto enx7403bd7f1c59
iface enx7403bd7f1c59 inet manual

auto br-maas #KVM側
 iface br-maas inet static
 address 192.168.100.1
 netmask 255.255.255.0
 broadcast 192.168.100.255
 dns-nameservers 192.168.100.1 192.168.1.1
 bridge_ports enx7403bd7f1c59 
 bridge_stp off

auto br-internet #neutron-gatewayのためにニセのブリッジ
 iface br-internet inet static
 address 192.168.2.1
 netmask 255.255.255.0
 broadcast 192.168.2.255
 bridge_ports none 
 bridge_stp off
```

newtron-gatewayが２つのブリッジのあるVMを期待しているので、br-internetを作っている。立ち上げたインスタンスからネット接続を実際にさせたい人は、ニセでなく本物のブリッジを作ってホームゲートウェイに繋がるようにしてください。

ここから先は、KVM側のみバカハブに繋いで完全にメインと分離。

### IPV6の使用を停止

いらないかもだけど念の為。どうせIPｖ６のPXEは失敗するのでそのままでもいいですが、無しだと僅かに起動が速いです。

```bash
ubuntu@nuc1:~$ sudo vi /etc/sysctl.conf 
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
ubuntu@nuc1:~$ sudo sysctl -p
```

接続としては簡単に言うと下記のようになっている

```
バカハブ - br - eth1  < forward >  eth0 - ホームゲートウェイ - インターネット
     192.168.1.x              192.168.100.x
```

### DHCP

操作用マシンから、下記のようにして100のネットワークに繋がるようにする。

```bash
$ sshuttle -r ubuntu@192.168.1.201 192.168.100.0/24
```

そうすると、下記のアドレスにブラウザからアクセスできるようになる。

http://192.168.100.1/MAAS

MAASのWEBから下記の要領で、DHCPをONにする（探すのすごい大変だった）

1. Subnetsタブからfabric-1(br-maas)へ
2. そこのVLAN (untagged)をくりっく
3. 右上のTake actionからdhcpをONに

## KVM

### VM作成

現在KVMはこのように無し状態

```bash
ubuntu@nuc1:~$ sudo apt install qemu-kvm libvirt0 libvirt-bin virt-manager bridge-utils
ubuntu@nuc1:~$ virsh -c qemu+ssh://ubuntu@192.168.100.1/system
 list Id    Name                           State
----------------------------------------------------
ubuntu@nuc1:~$
```

起動用イメージをダウンロード

```bash
ubuntu@nuc1:~$ cd /var/lib/libvirt/images/
ubuntu@nuc1:/var/lib/libvirt/images$ sudo wget http://ftp.riken.jp/Linux/ubuntu-releases/16.04/ubuntu-16.04-server-amd64.iso
```

### パス無し設定

絶対必要ではないですが、色々と楽です

```bash
wabuntu@nuc:~$ ssh-keygen -t rsa
wabuntu@nuc:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@192.168.1.201
wabuntu@nuc:~$ ssh-add
```

### VM作成試し

操作用マシンにvirt-managerをインストールして、試しに１個マシンを作成してみる。GUIですが、新しい接続を作成で、SSHで192.168.100.1（または1.201）に入るようにしてください。

作成時は、PXEを選んでブート順序でNICを最初に、画面をSpiceからVNCに変えて（やんないとループする）、MACアドレスをメモっておく。

下記のように新しいVMが確認できます。

```bash
ubuntu@nuc1:/var/lib/libvirt/images$ virsh -c qemu+ssh://ubuntu@192.168.100.1/system list
 Id Name State
 ----------------------------------------------------
 3 ubuntu16.04 running
```

### Comissionを試す

Nodesタブ、Add HardwareでMachineを選択。

- Power typeをVirshにします
- Power ID :　ubuntu16.04（VMのマシン名。上のコマンドで取得したやつ）
- Power Address: qemu+ssh://ubuntu@192.168.100.1/system
- Password: ubuntu@192.168.1.201のパスワード
- MAC: virt-managerで作ったゲストVMのMACをコピペ

試しにCommisionしてみて、virtmanagerの画面上でloginプロンプトから勝手に処理が進んで、SourceMAAS [http://192.168.100.1/MAAS/metadata/] が出る。そうするとMAASのWEB側でNodeがReadyになる。

ここでだんまりになるケースがよくあるので、Virt-managerの画面を開いておいて、DHCPが取れてるか、PXEのブートファイルが取れてるかよく見ておく。

## juju

### juju用VM

ここでjuju用のVMを作成。一応Openstackはsdaとsdbとが存在することを前提にしているようなので、ディスクは２本。MACアドレスを指定しておくと、MAASでのマシン登録の際に楽です。

```bash
NAME="juju"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:7a,model=virtio
```

ルートディレクトリはLXDを使う際に結構使用されるので、３２Gぐらいがいいそうな。（juju用VMは関係ないけど全部この構成で行きます）

### タグ

bootstrapというタグを追加する（タグでマシンを区分けしとくと、Deployのターゲットを選ぶのに楽）。このへんはMAASのGUIからも可能。

```bash
ubuntu@nuc1:~$ maas ubuntu tags read
Success.
Machine-readable output follows:
[
 {
 "kernel_opts": null,
 "definition": "",
 "comment": "",
 "resource_uri": "/MAAS/api/2.0/tags/virtual/",
 "name": "virtual"
 }
]
ubuntu@nuc1:~$ maas ubuntu tag nodes virtual
Success.
Machine-readable output follows:
....

ubuntu@nuc1:~$ maas ubuntu tags create name=bootstrap
Success.
Machine-readable output follows:
{
 "kernel_opts": "",
 "definition": "",
 "comment": "",
 "resource_uri": "/MAAS/api/2.0/tags/bootstrap/",
 "name": "bootstrap"
}

ubuntu@nuc1:~$ maas ubuntu tag nodes virtual | grep hostname
 "hostname": "juju",
ubuntu@nuc1:~$ maas ubuntu nodes read hostname=juju | grep system.id
 "system_id": "ghan78",
....
ubuntu@nuc1:~$ maas ubuntu tag update-nodes bootstrap add=ghan78
Success.
Machine-readable output follows:
{
 "added": 1,
 "removed": 0
}
ubuntu@nuc1:~$ maas ubuntu tag nodes bootstrap
....
```

### pubkey

SSHの公開鍵がDeployに必要なので作っておく。

```bash
ubuntu@nuc1:~$ ssh-keygen -t rsa 
ubuntu@nuc1:~$ cat .ssh/id_rsa.pub
```

MAAS WEBの右上の自分のユーザー名のアイコンをクリック、Add SSH keyにペースト。

### MAASと接続

クラウド設定ファイルを作成。ここがよくわからないが、maas-oauthにapiキーを追加

```bash
ubuntu@nuc1:~$ mkdir environments
ubuntu@nuc1:~$ vi environments/cloud.yaml
clouds:
  my-maas:    
type: maas    
auth-types: [oauth1]    
endpoint: http://192.168.100.1/MAAS/    
maas-oauth: wRw5JcNjJDmCpLdZB7:utJCPNswdB....
```

追加するとこのように出てくる

```bash
ubuntu@nuc1:~$ juju add-cloud my-maas environments/cloud.yaml
ubuntu@nuc1:~$ juju list-clouds
Cloud        Regions  Default          Type        Description
....
my-maas            0                   maas        Metal As A Service
```

MAASのAPIキーを再度コマンドで追加する（これやらないとbootstrapで怒られた）

```bash
ubuntu@nuc1:~$ juju add-credential my-maas
Enter credential name: maas-oauth
                                                            
Using auth-type "oauth1".
Enter maas-oauth: 
Credentials added for cloud my-maas.
```

### bootstrap

```bash
ubuntu@nuc1:~$ juju bootstrap my-maas
Creating Juju controller "my-maas" on my-maas
Looking for packaged Juju agent version 2.0.2 for amd64
Launching controller instance(s) on my-maas...
 - ghan78 (arch=amd64 mem=4G cores=2) 
Fetching Juju GUI 2.7.3
Waiting for address
Attempting to connect to 192.168.100.2:22
Logging to /var/log/cloud-init-output.log on the bootstrap machine
Running apt-get update
Running apt-get upgrade
Installing curl, cpu-checker, bridge-utils, cloud-utils, tmux
Fetching Juju agent version 2.0.2 for amd64
Installing Juju machine agent
Starting Juju machine agent (service jujud-machine-0)
Bootstrap agent now started
Contacting Juju controller at 192.168.100.2 to verify accessibility...
Bootstrap complete, "my-maas" controller now available.
Controller machines are in the "controller" model.
Initial model "default" added.
ubuntu@nuc1:~$
```

statusに出てくるようになる

```bash
ubuntu@nuc1:~$ juju status
Model Controller Cloud/Region Version
default my-maas my-maas 2.0.2
....
ubuntu@nuc1:~$
```

### マシーンを追加（これいらないかも）

```bash
ubuntu@nuc1:~$ juju add-machine nuc1.maas
created machine 0
ubuntu@nuc1:~$ juju list-machines
Machine State DNS Inst id Series AZ
0 pending pending xenial
```

### juju-gui

下記コマンドでjuju-guiのURIが確認できる。（2.0からはjuju-guiのデプロイは不要）

```bash
ubuntu@nuc1:~$ juju gui
Opening the Juju GUI in your browser.
If it does not open, open this URL:
https://192.168.100.2:17070/gui/66225dbb-1b77-48f5-8e2b-ad12221b4b8d/
Couldn't find a suitable web browser!
Set the BROWSER environment variable to your desired browser.
```

juju-guiにアクセスするユーザーとパスワードを取得

```bash
ubuntu@nuc1:~$ juju show-controller --show-password
my-maas:
 details:
 uuid: 8b7dcc2b-b934-4b18-8fa5-871cd9d5b42f
 api-endpoints: ['192.168.100.2:17070']
 ca-cert: |
 -----BEGIN CERTIFICATE-----
 MIIDzTCCArWgAwIBAgIUVfiAVNKdCrttaEbHttkQ1KW/QhEwDQYJKoZIhvcNAQEL
 .......
 3BoaYFFJd66V0M1/0p5wfcU=
 -----END CERTIFICATE-----
 cloud: my-maas
 agent-version: 2.0.2
 controller-machines:
 "0":
 instance-id: ghan78
 models:
 controller:
 uuid: bf7f5b9f-5301-4a86-8afc-50567ba0acb2
 machine-count: 1
 core-count: 2
 default:
 uuid: 66225dbb-1b77-48f5-8e2b-ad12221b4b8d
 machine-count: 1
 current-model: admin/default
 account:
 user: admin <====これを
 access: superuser
 password: ..... <====つかう
ubuntu@nuc1:~$
```

以上で、先ほどのURIからjujuの画面を見れるようになる。

## VM追加

### ２台目のVMを作成

```bash
NAME="nuc1-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:7b,model=virtio \
--network bridge=br-internet,model=virtio
```

Openstackの中で、neutron-gatewayが入るVMにはNICが２枚必要なのでこれを使う。

MAASのNodesに自動的に新しいVMが出てくるので、Powerの部分をいじり、Comissionする。相変わらずHDD等の情報が出るまで３回Comissionが必要だった。チェックボックスは外したほうがいいみたい。

jujuの画面にマシンが表示されないのはこの時点ではOK。

### マシンの追加

juju status では０番がDownとなっている

```bash
ubuntu@nuc1:~$ juju status
Model Controller Cloud/Region Version
default my-maas my-maas 2.0.2

App Version Status Scale Charm Store Rev OS Notes

Unit Workload Agent Machine Public address Ports Message

Machine State DNS Inst id Series AZ
0 down pending xenial
```

どうもコントローラーというものが２つ存在し、１台ずつ別々に登録されているようだ。defaultのが追加した１台。

```bash
ubuntu@nuc1:~$ juju list-models
Controller: my-maas

Model Cloud/Region Status Machines Cores Access Last connection
controller my-maas available 1 2 admin just now
default* my-maas available 1 - admin 17 minutes ago
```

remove-machineをするとこのように消えます。

```bash
ubuntu@nuc1:~$ juju remove-machine 0
ubuntu@nuc1:~$ juju status
Model Controller Cloud/Region Version
default my-maas my-maas 2.0.2

App Version Status Scale Charm Store Rev OS Notes

Unit Workload Agent Machine Public address Ports Message

Machine State DNS Inst id Series AZ
```

その状態でadd-machineをすると、juju画面に出てくるようになりました。Juju/admin/defaultというところにMachineが１台出ます。

```bash
ubuntu@nuc1:~$ juju add-machine
created machine 1
```

LXDに対してデプロイもできるし、SSHもできます。

```bash
ubuntu@nuc1:~$ juju deploy apache2 --to lxd:1
Located charm "cs:apache2-21".
Deploying charm "cs:apache2-21".
ubuntu@nuc1:~$ juju status
Model Controller Cloud/Region Version
default my-maas my-maas 2.0.2

App Version Status Scale Charm Store Rev OS Notes
apache2 waiting 0/1 apache2 jujucharms 21 ubuntu

Unit Workload Agent Machine Public address Ports Message
apache2/0 waiting allocating 1/lxd/0 waiting for machine

Machine State DNS Inst id Series AZ
1 started 192.168.100.4 mnqc4x xenial default
1/lxd/0 pending pending xenial

ubuntu@nuc1:~$ juju ssh 
error: no target name specified
ubuntu@nuc1:~$ juju ssh 1
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-81-generic x86_64)
```

## 物理マシン2台目

### OSインストール

ホームゲートウェイに繋いだ状態で、１台目と同じようにインストール。IPは1.202をふっておく。お金かポートに空きがあれば、物理マシン１台目と同じようにNIC２枚にした方が分かりやすいかも。

### ネットワーク設定

```
# The primary network interface
auto eno1
iface eno1 inet manual

auto br-maas
iface br-maas inet static
address 192.168.1.202
netmask 255.255.255.0
network 192.168.1.0
broadcast 192.168.1.255
gateway 192.168.1.1
dns-nameserver 192.168.1.1
bridge_ports eno1
bridge_stp off
```

### IPV6の使用を停止

IPｖ６無しだと僅かに起動が速いです。

```bash
ubuntu@nuc1:~$ sudo vi /etc/sysctl.conf 
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
ubuntu@nuc1:~$ sudo sysctl -p
```

### KVMインストール

```bash
ubuntu@nuc2:~$ sudo apt install qemu-kvm libvirt0 libvirt-bin virt-manager bridge-utils
```

### VM作成

MACアドレスは何でもいいですが、毎回少し変えてください。

```bash
NAME="nuc2-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:7c,model=virtio
```

PXEは当然失敗するので、virt-managerから強制的に電源を切ります。

```bash
NAME="nuc2-2"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:7d,model=virtio
```

２つめのVMも同じように。名前とMACを変えることを忘れずに。

### MAASに追加

IPを100.xの開いてる奴に変えて、LANはバカハブに繋ぎ直して、さらに再起動します。sshuttleが効いているので、操作マシンからそのIPにvirt-managerを繋げられます。

```
# The primary network interface
auto eno1
iface eno1 inet manual

auto br-maas
iface br-maas inet static
address 192.168.100.202
netmask 255.255.255.0
network 192.168.100.0
broadcast 192.168.100.255
gateway 192.168.100.1
dns-nameserver 192.168.100.1
bridge_ports eno1
bridge_stp off
```

virt-manager（１００．202につなぐ）から上記の２つのVMを起動すると、PXEで自動的にOSインストールが始まります。

そうするとMAASのGUIのNodesに新たな２台が追加されます。

### Comission

それぞれPowerTypeを書き換えます。左上の名前も変えていいです（クリックするとEditになる）。

- Power address : qemu+ssh://ubuntu@192.168.100.202/system
- Power ID : nuc2-1, nuc2-1

それからComissionします。なぜか２回Comissionしないと成功しませんでしたが、とりあえずReady状態になります。Comissionの時に出るチェックボックスは外したほうが吉かも。

### Jujuに追加（しなくてもよかった）

ここでメインのjujuが入ってるマシン(nuc-1)から下記を実行すると、勝手にMAASのデプロイが始まります。

```bash
ubuntu@nuc1:~$ juju add-machine nuc2-1.maas
```

ここまで途中、Virt-managerから強制OFFや起動などの手助けが必要な場合があるようです。

まとめるとこのような感じ。

1. virt-installでVM作成
2. VMが勝手に起動してMAASのGUIに出てくる
3. ここでインストールが勝手に走るがその結果は無視
4. 名前、power type、タグを編集（ステータスはNewになっている）
5. MAASからComission（チェックボックスは全部無しでいい）
6. cloud-init[12343]: Successから数分待って、勝手に電源が切れたらReady状態になる
7. MAASからHDD等の情報が見えるのを確認
8. ここまでで、jujuのguiにマシンが表示されなくてもOK
9. メインマシン(nuc-1)からjuju add-machine nuc?-?.maasと名前指定で走らせると、勝手にDeployが始まる（Deployが終了するまで数分、juju statusはpendingになる）
10. juju status及びjujuの画面でマシンが確認できる

## 物理マシン3台目

２台目と全く同じ。IPとかVM名だけ変えてね。もちろんMACも。

```bash
NAME="nuc3-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:80,model=virtio

NAME="nuc3-2"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:81,model=virtio
```

## OpenStack Bundle Deploy

この時点でdefaultというモデルには、マシンが１台も表示されて無くて正解。何か失敗してやり直したいときは下記。

```bash
juju destroy-model default
juju add-model default
```

真ん中の緑の◯をクリックして、juju-charmのページに飛び、OpenstackをView、そこでOpenstack baseをクリック。右端の緑の◯をクリック（ここでAdd modelにしないように）。自動でJujuの画面が動き出す。Commit Changes、そしてDeploy。

ここで２つ問題が有ります。

- neutron-gatewayは確実にnuc1-1（マシン番号0）に入るようにしたい
- newtron-gateway用のVMで、インターネット向けのブリッジを名前で指定しないといけない（昔はeth0とか固定的な名前だったので必要なかった）

なので下記のアドレスの右中程にある、Download.zipを、nuc-1というVMにダウンロードして解凍します。使用するファイルはbundle.yamlです。

インターネット用のブリッジが何になってるかは、まずMAASのNodes、nuc1-1、Interfaceに行って今のIPを調べ、それにubuntuユーザーでsshします。

```bash
ubuntu@nuc1-1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
 inet 127.0.0.1/8 scope host lo
 valid_lft forever preferred_lft forever
 inet6 ::1/128 scope host 
 valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br-ens3 state UP group default qlen 1000
 link/ether 52:54:00:63:7e:7b brd ff:ff:ff:ff:ff:ff
3: ens4: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP group default qlen 1000
 link/ether 52:54:00:e5:95:ac brd ff:ff:ff:ff:ff:ff
 inet6 fe80::5054:ff:fee5:95ac/64 scope link 
 valid_lft forever preferred_lft forever
4: br-ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
 link/ether 52:54:00:63:7e:7b brd ff:ff:ff:ff:ff:ff
 inet 192.168.100.135/24 brd 192.168.100.255 scope global br-ens3
 valid_lft forever preferred_lft forever
 inet6 fe80::5054:ff:fe63:7e7b/64 scope link 
 valid_lft forever preferred_lft forever
```

このようになっていた場合、ens4がインターネット用ニセブリッジです。

次にbundle.yamlを編集します。下記のように、インターネット用のブリッジ(ens4)と、nuc1-1（マシン番号０）を指定。

```yaml
 neutron-gateway:
 annotations:
 gui-x: '0'
 gui-y: '0'
 charm: cs:neutron-gateway-234
 num_units: 1
 options:
 bridge-mappings: physnet1:br-ex
 data-port: br-ex:ens4
 openstack-origin: cloud:xenial-newton
 worker-multiplier: 0.25
 to:
 - '0'
```

そしていよいよDeployです。

```bash
ubuntu@nuc1:~$ juju deploy bundle.yaml
```

下記でDeployのステータスが見れます。

```bash
ubuntu@nuc1:~$ watch --color "juju status --color"
```

または

```bash
juju debug-log
```

私のケースでは４５分でデプロイ完了しました。

ちなみに現在MAAS２．１の問題がありますが、MAASのデフォのデプロイカーネルをgaからhweに変えればいけます。

### Openstack Dashboard

最期にダッシュボードにアクセスする方法を

```bash
ubuntu@nuc1:~$ juju expose openstack-dashboard
ubuntu@nuc1:~$ juju status | grep dash
openstack-dashboard 10.0.3 active 1 openstack-dashboard jujucharms 245 ubuntu exposed
openstack-dashboard/0* active idle 3/lxd/2 192.168.100.148 80/tcp,443/tcp Unit is ready
identity-service keystone openstack-dashboard regular
cluster openstack-dashboard openstack-dashboard peer
```

下記アドレスにブラウザで。ユーザー＆パスはadmin/openstackです。（いいのか？）

http://192.168.100.148/horizon/

**完**

---

## DHCP格闘メモ

```bash
sudo iptables -A OUTPUT -p udp --dport 67:68 --sport 67:68 -j LOG
sudo iptables -A INPUT -p udp --dport 67:68 --sport 67:68 -j LOG
sudo iptables -A FORWARD -p udp --dport 67:68 --sport 67:68 -j LOG

Jun 23 16:17:25 nuc1 kernel: [ 1099.839033] IN=br-maas OUT= MAC=ff:ff:ff:ff:ff:ff:52:54:00:b6:a4:40:08:00 SRC=0.0.0.0 DST=255.255.255.255 LEN=430 TOS=0x00 PREC=0x00 TTL=64 ID=256 PROTO=UDP SPT=68 DPT=67 LEN=410 
Jun 23 16:17:25 nuc1 kernel: [ 1099.839139] IN=enp0s25 OUT= MAC=ff:ff:ff:ff:ff:ff:52:54:00:b6:a4:40:08:00 SRC=0.0.0.0 DST=255.255.255.255 LEN=430 TOS=0x00 PREC=0x00 TTL=64 ID=256 PROTO=UDP SPT=68 DPT=67 LEN=410

16:40:28.701221 IP (tos 0xb8, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    192.168.1.1.67 > 192.168.1.6.68: [udp sum ok] BOOTP/DHCP, Reply, length 300, xid 0xe013536e, Flags [none] (0x0000)
  Your-IP 192.168.1.6
  Client-Ethernet-Address 52:54:00:b6:a4:40
  Vendor-rfc1048 Extensions
    Magic Cookie 0x63825363
    DHCP-Message Option 53, length 1: Offer
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2017/06/28/ubuntu16-04でopenstackjuju-guiまで成功/).*
