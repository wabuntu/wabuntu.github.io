---
layout: post
title: "NUC１台でOpenstackは動くのか"
date: 2016-06-17 00:00:00 +0900
lang: ja
---

## 環境

### 使用したマシン（nuc i7モデル+16GBRAM+256G M.2）

- Intel NUC Kit NUC5i7RYH BOXNUC5I7RYH ¥62000
- Transcend SSD 256GB M.2 2242 SATA III 6Gb/s TS256GMTS40 ¥12000
- PC3L-12800 DDR3L 1600 8GB×2 TS1600KWSH-16GK ¥7300
- USB-LAN
- HDMI-miniケーブル
- 16.04 server： ext4（LVMなし）、ホーム暗号化なし、IP固定、基本パッケージ＋SSH

このモデルはミッキーケーブルを別に買う必要はないようでした。

### 操作用マシン（GUIあり）

- 16.04デスクトップ

## KVM

### インストール

インストール前のネットワーク構成は下記です。

```bash
ubuntu@nuc7:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
#これがメインのLAN
2: enp0s25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether b8:ae:ed:7f:0b:46 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global enp0s25
       valid_lft forever preferred_lft forever
3: wlp2s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 00:21:5c:d7:ce:d1 brd ff:ff:ff:ff:ff:ff
#これがUSB-LAN
7: enx7403bd7f1c59: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 74:03:bd:7f:1c:59 brd ff:ff:ff:ff:ff:ff
```

KVMのセットをインストールします。

```bash
ubuntu@nuc7:~$ sudo apt-get install kvm virt-manager libvirt-bin bridge-utils
```

インストール後ネットワークが下記のように変わりました。

```bash
ubuntu@nuc7:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
2: enp0s25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether b8:ae:ed:7f:0b:46 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global enp0s25
       valid_lft forever preferred_lft forever
3: wlp2s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 00:21:5c:d7:ce:d1 brd ff:ff:ff:ff:ff:ff
7: enx7403bd7f1c59: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 74:03:bd:7f:1c:59 brd ff:ff:ff:ff:ff:ff
#下記の２つが追加されてる。これ要るの？
8: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:a0:2b:a4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
9: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue master virbr0 state DOWN group default qlen 500
    link/ether 52:54:00:a0:2b:a4 brd ff:ff:ff:ff:ff:ff
```

### ブリッジ作成

```bash
ubuntu@nuc7:~$ cat /etc/network/interfaces
source /etc/network/interfaces.d/*
# The loopback network interface
auto lo
iface lo inet loopback

#メインのLANは固定IPです
# The primary network interface
auto enp0s25
iface enp0s25 inet static
address 192.168.1.100
gateway 192.168.1.1
dns-nameservers 192.168.1.1

#USB-LANの方を、ブリッジにします
auto enx7403bd7f1c59
iface enx7403bd7f1c59 inet manual
iface br0 inet static
address 10.0.0.1
netmask 255.255.255.0
dns-nameservers 192.168.1.1
bridge_ports enx7403bd7f1c59
bridge_stp 0ff
auto br0
```

すると下記のようなNICが追加されます（中略）

```bash
ubuntu@nuc7:~$ sudo ifup br0
ubuntu@nuc7:~$ ip a
10: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 100
```

下記の３行をsysctrlに追加

```bash
ubuntu@nuc7:~$ sudo vi /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
```

ブリッジはこのような形になります。

```bash
ubuntu@nuc7:~$ brctl show
bridge name    bridge id        STP enabled    interfaces
br0        8000.7403bd7f1c59    no        enx7403bd7f1c59
virbr0        8000.525400a02ba4    yes        virbr0-nic
```

ちなみにUSB-LANの方は、LANケーブルは繋いでおりません。

### KVM使用準備

KVM用のイメージをダウンロードします

```bash
ubuntu@nuc7:/var/lib/libvirt/images$ sudo wget http://ftp.riken.jp/Linux/ubuntu-releases/16.04/ubuntu-16.04-server-amd64.iso
```

### 操作用マシン準備

操作用マシンにvirt-managerをインストールして、起動します。

```bash
wabuntu@nuc:~$ sudo apt install virt-manager ssh-askpass-gnome
```

1. ファイル→新しい接続から、QEMU/KVMでSSH,ユーザーをubuntuにして、ホスト名にIPを指定します。
2. 新規作成でネットワークブート（PXE)を選び、最後のネットワークの選択でbr0を選びます
3. ディスプレイVNCで種類をVNCサーバにしないとパスワードを聞かれ続けるので注意
4. 今回は黒画面が出るとこまで確認できたらOKです

## MAAS

### MAASインストール

インストール。PPAは無くなったみたい。

```bash
ubuntu@nuc7:~$sudo apt install maas
```

### maas-rack-controller(旧maas-cluster-controller)を再設定

```bash
ubuntu@nuc7:~$ sudo dpkg-reconfigure maas-rack-controller
```

サイトを先ほどのbr0のアドレスに変える。（http://10.0.0.1:5240/MAAS_）

### maas-region-controllerを設定

```bash
ubuntu@nuc7:~$ sudo dpkg-reconfigure maas-region-controller
```

PXE/Provisioning network => 10.0.0.1　こっちはいらないかも

### 管理者を作成

```bash
ubuntu@nuc7:~$ sudo maas-region-admin createadmin
WARNING: The maas-region-admin command is deprecated and will be removed in a future version. From now on please use 'maas-region' instead.
Username: ubuntu
Password: 
Again: 
Email: wabuntu@wabuntu.com
```

maas-region-adminは無くなって、maas-regionになるそうな。

### キー等の準備

```bash
#API keyを作成して
ubuntu@nuc7:~$ sudo maas-region-admin apikey > apikey

#クリップボードにコピー
ubuntu@nuc7:~$ cat apikey 
DfLbSJwPmPtwVmcDRP:pTHnyp4uPxkUAhhTBY:pkY5h3Ngxxxxxxxxxxx

#maas loginするとキーを聞かれるのでペースト
ubuntu@nuc7:~$ maas login ubuntu http://localhost/MAAS
API key (leave empty for anonymous access): 

#成功するとこのようにリストが出る
ubuntu@nuc7:~$ maas list
ubuntu http://localhost/MAAS/api/2.0/ DfLbSJwPmPtwVmcDRP:pTHnyp4uPxkUAhhTBY:pkY5h3Ngxxxxxxxxxxxxx

#後ほどsshのキーを使用するのでクリップボードにコピーしておく
ubuntu@nuc7:~$ ssh-keygen -t rsa
ubuntu@nuc7:~$ cat .ssh/id_rsa.pub 
```

### MAAS　WEBへのアクセス

1. `sshuttle -r ubuntu@192.168.1.100 10.0.0.0/24`
2. [http://10.0.0.1:5240/MAASにアクセスし、ubuntuユーザーとパスワードを入れる。](http://10.0.0.1:5240/MAAS)
3. Imagesタブに行ってImport Images、そのあとApply Changes
4. Networksタブを見ると、使いたいbr0の１０．０．０．１がfabric0に割付いている。削除して他のFabricに作りなおす、というのができない。他のFabricにはなぜかサブネットを追加できない。
5. VLANを選択してReconfigure DHCPから10.0.0.1をゲートウェイに変更。
6. 下記コマンドで手動設定を試みる

```bash
#system_idを確認
ubuntu@nuc7:~$ maas ubuntu nodes read | grep system_id
        "system_id": "4y3h7n",

#system_idを指定してネットワークインターフェースを表示
ubuntu@nuc7:~$ maas ubuntu interfaces read 4y3h7n
Success.
Machine-readable output follows:
#br0はこのような情報になっている（中略）
        "name": "br0",
        "vlan": {
            "id": 5001,
            "external_dhcp": "192.168.1.1",
        "type": "bridge",
        "params": "",
        "discovered": null,
        "id": 8,
        "resource_uri": "/MAAS/api/2.0/nodes/4y3h7n/interfaces/8/"

ubuntu@nuc7:~$ maas ubuntu interface update 4y3h7n 8 \
ip_range_high=10.0.0.200 \ 
ip_range_low=10.0.0.100 \ 
management=2 \ 
broadcast_ip=10.0.0.255 \ 
router_ip=10.0.0.1
```

7. Nodeタブに行って、上から２段めの１Controllerを選び、リストアップされた自分のマシンを選ぶ（場所が分かりにくいやろ！）。Interfacesでbr0がfabric0になっていたので１に修正する
8. その後Networkタブから、Fabric0のところのVLAN（Untagged）を選び、DHCPをオフにする。Fabric１の方のDHCPを代わりにON

### Nodeのコミッション

KVM仕様準備で作っておいたゲストVMをここで活用する。

VMの名前は下記のように調べられる

```bash
ubuntu@nuc7:~$ virsh -c qemu+ssh://ubuntu@10.0.0.1/system list
ubuntu@10.0.0.1's password: 
 Id    Name                           State
----------------------------------------------------
 2     ubuntu16.04                    running
```

Nodesタブ、Add HardwareでMachineを選択。

- Power typeをVirshにします
- Power ID :　ubuntu16.04（上のコマンドで取得したやつ）
- Power Address: qemu+ssh://wabuntu@10.0.0.1/system
- Password:
- MAC: virt-managerで作ったゲストVMのMACをコピペ

以上でComissionスタート。トラブルシュートには、virt-managerの画面を開いておくのが便利（DHCPが取れない等が見える）。成功すると１，２分でReadyになるので、なってなければ画面を要チェック。

## Juju

### Jujuのブートストラップ(juju-1：不要。飛ばして可)

yamlファイルを作成

```bash
ubuntu@nuc7:~$ juju-1 generate-config
```

下記に保存されているので、適宜修正します

```bash
ubuntu@nuc7:~$ vi .juju/environments.yaml

default: maas
    maas:
        type: maas
        maas-server: 'http://10.0.0.1:5240/MAAS/'
        maas-oauth: 'DfLbSJwPmPtwVmcDRP:pTHnyp4uPxkUAhhTBY:pkY5h3NgtNgjCsMnvPFqRFesBFEKn7Yy'
        bootstrap-timeout: 1800
        admin-secret: 'xxxxxx'
```

juju-1をつけてversion:1でブートストラップしてみると・・・

```bash
ubuntu@nuc7:~$ juju-1 bootstrap -e maas --debug

このようなエラーが出て失敗！しかもfindしてもそんなファイルは見つからず。

2016-06-17 08:25:17 ERROR juju.cmd supercommand.go:429 cannot determine if environment is already bootstrapped.: could not access file 'd2b615a8-e447-4ff2-866b-09e1ed87bc6f-provider-state': Requested map, got <nil>.
```

iptablesで直るよ！という情報があったので入れてみましたがダメでした。そもそもmaasからネットに繋がるなら下記は不要？(後日談：後ほどjujuのbootstrapの際に、streams.canonical.comからモノをダウンロードしてくる際にこれが必要になります)。設定の参考：http://www.atmarkit.co.jp/ait/articles/0505/17/news131_2.html

#### iptalbes

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s25 -j MASQUERADE
sudo iptables -A FORWARD -i br0 -o enp0s25 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i br0 -o enp0s25 -j ACCEPT

ubuntu@nuc7:~$ sudo iptables-save -c | sudo tee /etc/iptables.rules
ubuntu@nuc7:~$ cat /etc/network/if-pre-up.d/iptables_start 
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.rules
exit 0
ubuntu@nuc7:~$ sudo chmod +x /etc/network/if-pre-up.d/iptables_start
```

ネットをウロウロしてたら**バグのようでした。2.0-beta7で直ってるらしい。**https://bugs.launchpad.net/juju-core/+bug/1564577

### ではjuju-2.0ではどうか

https://jujucharms.com/docs/devel/clouds-maas

下記ファイルを作成。environment.yamlのようなファイルが**新形式**になっているみたい。

```bash
ubuntu@nuc7:~/.juju$ vi environments/cloud.yaml 
clouds:
  my-maas:
    type: maas
    auth-types: [oauth1]
    endpoint: http://10.0.0.1:5240/MAAS/

add-cloudで追加して、list-cloudsで確認。

ubuntu@nuc7:~/.juju$ juju-2.0 add-cloud my-maas ./environments/cloud.yaml 
ubuntu@nuc7:~/.juju$ juju-2.0 list-clouds
CLOUD          TYPE        REGIONS
aws            ec2         us-east-1, us-west-1, us-west-2, eu-west-1, eu-central-1, 
中略
local:my-maas  maas    

#maasのマシンも足しちゃう
ubuntu@nuc7:~/.juju$juju-2.0 add-model nuc7.maas
ubuntu@nuc7:~/.juju$juju-2.0 add-credential my-maas
```

さらに下記の**固定場所、固定ファイル**に、APIキーが必要みたい。

```bash
ubuntu@nuc7:~/.juju$vi ~/.local/share/juju/credentials.yaml 
credentials:
  my-maas:
    maas:
      auth-type: oauth1
      maas-oauth: DfLbSJwPmPtwVmcDRP:pTHnyp4uPxkUAhhTBY:pkY5xxxxxxxxxxxxxx

でそのファイルを引数に入れて、ブートストラップ

ubuntu@nuc7:~/.juju$ juju-2.0 bootstrap nuc7 my-maas --config ~/.local/share/juju/credentials.yaml 
Creating Juju controller "local.nuc7" on my-maas
Bootstrapping model "admin"
Starting new instance for initial controller
Launching instance
WARNING no architecture was specified, acquiring an arbitrary node
ERROR failed to bootstrap model: cannot start bootstrap instance: unexpected: ServerError: 400 BAD REQUEST ({"network": ["Node must be configured to use a network"]})
```

こっちもダメやんか・・・！と思いましたが、Nodesのタブでネットワークの所にきちんとDHCPを指定してやると回避できました。

しかしながらお次はこのエラーが・・・。

```bash
 ubuntu@nuc7:~$ juju-2.0 bootstrap nuc7 my-maas
Creating Juju controller "local.nuc7" on my-maas
Bootstrapping model "admin"
Starting new instance for initial controller
Launching instance
WARNING no architecture was specified, acquiring an arbitrary node
ERROR failed to bootstrap model: cannot start bootstrap instance: unexpected: ServerError: 400 BAD REQUEST ({"storage": ["Specify a storage device to be able to deploy this node.", "Mount the root '/' filesystem to be able to deploy this node."]})
```

ディスクの情報が取れないのはmaasユーザーが使えないからか？と後付けながら追加してみる。ちなみにmaasユーザーそのものはusersでも出てこないが気にせず下記のコマンドを打てばよろし。

#### maasユーザーを追加

```bash
ubuntu@nuc7:~$ sudo mkdir /home/maas
ubuntu@nuc7:~$ sudo chown maas:maas /home/maas/
ubuntu@nuc7:~$ sudo chsh -s /bin/bash maas
ubuntu@nuc7:~$ sudo su - maas
maas@nuc7:~$ ssh-keygen -f ~/.ssh/id_rsa -N ''
maas@nuc7:~$ ssh-copy-id -i ~/.ssh/id_rsa ubuntu@10.0.0.1
maas@nuc7:~$ virsh -c qemu+ssh://ubuntu@10.0.0.1/system list --all
 Id    Name                           State
----------------------------------------------------
 -     kvm0                           shut off
```

前進したのか何なのか新たなエラーが。

```bash
 ubuntu@nuc7:~$ juju-2.0 bootstrap nuc7 my-maas
Creating Juju controller "local.nuc7" on my-maas
Bootstrapping model "admin"
Starting new instance for initial controller
Launching instance
WARNING no architecture was specified, acquiring an arbitrary node
ERROR failed to bootstrap model: cannot start bootstrap instance: cannot run instances: cannot run instance: No available machine matches constraints: zone=default
```

この状態で一度Nodeを削除して作りなおしてみる。すると今回は**自動でInterfacesがens3に、Storageがvda-part1のext4が/に**なってました。つまりmaasユーザーは必要なようだ。ちなみに何度やってもStorage側の情報が取れないことがあったが、MAAS側でNode削除→KVM側でVMを完全に削除して新たに作りなおしたらできたりする。

```bash
ubuntu@nuc7:~$ juju-2.0 bootstrap nuc7 my-maas
```

ここまで持って行くと、上記のコマンドが成功する。(ちなみにbootstrapにはMAAS上のAcuireとかDeployがいるのかな、と思ったが無くても走っちゃった。以前のゴミのせい？)

**まとめるとbootstrapはjuju-2.0を使い、その際にmaasユーザーが必要。ちなみにゲストVMのブートオプションはPXEのみでいいみたいだ。**

この状態で、juju-2.0 bootstrapはattempting to connect to…で停止し、ゲスト側には、ci-info: no authorized ssh keys fingerprints found for user ubuntu.と表示されたが、再起動で直った。おそらく気を利かせて一度HDD起動に変えたのがいけなかった。

成功するとこのようにjuju statusが取れるようになります。

```bash
ubuntu@nuc7:~$ juju status
[Services] 
NAME       STATUS EXPOSED CHARM 
[Units] 
ID      WORKLOAD-STATUS JUJU-STATUS VERSION MACHINE PORTS PUBLIC-ADDRESS MESSAGE 
[Machines] 
ID         STATE DNS INS-ID SERIES AZ

・・・なにも出てへんやんけ！
```

### juju-gui(不要でした。無視していいよ)

案の定juju-guiもデプロイできない。

```bash
ubuntu@nuc7:~$ juju-2.0 deploy juju-gui --to 0
Added charm "cs:juju-gui-130" to the model.
Deploying charm "cs:juju-gui-130" with the default charm metadata series "trusty".
ERROR cannot deploy "juju-gui" to machine 0: machine 0 not found
```

色々出してみるが・・・

```bash
ubuntu@nuc7:~$ juju-2.0 list-controllers
CONTROLLER   MODEL    USER         SERVER
local.nuc7*  default  admin@local  10.0.0.254:17070

ubuntu@nuc7:~$ juju-2.0 list-models
NAME      OWNER        STATUS     LAST CONNECTION
admin     admin@local  available  never connected
default*  admin@local  available  56 seconds ago

ubuntu@nuc7:~$ juju-2.0 list-clouds
CLOUD          TYPE        REGIONS
aws            ec2         us-east-1, us-west-1, us-west-2, eu-west-1, eu-central-1, 
local:my-maas  maas        

ubuntu@nuc7:~$ juju-2.0 list-machines
[Machines] 
ID         STATE DNS INS-ID SERIES AZ
```

どうもadd-machineが再度必要なようだ（なぜ消えた？）

```bash
ubuntu@nuc7:~$ juju-2.0 add-machine nuc7.maas
created machine 0
ubuntu@nuc7:~$ juju-2.0 list-machines

[Machines] 
ID         STATE   DNS INS-ID  SERIES AZ 
0          pending     pending xenial    

ubuntu@nuc7:~$ juju status
[Services] 
NAME       STATUS EXPOSED CHARM 
[Units] 
ID      WORKLOAD-STATUS JUJU-STATUS VERSION MACHINE PORTS PUBLIC-ADDRESS MESSAGE 
[Machines] 
ID         STATE   DNS INS-ID  SERIES AZ 
0          pending     pending xenial
```

しかしながら勝手に使われたCharmがTrusty用らしく怒られて、deployできない。

```bash
ubuntu@nuc7:~$ juju-1 deploy juju-gui --to 2
WARNING unknown config field "noproxy"
WARNING unknown config field "noproxy"
ERROR could not access file '176fe867-d3df-452a-8f2f-a1017c1976ff-provider-state': Requested map, got <nil>.

 ubuntu@nuc7:~$ juju-2.0 deploy juju-gui --to 0
Added charm "cs:juju-gui-130" to the model.
Deploying charm "cs:juju-gui-130" with the default charm metadata series "trusty".
ERROR cannot add service "juju-gui": cannot deploy to machine 0: series does not match
```

何をしてもdefault-seriesがtrustyから変更できない・・・

```bash
#juju-1だとこうなる
ubuntu@nuc7:~$ juju-1 set-env "default-series=xenial"
ERROR could not access file '176fe867-d3df-452a-8f2f-a1017c1976ff-provider-state': Requested map, got <nil>.

#juju-2.0でやろうとすると・・・
ubuntu@nuc7:~$ grep series .juju/environments.yaml
        default-series: 'xenial'
ubuntu@nuc7:~$ grep series .juju/environments/cloud.yaml 
    default-series: xenial
ubuntu@nuc7:~$ juju-2.0 deploy cs:xenial/juju-gui --to 2
Added charm "cs:juju-gui-130" to the model.
Deploying charm "cs:juju-gui-130" with the default charm metadata series "trusty".
ERROR cannot add service "juju-gui": cannot deploy to machine 2: series does not match
ubuntu@nuc7:~$ juju-2.0 deploy juju-gui --to 2
Added charm "cs:juju-gui-130" to the model.
Deploying charm "cs:juju-gui-130" with the default charm metadata series "trusty".
ERROR cannot add service "juju-gui": cannot deploy to machine 2: series does not match

#localだとこうなる
ubuntu@nuc7:~$ juju-2.0 deploy --repository=/home/ubuntu/charms/ local:xenial/juju-gui --to 2
error: flag provided but not defined: --repository
ubuntu@nuc7:~$ juju-1 deploy --repository=/home/ubuntu/charms/ local:xenial/juju-gui --to 2
WARNING unknown config field "noproxy"
WARNING unknown config field "noproxy"
ERROR could not access file '176fe867-d3df-452a-8f2f-a1017c1976ff-provider-state': Requested map, got <nil>.
```

### juju-gui（こっちが正解）

・・・と、上記には色々書いておりましたが、実は**juju-2.0からはjuju-guiはデプロイしなくても入ってる**んだそうです（ええぇえ！）。

```bash
ubuntu@nuc7:~$ juju gui
Opening the Juju GUI in your browser.
If it does not open, open this URL:
https://10.0.0.254:17070/gui/cd5a337b-2905-4557-807a-a52573132976/
Couldn't find a suitable web browser!
Set the BROWSER environment variable to your desired browser.
```

このように出ますんで、その中のURIをブラウザにコピペするとアクセスできます。ユーザーとパスワードは下記で取れます。

```bash
ubuntu@nuc7:~$ juju show-controller --show-passwords
local.nuc7:
  details:
 （中略）
  accounts:
    admin@local:
      user: admin@local
      password: 1632c23ba9797de378f3e20cxxxxxxxxx
```

ちなみにjuju-statusを出すときには、-m でモデルを指定してやるといいそうな。

```bash
ubuntu@nuc7:~$ juju status -m admin
[Services] 
NAME       STATUS EXPOSED CHARM 
[Units] 
ID      WORKLOAD-STATUS JUJU-STATUS VERSION MACHINE PORTS PUBLIC-ADDRESS MESSAGE 
[Machines] 
ID         STATE   DNS        INS-ID SERIES AZ      
0          started 10.0.0.254 4y3h83 xenial default
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/06/17/nuc１台でopenstackは動くのか/).*
