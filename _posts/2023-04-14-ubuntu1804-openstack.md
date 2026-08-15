---
layout: post
title: "Ubuntu 18.04でOpenStack"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

基本的に流れは下記と同様。

> [Ubuntu16.04でOpenStack(成功)](https://wabuntu.wordpress.com/2017/06/28/ubuntu16-04%e3%81%a7openstackjuju-gui%e3%81%be%e3%81%a7%e6%88%90%e5%8a%9f/)

大きく変わったのは１８．０４はネットワークの設定がまるで違うこと、あとMAASが新しくなっていること。

## サーバのセットアップ

１８．０４のサーバを普通にインストールして開始

初期状態でLC\_ALLが無いがこのままで行ってみる。

```
ubuntu@nuc1:~$ env

SSH_CONNECTION=192.168.1.6 40428 192.168.1.201 22
 LESSCLOSE=/usr/bin/lesspipe %s %s
 LANG=en_US.UTF-8
 XDG_SESSION_ID=1
 USER=ubuntu
 PWD=/home/ubuntu
 HOME=/home/ubuntu
 SSH_CLIENT=192.168.1.6 40428 22
 XDG_DATA_DIRS=/usr/local/share:/usr/share:/var/lib/snapd/desktop
 SSH_TTY=/dev/pts/0
 MAIL=/var/mail/ubuntu
 TERM=xterm-256color
 SHELL=/bin/bash
 SHLVL=1
 LOGNAME=ubuntu
 XDG_RUNTIME_DIR=/run/user/1000
 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
 LESSOPEN=| /usr/bin/lesspipe %s
 _=/usr/bin/env
```

## ネットワーク

```bash
ubuntu@nuc1:~$ ip a
 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group
 2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP inet 192.168.1.201/24 brd 192.168.1.255 scope global eno1
 3: enx7403bd7f1c59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel inet 192.168.100.1/24 brd 192.168.100.255 scope global enx7403bd7f1c59
```

ネットワークはこのように。eno1はホームゲートウェイに繋いでます。

```bash
ubuntu@nuc1:~$ cat /etc/resolv.conf
 nameserver 127.0.0.53
```

ん？こんなの指定してないぞ？

```bash
ubuntu@nuc1:~$ cat /etc/network/interfaces
 # ifupdown has been replaced by netplan(5) on this system. See
 # /etc/netplan for current configuration.
 # To re-enable ifupdown on this system, you can run:
 # sudo apt install ifupdown
```

あれ、interfaces無くなってる・・・。netplanというのに移行したらしい。・・・そしてネットに繋がらない・・・。

```bash
ubuntu@nuc1:~$ cat /etc/netplan/50-cloud-init.yaml
 network:
 ethernets:
 eno1:
 addresses:
 - 192.168.1.201/24
 gateway4: 192.168.1.1
 nameservers:
 addresses:
 - 192.168.1.1
 search: []
 optional: true
 enx7403bd7f1c59:
 addresses:
 - 192.168.100.1/24
 gateway4: 192.168.100.1
 nameservers:
 addresses:
 - 192.168.100.1
 search: []
 optional: true
 version: 2
```

aptできないのでとりあえず上記を1.1に書き換え。デフォルトゲートウェイとかどうなってるのかわからないけど後で調べることにしよう。

参考：https://qiita.com/to_su/items/9b6eae54e59cd3699ae2

```bash
ubuntu@nuc1:~$ sudo vi /etc/netplan/50-cloud-init.yaml
 ubuntu@nuc1:~$ sudo netplan apply
 ubuntu@nuc1:~$ ping google.com
 PING google.com (172.217.161.238) 56(84) bytes of data.
 64 bytes from kix06s05-in-f14.1e100.net (172.217.161.238): icmp_seq=1 ttl=55 time=6.54 ms
```

## MAAS

下記を参考にしながら・・・

https://maas.io/install

```bash
ubuntu@nuc1:~$ sudo maas init
 Create first admin account:
 Username: admin
 Password:
 Again:
 Email: wabuntu@wabuntu.com
 Import SSH keys [] (lp:user-id or gh:user-id):
```

これで下記のWEBにつながるようになる

http://192.168.1.201:5240/MAAS/

自分のユーザーのキーをMAAS画面にコピペ

```bash
ubuntu@nuc1:~$ ssh-keygen -t rsa
ubuntu@nuc1:~$ cat .ssh/id_rsa.pub
```

## KVM

ニセのマシンを２台作るためにKVMを

```bash
ubuntu@nuc1:~$ sudo apt install qemu-kvm libvirt0 libvirt-bin virt-manager bridge-utils
ubuntu@nuc1:~$ wget http://ftp.riken.jp/Linux/ubuntu-releases/18.04/ubuntu-18.04-live-server-amd64.iso
ubuntu@nuc1:~$ sudo mv ubuntu-18.04-live-server-amd64.iso /var/lib/libvirt/images/
```

現在VMは無し状態

```bash
desktop:~$ virsh -c qemu+ssh://ubuntu@192.168.1.201/system list
 ubuntu@192.168.1.201's password:
 Id Name State
 ----------------------------------------------------
```

Virt-managerを使おうとしたらコレ入れろって言われた

```bash
desktop:~$ sudo apt install ssh-askpass
```

Virt-managerで何度もパスワード聞かれたくないのでキーをコピー

```bash
desktop:~$ ssh-copy-id ubuntu@192.168.1.201
```

これでVirt-managerからVMが見られるようになります。

```bash
desktop:~$ virt-manager
```

## ネットの微調整

参考：https://netplan.io/examples

### デフォルトゲートウェイの修正

とりあえずこれで外部につながるようになった。ちなみにroutes:は各NICの中に書く決まりのようだ。

```yaml
network:
 ethernets:
 eno1:
 addresses:
 - 192.168.1.201/24
 routes:
 - to: 0.0.0.0/0
 via: 192.168.1.1
 on-link: true
 gateway4: 192.168.1.1
 nameservers:
 addresses:
 - 192.168.1.1
 search: []
 optional: true
 enx7403bd7f1c59:
 addresses:
 - 192.168.100.1/24
 # gateway4: 192.168.100.1
 nameservers:
 addresses:
 - 192.168.100.1
 search: []
 optional: true
 version: 2
```

### ブリッジの設定

ちゃんとブリッジを使うように書き直す

```bash
ubuntu@nuc1:~$ cat /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
 ethernets:
 eno1:
 addresses:
 - 192.168.1.201/24
 routes:
 - to: 0.0.0.0/0
 via: 192.168.1.1
 on-link: true
 gateway4: 192.168.1.1
 nameservers:
 addresses:
 - 192.168.1.1
 search: []
 optional: true
 enx7403bd7f1c59:
 # addresses:
 # - 192.168.100.1/24
 # gateway4: 192.168.100.1
 # nameservers:
 # addresses:
 # - 192.168.100.1
 # search: []
 optional: true
 bridges:
 br-maas:
 addresses:
 - 192.168.100.1/24
 interfaces:
 - enx7403bd7f1c59
 version: 2
```

アプライするとこのように

```bash
ubuntu@nuc1:~$ sudo netplan apply
 ubuntu@nuc1:~$ ip a
 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group
 2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
 inet 192.168.1.201/24 brd 192.168.1.255 scope global eno1
 3: enx7403bd7f1c59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel
 4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state
 inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
 5: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state
 12: br-maas: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
 inet 192.168.100.1/24 brd 192.168.100.255 scope global br-maas
 13: vlan10@enx7403bd7f1c59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc
```

KVMから使えるようにする（これしてもvirt-managerに自動でリストアップされないので必要かどうか不明）

```bash
ubuntu@nuc1:~$ cat /etc/libvirt/qemu/networks/br-maas.xml
 <network>
 <name>br0</name>
 <bridge name='br-maas'/>
 <forward mode="bridge"/>
 </network>
```

MAASの画面から、Subnetsタブに行き、MAAS用のブリッジbr-maas（この場合fabric-1と表示されている）のuntaggedをクリック、Take actionからProvide DHCPを選ぶ。これで対象サブネットでPXEが使えるようになるはず。

## VM作成

一台VMをためしに作成。ディスクは未だに２台想定しているか分からないけど念の為そうしておく。（MACをMAASに登録する必要があるので覚えておくこと）

```bash
ubuntu@nuc1:~$ NAME="vm1-1"
 ubuntu@nuc1:~$ virt-install --name ${NAME} --vcpus 2 --ram 4096 --pxe --boot network,hd,menu=on --graphics vnc --controller scsi,model=virtio-scsi,index=0 --disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 --disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 --network bridge=br-maas,mac=52:54:00:63:7e:11,model=virtio
```

この状態でVirt-managerからブート（PXE)して、画面を見ていてインストールが始まらないようであれば、DHCPかTFTP（BOOTP)、もしくはMACの登録、MAASのサブネット設定でミスっている。（最後にDevice notfoundとか出る）

## コミッション

MAASのMachinesタブからAdd hardware、Machineを選択。

Machine name,Domainそのまま、MACを入力、Power typeでVirshを選び、Virsh addressにを、パスワードはubuntuユーザーのパスを、Virsh VM IDにはvirsh installで使った名前（この場合にはvm1-1）を入れる。Save machineしてコミッション。（オプションはチェックしないほうがいい感じ）

一個目のコミッションが成功したあたりで、マネージメントネットワークの使用に切り替える

```bash
$ sshuttle -r ubuntu@192.168.1.201 192.168.100.0/24
```

移行は各マシン（３台）に２個ずつVMを作成（１つはJuju用）、それぞれコミッションする。

juju用VMにbootstrapというタグを追加しとく（必要かどうか不明）

## 偽ブリッジ作成

```bash
ubuntu@nuc1:~$ cat /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
 ethernets:
 eno1:
 addresses:
 - 192.168.1.201/24
 routes:
 - to: 0.0.0.0/0
 via: 192.168.1.1
 on-link: true
 gateway4: 192.168.1.1
 nameservers:
 addresses:
 - 192.168.1.1
 search: []
 optional: true
 enx7403bd7f1c59:
 optional: true
 bridges:
 br-maas:
 addresses:
 - 192.168.100.1/24
 interfaces:
 - enx7403bd7f1c59
 br-ext:
 addresses:
 - 192.168.2.1/24
 version: 2
```

VMを削除して作り直し、その後再度コミッション

```bash
ubuntu@nuc1:~$ NAME="vm1-1"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--pxe --boot network,hd,menu=on --graphics vnc \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}-sdb.qcow2,size=64,format=qcow2 \
--network bridge=br-maas,mac=52:54:00:63:7e:11,model=virtio \
--network bridge=br-ext,model=virtio
```

## Bootstrapの下準備

https://docs.jujucharms.com/2.3/en/clouds

```bash
ubuntu@nuc1:~$ sudo maas-region apikey --username=admin
[sudo] password for ubuntu: 
LH4FUUZGvJyavWvY99:....
```

```bash
ubuntu@nuc1:~$ cat mymaas.yaml 
clouds:
 my-maas: 
type: maas 
auth-types: [oauth1] 
endpoint: http://192.168.100.1/MAAS/ 
maas-oauth: LH4FUUZGvJyavWvY99:....
```

## Jujuのセットアップ

参考

https://docs.jujucharms.com/2.3/en/reference-install

```bash
ubuntu@nuc1:~$ sudo snap install juju --classic
juju 2.3.8 from 'canonical' installed
ubuntu@nuc1:~$ snap list juju
Name Version Rev Tracking Developer Notes
juju 2.3.8 4423 stable canonical classic
```

```bash
ubuntu@nuc1:~$ juju clouds
Cloud Regions Default Type Description
aws 15 us-east-1 ec2 Amazon Web Services
aws-china 1 cn-north-1 ec2 Amazon China
aws-gov 1 us-gov-west-1 ec2 Amazon (USA Government)
azure 26 centralus azure Microsoft Azure
azure-china 2 chinaeast azure Microsoft Azure China
cloudsigma 5 hnl cloudsigma CloudSigma Cloud
google 13 us-east1 gce Google Cloud Platform
joyent 6 eu-ams-1 joyent Joyent Cloud
oracle 5 uscom-central-1 oracle Oracle Cloud
rackspace 6 dfw rackspace Rackspace Cloud
localhost 1 localhost lxd LXD Container Hypervisor
```

コレやったら死んだ・・・

```bash
ubuntu@nuc1:~$ juju add-cloud my-maas mymaas.yaml
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x78 pc=0x8061a8]
```

手動なら行けることが判明

```bash
ubuntu@nuc1:~$ juju add-cloud
Cloud Types
 maas
 manual
 openstack
 oracle
 vsphere

Select cloud type: maas

Enter a name for your maas cloud: mymaas

Enter the API endpoint url: http://192.168.100.1/MAAS/
Can't validate endpoint: No MAAS server running at http://192.168.100.1/MAAS/

Enter the API endpoint url: http://192.168.100.1:5240/MAAS/

Cloud "mymaas" successfully added
You may bootstrap with 'juju bootstrap mymaas'
```

キーをコピーしなさいと怒られたので再度

```bash
ubuntu@nuc1:~/snap$ juju bootstrap mymaas
ERROR detecting credentials for "mymaas" cloud provider: maas credentials not found
ubuntu@nuc1:~/snap$ juju add-credential mymaas
Enter credential name: admin

Using auth-type "oauth1".

Enter maas-oauth:

Credentials added for cloud mymaas.
```

## Bootstrap

```bash
ubuntu@nuc1:~/snap$ juju bootstrap mymaas
Creating Juju controller "mymaas" on mymaas
Looking for packaged Juju agent version 2.3.8 for amd64
Launching controller instance(s) on mymaas...
ERROR failed to bootstrap model: cannot start bootstrap instance: unexpected: ServerError: 400 Bad Request ({"distro_series": ["'xenial' is not a valid distro_series.  It should be one of: '', 'ubuntu/bionic'."]})
```

Xenialは対象じゃないから駄目だと言ってる・・・。全部Bionicだよ・・・。

MAASに入って調べてみる。参考：（ここで言ってるPROFILEというのはMAASにログインするユーザ名のこと）

https://docs.maas.io/2.1/en/manage-cli-common

```bash
ubuntu@nuc1:~$ maas login admin http://192.168.100.1:5240/MAAS
ubuntu@nuc1:~$ maas admin nodes read | grep distro
 "distro_series": "",
 "distro_series": "",
 "distro_series": "",
 "distro_series": "",
 "distro_series": "",
 "distro_series": "",
 "distro_series": "bionic",
```

どうもVMにはディストロシリーズというのが明記されないようだ。

https://bugs.launchpad.net/maas/+bug/1767137

バグがすでに報告されていて、２．３．３で直っているようだ。

あれ、２．３．３インストールできない・・・。

```bash
ubuntu@nuc1:~$ sudo apt install maas=2.3.3
Reading package lists... Done
Building dependency tree 
Reading state information... Done
E: Version '2.3.3' for 'maas' was not found
```

```bash
ubuntu@nuc1:~$ sudo apt policy maas
maas:
 Installed: 2.4.0~beta2-6865-gec43e47e6-0ubuntu1
 Candidate: 2.4.0~beta2-6865-gec43e47e6-0ubuntu1
 Version table:
 *** 2.4.0~beta2-6865-gec43e47e6-0ubuntu1 500
 500 http://archive.ubuntu.com/ubuntu bionic/main amd64 Packages
 100 /var/lib/dpkg/status
```

どうも自分でダウンロードしてやらないといけないみたいだ。めんどいので１６．０４をデフォルトにしようとするが、MAASがSyncのまま止まったし、maas initでは古いデータがどっかに残ってる（多分DB程度）ので再インストールする。

一度全部消す

```bash
ubuntu@nuc1:~$ sudo apt-get remove maas --auto-remove --purge
ubuntu@nuc1:~$ dpkg -l|grep -Ei '(maas|postgres)'
ubuntu@nuc1:~$ sudo apt-get remove postgresql-10postgresql-clientpostgresql-client-common --auto-remove --purge
```

どうも新しいmaasがあるようなのでそっちを試す

```bash
ubuntu@nuc1:~$ sudo add-apt-repository ppa:maas/stable
ubuntu@nuc1:~$ sudo apt search maas
Sorting... Done
Full Text Search... Done
maas/bionic 2.4.0-6981-g011e51b7a-0ubuntu1~18.04.1 all
 "Metal as a Service" is a physical cloud and IPAM
ubuntu@nuc1:~$ sudo apt install maas=2.4.0-6981-g011e51b7a-0ubuntu1~18.04.1
```

```bash
ubuntu@nuc1:~$ sudo maas init
Create first admin account:
Username: ubuntu
Password: 
Again: 
Email: wabuntu@wabuntu
Import SSH keys [] (lp:user-id or gh:user-id):
```

ここで気づいたのだが、maas-2.4.0はコミッション対象はbionicでないと駄目なようだ・・・。どうなっとるんだ・・・。

というわけで、２．４．０を入れてかつデフォのイメージを１６．０４にしてみた。（MAASのデータが残らないようにポスグレも念入りに消す）

```bash
sudo apt-get remove maas --auto-remove --purge
dpkg -l|grep -Ei '(maas|postgres)'
sudo apt-get remove postgresql --auto-remove --purge
sudo apt-get remove postgresql-10 --auto-remove --purge
sudo apt-get remove postgresql-client --auto-remove --purge
sudo apt-get remove postgresql-client-common --auto-remove --purge

sudo add-apt-repository ppa:maas/stable
sudo apt install maas=2.4.0-6981-g011e51b7a-0ubuntu1~18.04.1
sudo maas init
```

そしたらいじってるうちにコミッションできなくなった。

赤字のとこがあるとデフォゲがmaas-brになってしまうのでDHCPがうまく行かなかった。消してみるとうまく行く。

```bash
ubuntu@nuc1:~$ cat /etc/netplan/50-cloud-init.yaml 
```

```yaml
network:
 ethernets:
 eno1:
 addresses:
 - 192.168.1.201/24
 routes:
 - to: 0.0.0.0/0
 via: 192.168.1.1
 on-link: true
 gateway4: 192.168.1.1
 nameservers:
 addresses:
 - 192.168.1.1
 search: []
 optional: true
 enx7403bd7f1c59:

 optional: true
 bridges:
 br-maas:
 addresses:
 - 192.168.100.1/24
 # routes:
 # - to: 192.168.100.0/0 
 # via: 192.168.100.1
 interfaces:
 - enx7403bd7f1c59
 # optional: true
 br-ext:
 addresses:
 - 192.168.2.1/24
 # optional: true
 version: 2
```

```bash
ubuntu@nuc1:~$ sudo route
Kernel IP routing table
Destination Gateway Genmask Flags Metric Ref Use Iface
default ntt.setup 0.0.0.0 UG 0 0 0 eno1
default ntt.setup 0.0.0.0 UG 0 0 0 eno1
192.168.1.0 0.0.0.0 255.255.255.0 U 0 0 0 eno1
192.168.2.0 0.0.0.0 255.255.255.0 U 0 0 0 br-ext
192.168.100.0 0.0.0.0 255.255.255.0 U 0 0 0 br-maas
192.168.122.0 0.0.0.0 255.255.255.0 U 0 0 0 virbr0
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/ubuntu-18-04%e3%81%a7openstack/).*
