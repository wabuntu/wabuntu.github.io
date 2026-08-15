---
layout: post
title: "マシン１台でOpenStack (Ubuntu 18.04)"
date: 2020-03-16 00:00:00 +0900
lang: ja
---

## 構成

今回はマシン一台でOpenStackをデプロイできるか試してみる。今まで複数台のNUCを使っていたが、一台の中にVMを６つ作成する。一台構成のメリットとして

*   ネットワーク構成がシンプルで済む（DHCPやルーティング関連）
*   VMのスナップショットを取ることで構成まるごとバックアップできる

ということが考えられます。

使ったマシンはこのようなものです。

*   **CPU** : Corei9 10コア 20スレッド（おそらく10スレッドで十分）
*   **メモリ**: 48GB（32GBでも大丈夫）
*   **HDD** : M.2 SDD 1TB（個人的にディスクの速度は重要な気がする）

OpenStack-base charmの要件として、VMが最低４台、それぞれに仮想NICが２枚ずつ、HDDが２つ（２つめはCeph用）というのがあります。Juju controllerにはメモリが４GB必要です。

VMはVCPUを各2、メモリを各4GBでも快適に動きました。ルートディスクは20GB程度消費します（LXDが結構消費する）。ディスク1を32GB、ディスク2を64GBとすると、合計でVCPU 11, MEM 22GB, HDD 430GB必要です。

*   **MAAS VM** : VCPU 1, MEM 2GB, HDD 15GB
*   **Juju controller VM** : VCPU 2, MEM 4GB, HDD 15GB
*   **Node VM X 4** : VCPU 2, MEM 4G, HDD 32+64GB

環境全部バックアップできるように、MAAS VMを操作用端末にします。

## KVM

### KVMネットワーク設定

#### virbr0

1.  初期画面でQEMU/KVMを右クリック→詳細→仮想ネットワーク
2.  既存のやつを削除して新しくdefaultを作り直す
3.  新規作成のステップ２でDHCPv4をOFF、サブネットは192.168.122.0にした
4.  IPv6は無視
5.  ステップ４の物理ネットワークにフォワードで、いずれかのデバイス、NATを選ぶ

#### virbr1

1.  上記同様にvirbr1名義で追加のネットワークを作成（VMはNICが２つずつあるので、もう一個用だが実際には使わない）。物理ネットワークへの接続は隔離された仮想ネットワークを選択。

ホストマシンのIPは下記のようになる。

```
56: virbr0: mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether 52:54:00:59:09:78 brd ff:ff:ff:ff:ff:ff
inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0

80: virbr1: mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether 52:54:00:06:73:3a brd ff:ff:ff:ff:ff:ff
inet 192.168.123.1/24 brd 192.168.123.255 scope global virbr1
```

LibvirtdのDHCPを殺すことがポイント

### VM作成

#### MAAS用VM

MAAS用はVirt Managerで作ってもいいと思います。Ubuntu Server のisoをダウンロードしてきてインストール

TRIM機能(fstrim)が使えるということで、–controller scsi, model=virtio-scsiとbus=scsiを入れる。あとでxmlのdisk driverにdiscard='unmap'を指定して、rootで"fstrim -v /"を実行すると無駄なゴミスペースを削除できるらしい。

[http://dustymabe.com/2013/06/11/recover-space-from-vm-disk-images-by-using-discardfstrim/](http://dustymabe.com/2013/06/11/recover-space-from-vm-disk-images-by-using-discardfstrim/)

```bash
cd /var/lib/libvirt/images
wget http://releases.ubuntu.com/18.04.4/ubuntu-18.04.4-live-server-amd64.iso

NAME="maas"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--boot hd,menu=on --graphics spice \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-vda.qcow2,size=16,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:aa,model=virtio \
--cdrom=ubuntu-18.04.4-live-server-amd64.iso
```

## MAAS

やり方はここに従う。だいぶ以前と変わったようだ。

[https://maas.io/install](https://maas.io/install)

```bash
ubuntu@maas:~$ sudo snap install maas --channel=2.7 
[sudo] password for ubuntu: 
maas (2.7/stable) 2.7.0-8234-g.de2727f6c from Canonical✓ installed ubuntu@maas:~$ sudo maas init 
Mode (all/region+rack/region/rack/none) [default=all]? 
MAAS URL [default=http://192.168.122.2:5240/MAAS]: 
Create first admin account Username: maasadmin 
Password: Again: 
Email: uso@uso.com 
Import SSH keys [] (lp:user-id or gh:user-id):  
```

上記のURLに接続すると、初期のセットアップが要求される。region名はmaasregionにしました。

KVMという新しいタブがあるのでクリックして、ホストマシンの下記のアドレスとパスワードを入れてみた。（ほかはデフォルト） qemu+ssh://wabuntu@192.168.0.5/system

**自動で全VMComissionしようとする**ので、別の用途のVMが消されそうになって焦った。

### Juju controller用VM

Jujuのサーバ機能が入るVMです。MAASからコミッションするので –pxeをつけて。

```bash
NAME="controller"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--boot network,hd,menu=on --graphics spice --pxe \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-vda.qcow2,size=16,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:bb,model=virtio
```

コマンド終了するなりインストールがはじまり、再起動後もう一回インストールが。MAASのMachinesに出てくるので、Comissionする。MAASの該当Machineでbootstrapのタグをつける。

### Comissionに失敗するときは

Virt-managerの画面で起動を見ていて、Ctrl-Bを連射してPXEのコマンドに入ってdhcp, show ip, show filenameを見るとファイル名が取れてないなどがわかる。

```bash
tcpdump -vvv port 67 or port 68 or port 69 or port 4011
```

を実行してやりとりを確認するなど。

ちなみにCommissionがちゃんと進むときは、対象のVMの画面を見ているとPXIEでBoot-kernelというファイルを取ってきたあと、Maas-enlisting何とかというホスト名が見える。

ログインプロンプトで止まってだんまりのときは失敗。プロンプト後もCloud-initが走っている状態がよし。Comissionのあと電源が切れる。

### ノード用VM

合計４台作成してください。MACがノード番号ごとに変わるようにしました。ネステッドVMになるので、–cpu hostをつけて。vcpuやramのサイズはお好みで。–controller scsiの方が、ディスクを節約できるらしい。

```bash
NODE="01"
virt-install --name node${NODE} --vcpus 3 --ram 6144 --cpu host \
--boot network,hd,menu=on --graphics spice --pxe \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/node${NODE}-vda.qcow2,size=32,format=qcow2,bus=scsi \
--disk path=/var/lib/libvirt/images/node${NODE}-vdb.qcow2,size=64,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:${NODE},model=virtio \
--network bridge=virbr1,mac=52:54:00:00:02:${NODE},model=virtio
```

Controllerと同じように、NewステータスになったらComissionする。

## Juju 

### MAASをJujuに登録

```bash
ubuntu@maas:~$ sudo snap install juju --classic
ubuntu@maas:~$ cat maas-cloud.yaml
type: maas
auth-types: [oauth1]
endpoint: http://192.168.122.2:5240/MAAS

ubuntu@maas:~$ juju add-cloud --local maas-cloud maas-cloud.yaml
ubuntu@maas:~$ juju clouds --local
maas-cloud 1 default maas 0 local Metal As A Service
#ここでMAASの右上のユーザー=>API Keysからキーをコピーしておく
ubuntu@maas:~$ juju add-credential maas-cloud
Do you ONLY want to add a credential to this client? (Y/n): y
Enter credential name: maas-cloud-creds
Regions
defaul
Select region [any region, credential is not region specific]:
Using auth-type "oauth1".
Enter maas-oauth: #ここにAPIキーをペースト
Credential "maas-cloud-creds" added locally for cloud "maas-cloud".
ubuntu@maas:~$ juju credentials
maas-cloud maas-cloud-creds
ubuntu@maas:~$ juju bootstrap maas-cloud maas-controller
```

Jujuは最低3584MBのメモリが必要

### JujuのGUIにログインする

コマンドでURLとユーザー、パスワードが確認できる。パスワードは煩雑なので更新。

```bash
ubuntu@maas:~$ juju gui
GUI 2.15.0 for model "admin/default" is enabled at:
https://192.168.122.10:17070/gui/u/admin/default
Your login credential is:
username: admin
password: dfdd47e8ace514c8f7b10a00c8b3a3d4
ubuntu@maas:~$ juju change-user-password admin
new password:
type new password again:
Your password has been changed.
```

## OpenStack

### Bundleの書き換え

以降は下記サイトに従う

[https://jaas.ai/openstack-base](https://jaas.ai/openstack-base)

```bash
ubuntu@maas:~$ charm pull openstack-base ~/openstack-base
cs:bundle/openstack-base-65
ubuntu@maas:~$ vi openstack-base/bundle.yaml
```

MAASのMachineでHDDのドライブ名を確認して、ceph-osdの設定を書き換える

```yaml
ceph-osd:
  annotations:
    gui-x: '1000'
    gui-y: '500'
  charm: cs:ceph-osd-294
  comment: SET osd-devices to match your environment
  num_units: 3
  options:
    osd-devices: /dev/sdb
    source: cloud:bionic-train
```

MAASのMachineで見ると、全て２番目のNIC（External用ニセNIC）の名前がens4になっているので、neutron-gatewayのExternal用NICをその旨設定

| NAME | MAC | LINK SPEED | FABRIC_HELP: | DHCP | SR-IOV |
|------|-----|-----------|--|------|--------|
| ens3 | 52:54:00:00:01:01 | 0 Mbps | fabric-0 | MAAS-provided | No |
| ens4 | 52:54:00:00:02:01 | 0 Mbps | Unknown | No DHCP | No |

```yaml
neutron-gateway:
  annotations:
    gui-x: '0'
    gui-y: '0'
  charm: cs:neutron-gateway-276
  comment: SET data-port to match your environment
  num_units: 1
  options:
    bridge-mappings: physnet1:br-ex
    data-port: br-ex:ens4
    openstack-origin: cloud:bionic-train
    worker-multiplier: 0.25
  to:
    - '0'
```

### デプロイ

モデルをdefaultにスイッチしてデプロイ

```bash
ubuntu@maas:~$ juju switch maas-controller:default
maas-controller:admin/default (no change)
ubuntu@maas:~$ cd openstack-base/
ubuntu@maas:~/openstack-base$ juju deploy ./bundle.yaml
ubuntu@maas:~/openstack-base$ watch -c juju status --color
```

### OpenStackの操作

```bash
ubuntu@maas:~$ sudo snap install openstackclients --classic
ubuntu@maas:~$ source openstack-base/openrc
ubuntu@maas:~$ openstack service list
+----------------------------------+-----------+--------------+
| ID | Name | Type |
+----------------------------------+-----------+--------------+
| 02528ea8969a4e3d97342dd8bdba7806 | swift | object-store |
| 211baebbfc5648b29c9296f6c9e9c04e | keystone | identity |
| 373df7cf7248445ab093aea5a5fff407 | cinderv2 | volumev2 |
| 4195683b6279403188498ed6631c31b3 | cinderv3 | volumev3 |
| 5c4ae4a16def4eada29827f6cbd73839 | neutron | network |
| be3d524e147e49b1901e8c59044e310e | placement | placement |
| eae6756782da499fa1b2e6c296fef7ce | glance | image |
| efb7b0a91c844c81823fcba295877766 | nova | compute |
+----------------------------------+-----------+--------------+
```

イメージをダウンロードして作成

```bash
curl http://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img | \
   openstack image create --public --container-format=bare --disk-format=qcow2 \
   bionic
```

一部追加でインストールしないとけないツールがあった

```bash
ubuntu@maas:~/openstack-base$ sudo apt install python  
ubuntu@maas:~/openstack-base$ sudo apt install python-keystoneclient python-neutronclient  
$ sudo apt install python-novaclient python-keystoneclient python-glanceclient python-neutronclient python-openstackclient -y  
$ source openrc  
```

KVMで作っておいたExtネットに基づいて外部ネットワーク作成

```bash
ubuntu@maas:~/openstack-base$ ./neutron-ext-net-ksv3 –network-type flat -g 192.168.123.1 -c 192.168.122.0/24 -f 192.168.123.100:192.168.123.200 ext_net  
```

ルーターを作成

```bash
ubuntu@maas:~/openstack-base$ ./neutron-tenant-net-ksv3 -p admin -r provider-router internal 10.5.5.0/24  
```

フレーバーを作成

```bash
ubuntu@maas:~/openstack-base$ openstack flavor create –ram 1024 –disk 6 m1.tiny  
```

キーペアを設定

```bash
ubuntu@maas:~/openstack-base$ ssh-keygen -q -N " -f ~/.ssh/id_mykey  
ubuntu@maas:~/openstack-base$ openstack keypair create –public-key ~/.ssh/id_mykey.pub mykey  
```

PingとSSHを許可するセキュリティグループを作成

```bash
ubuntu@maas:~/openstack-base$ for i in $(openstack security group list | awk '/default/{ print $2 }'); do  
  openstack security group rule create $i –protocol icmp –remote-ip 0.0.0.0/0
  openstack security group rule create $i –protocol tcp –remote-ip 0.0.0.0/0 –dst-port 22
done  
```

インスタンスを作成

```bash
ubuntu@maas:~/openstack-base$ openstack server create –image bionic –flavor m1.tiny –key-name mykey \
  –nic net-id=$(openstack network list | grep internal | awk '{ print $2 }') \
  bionic-1
```

ボリュームを作成してアタッチ

```bash
ubuntu@maas:~/openstack-base$ openstack volume create –size=10 vol10
ubuntu@maas:~/openstack-base$ openstack server add volume bionic-1 vol10  
```

フローティングIPを作成して追加

```bash
ubuntu@maas:~/openstack-base$ FLOATING_IP=$(openstack floating ip create -f value -c floating_ip_address ext_net)  
ubuntu@maas:~/openstack-base$ openstack server add floating ip bionic-1 $FLOATING_IP  
```

SSHでログインできることを確認

```bash
ubuntu@maas:~/openstack-base$ ssh -i ~/.ssh/id_mykey ubuntu@$FLOATING_IP
```

今ぐらいの強力なマシンは多分必要ないな、というくらい応答も処理もスムーズ。おそらくVCPUは各２、メモリは各４GBで十分。余力があるぐらいかもしれない。

## その他

### Dashboard(horizon)で、コンソールアクセスを使用できるようにする

```yaml
vi bundle.yaml 
nova-cloud-controller:
  annotations:
    gui-x: '0'
    gui-y: '500'
  charm: cs:nova-cloud-controller-339
  num_units: 1
  options:
    network-manager: Neutron
    openstack-origin: cloud:bionic-train
    worker-multiplier: 0.25
    console-access-protocol: spice
  to:
    - lxd:2
```

```bash
ubuntu@maas:~/openstack-base$ juju deploy ./bundle.yaml
juju status
nova-cloud-controller/0* active executing 2/lxd/2 192.168.122.56 8774/tcp,8775/tcp (config-changed) Unit is ready
```

もしくはコマンドラインでConfigを変更

```bash
ubuntu@maas:~/openstack-base$ juju config nova-cloud-controller 'console-access-protocol=novnc'
ubuntu@maas:~/openstack-base$ juju debug-log
```

Console用のproxyというプロセスが走るようになる。（spiceは実験のゴミ）

```bash
ubuntu@maas:~/openstack-base$ juju ssh nova-cloud-controller/0 ps -deaf | grep nova
root 267 1 0 02:53 ? 00:00:00 bash /etc/systemd/system/jujud-unit-nova-cloud-controller-0-exec-start.sh
root 283 267 0 02:53 ? 00:00:10 /var/lib/juju/tools/unit-nova-cloud-controller-0/jujud unit --data-dir /var/lib/juju --unit-name nova-cl
nova 5645 1 1 03:03 ? 00:02:43 /usr/bin/python3 /usr/bin/nova-conductor --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova-
nova 5646 1 1 03:03 ? 00:03:25 /usr/bin/python3 /usr/bin/nova-scheduler --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova-
nova 5663 5646 0 03:03 ? 00:00:21 /usr/bin/python3 /usr/bin/nova-scheduler --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova-
nova 5664 5646 0 03:03 ? 00:00:20 /usr/bin/python3 /usr/bin/nova-scheduler --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova-
nova 5665 5646 0 03:03 ? 00:00:20 /usr/bin/python3 /usr/bin/nova-scheduler --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova-
61512 1 0 06:50 ? 00:00:01 /usr/bin/python3 /usr/bin/nova-spicehtml5proxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova
nova 67536 1 0 06:56 ? 00:00:01 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 79510 425 1 07:15 ? 00:00:06 (wsgi:nova-api-os -k start
nova 79511 425 0 07:15 ? 00:00:01 (wsgi:nova_meta) -k start
nova 82159 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 82178 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 82179 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 82180 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 82181 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 82182 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
nova 82201 67536 0 07:20 ? 00:00:00 /usr/bin/python3 /usr/bin/nova-novncproxy --config-file=/etc/nova/nova.conf --log-file=/var/log/nova/nova
```

これで以降の新しいプロセスからは、InstanceのConsoleタブから画面が見えるようになる。

単に画面上のログが見たい場合はこのような方法も

```bash
$openstack console log show bionic-1
```

### Cephの利用のされ方

```bash
ubuntu@juju-bde0e9-1-lxd-0:~$ sudo ceph osd tree
ID CLASS WEIGHT TYPE NAME STATUS REWEIGHT PRI-AFF
-1 0.18750 root default
-5 0.06250 host node02
2 hdd 0.06250 osd.2 up 1.00000 1.00000
-7 0.06250 host node03
1 hdd 0.06250 osd.1 up 1.00000 1.00000
-3 0.06250 host node04
0 hdd 0.06250 osd.0 up 1.00000 1.00000
ubuntu@juju-bde0e9-1-lxd-0:~$ sudo ceph -s
cluster:
  id: e5afd152-64ff-11ea-ac77-00163e698e83
  health: HEALTH_OK
services:
  mon: 3 daemons, quorum juju-bde0e9-3-lxd-0,juju-bde0e9-1-lxd-0,juju-bde0e9-2-lxd-0 (age 3h)
  mgr: juju-bde0e9-1-lxd-0(active, since 3h), standbys: juju-bde0e9-2-lxd-0, juju-bde0e9-3-lxd-0
  osd: 3 osds: 3 up (since 3h), 3 in (since 2d)
  rgw: 1 daemon active (juju-bde0e9-0-lxd-0)
data:
  pools: 17 pools, 82 pgs
  objects: 270 objects, 330 MiB
  usage: 4.0 GiB used, 188 GiB / 192 GiB avail
  pgs: 82 active+clean
```

64GB x 3 = 192GBなので、ちゃんとCephに割り当てられているようだ。

ボリュームを作ると消費されるのか？

```bash
ubuntu@maas:~$ openstack volume create vol5 --size 5
ubuntu@maas:~$ openstack volume list
+--------------------------------------+------+-----------+------+-------------+
| ID | Name | Status | Size | Attached to |
+--------------------------------------+------+-----------+------+-------------+
| 380ef166-06ef-439c-9c63-aa73a1b717e6 | vol5 | available | 5 | |
+--------------------------------------+------+-----------+------+-------------+
ubuntu@maas:~$ openstack volume show vol5
+--------------------------------+--------------------------------------+
| Field | Value |
+--------------------------------+--------------------------------------+
| attachments | [] |
| availability_zone | nova |
| bootable | false |
| consistencygroup_id | None |
| created_at | 2020-03-16T06:25:09.000000 |
| description | None |
| encrypted | False |
| id | 380ef166-06ef-439c-9c63-aa73a1b717e6 |
| migration_status | None |
| multiattach | False |
| name | vol5 |
| os-vol-host-attr:host | cinder@cinder-ceph#cinder-ceph |
| os-vol-mig-status-attr:migstat | None |
| os-vol-mig-status-attr:name_id | None |
| os-vol-tenant-attr:tenant_id | cf31276ca13d43cd807f675b3316dd29 |
| properties | |
| replication_status | None |
| size | 5 |
| snapshot_id | None |
| source_volid | None |
| status | available |
| type | __DEFAULT__ |
| updated_at | 2020-03-16T06:25:10.000000 |
| user_id | 9f2f1c16f334408d99ef87a796c2cbee |
+--------------------------------+--------------------------------------+
ubuntu@maas:~$ juju ssh ceph-mon/0 sudo ceph osd tree
ID CLASS WEIGHT TYPE NAME STATUS REWEIGHT PRI-AFF
-1 0.18750 root default
-5 0.06250 host node02
2 hdd 0.06250 osd.2 up 1.00000 1.00000
-7 0.06250 host node03
1 hdd 0.06250 osd.1 up 1.00000 1.00000
-3 0.06250 host node04
0 hdd 0.06250 osd.0 up 1.00000 1.00000
Connection to 192.168.122.58 closed.
ubuntu@maas:~$ juju ssh ceph-mon/0 sudo ceph -s
cluster:
  id: e5afd152-64ff-11ea-ac77-00163e698e83
  health: HEALTH_OK
services:
  mon: 3 daemons, quorum juju-bde0e9-3-lxd-0,juju-bde0e9-1-lxd-0,juju-bde0e9-2-lxd-0 (age 3h)
  mgr: juju-bde0e9-1-lxd-0(active, since 3h), standbys: juju-bde0e9-2-lxd-0, juju-bde0e9-3-lxd-0
  osd: 3 osds: 3 up (since 3h), 3 in (since 2d)
  rgw: 1 daemon active (juju-bde0e9-0-lxd-0)
data:
  pools: 17 pools, 82 pgs
  objects: 273 objects, 330 MiB
  usage: 4.0 GiB used, 188 GiB / 192 GiB avail
  pgs: 82 active+clean
Connection to 192.168.122.58 closed.
```

ボリュームを作っただけではCephは消費されない？

サーバにアタッチしてみると・・・

```bash
ubuntu@maas:~$ openstack server add volume bionic-1 vol5
ubuntu@maas:~$ juju ssh ceph-mon/0 sudo ceph -s
cluster:
  id: e5afd152-64ff-11ea-ac77-00163e698e83
  health: HEALTH_OK
services:
  mon: 3 daemons, quorum juju-bde0e9-3-lxd-0,juju-bde0e9-1-lxd-0,juju-bde0e9-2-lxd-0 (age 3h)
  mgr: juju-bde0e9-1-lxd-0(active, since 3h), standbys: juju-bde0e9-2-lxd-0, juju-bde0e9-3-lxd-0
  osd: 3 osds: 3 up (since 3h), 3 in (since 2d)
  rgw: 1 daemon active (juju-bde0e9-0-lxd-0)
data:
  pools: 17 pools, 82 pgs
  objects: 273 objects, 330 MiB
  usage: 4.0 GiB used, 188 GiB / 192 GiB avail
  pgs: 82 active+clean
```

サーバーにアタッチしても変わらず。

実際に使用してみると…

```bash
ubuntu@maas:~$ ssh -i .ssh/id_mykey ubuntu@192.168.123.154
sudo fdisk -l
sudo mkfs.ext4 /dev/vdb
sudo mkdir /mnt/vdb
sudo mount /dev/vdb /mnt/vdb
touch /mnt/vdb/test.txt
ubuntu@maas:~$ juju ssh ceph-mon/0 sudo ceph -s
cluster:
  id: e5afd152-64ff-11ea-ac77-00163e698e83
  health: HEALTH_OK
services:
  mon: 3 daemons, quorum juju-bde0e9-3-lxd-0,juju-bde0e9-1-lxd-0,juju-bde0e9-2-lxd-0 (age 3h)
  mgr: juju-bde0e9-1-lxd-0(active, since 3h), standbys: juju-bde0e9-2-lxd-0, juju-bde0e9-3-lxd-0
  osd: 3 osds: 3 up (since 3h), 3 in (since 2d)
  rgw: 1 daemon active (juju-bde0e9-0-lxd-0)
data:
  pools: 17 pools, 82 pgs
  objects: 320 objects, 480 MiB
  usage: 4.4 GiB used, 188 GiB / 192 GiB avail
  pgs: 82 active+clean
io:
  client: 2.8 KiB/s rd, 1.8 MiB/s wr, 3 op/s rd, 3 op/s wr
```

ファイルシステム作成して、実際にファイル作ってみて０．４Gぐらい消費された

### CPUの使用率

各ノード２コアずつで問題ないようだ

```bash
ubuntu@maas:~$ for i in 0 1 2 3 4 5; do juju ssh $i "uptime"; done;
02:33:48 up 1:14, 1 user, load average: 0.31, 0.30, 0.28
02:33:48 up 1:13, 1 user, load average: 0.03, 0.15, 0.17
02:33:49 up 1:14, 1 user, load average: 1.02, 0.59, 0.55
02:33:50 up 1:14, 1 user, load average: 0.19, 0.22, 0.27
ubuntu@maas:~$ for i in 0 1 2 3 4 5; do juju ssh $i "free -h"; done;
total used free shared buff/cache available
Mem: 3.9G 2.4G 243M 1.7M 1.2G 1.3G
Swap: 3.9G 76M 3.8G
total used free shared buff/cache available
Mem: 3.9G 2.2G 207M 1.8M 1.4G 1.7G
Swap: 3.9G 10M 3.8G
total used free shared buff/cache available
Mem: 3.9G 1.6G 819M 1.3M 1.4G 2.0G
Swap: 3.9G 268K 3.9G
total used free shared buff/cache available
Mem: 3.9G 2.0G 336M 2.0M 1.6G 1.7G
Swap: 3.9G 58M 3.8G
```

### ディスクの使用率

まずCephがVdbを使ってないし、ルートは２０Gぐらいしか使用されてない。OSのみで６GBほど使用するようだ。

```bash
ubuntu@maas:~$ for i in 0 1 2 3 4 5; do juju ssh $i "sudo df -h"; done;
Filesystem Size Used Avail Use% Mounted on
/dev/vda1 32G 5.8G 24G 20% /

/dev/vda1 32G 12G 18G 41% /
tmpfs 2.0G 24K 2.0G 1% /var/lib/ceph/osd/ceph-1

/dev/vda1 32G 13G 18G 41% /
tmpfs 2.0G 24K 2.0G 1% /var/lib/ceph/osd/ceph-0

/dev/vda1 32G 12G 19G 38% /

/dev/vda1 32G 15G 16G 48% /
tmpfs 2.0G 24K 2.0G 1% /var/lib/ceph/osd/ceph-2
```

### H/Aを試してみる

### ネットワークについて考察

```bash
$ sudo iptables -L
Chain INPUT (policy ACCEPT)
target prot opt source destination
ACCEPT udp -- anywhere anywhere udp dpt:domain
ACCEPT tcp -- anywhere anywhere tcp dpt:domain
ACCEPT udp -- anywhere anywhere udp dpt:bootps
ACCEPT tcp -- anywhere anywhere tcp dpt:bootps
ACCEPT udp -- anywhere anywhere udp dpt:domain
ACCEPT tcp -- anywhere anywhere tcp dpt:domain
ACCEPT udp -- anywhere anywhere udp dpt:bootps
ACCEPT tcp -- anywhere anywhere tcp dpt:bootps
Chain FORWARD (policy ACCEPT)
target prot opt source destination
ACCEPT all -- anywhere 192.168.122.0/24 ctstate RELATED,ESTABLISHED
ACCEPT all -- 192.168.122.0/24 anywhere
ACCEPT all -- anywhere anywhere
REJECT all -- anywhere anywhere reject-with icmp-port-unreachable
REJECT all -- anywhere anywhere reject-with icmp-port-unreachable
ACCEPT all -- anywhere anywhere
REJECT all -- anywhere anywhere reject-with icmp-port-unreachable
REJECT all -- anywhere anywhere reject-with icmp-port-unreachable
Chain OUTPUT (policy ACCEPT)
target prot opt source destination
ACCEPT udp -- anywhere anywhere udp dpt:bootpc
ACCEPT udp -- anywhere anywhere udp dpt:bootpc
```

KVMに設定した２つのネットワークはホスト側に出ている

```bash
$ sudo route  
カーネルIP経路テーブル  
受信先サイト ゲートウェイ ネットマスク フラグ Metric Ref 使用数 インタフェース  
default \_gateway 0.0.0.0 UG 100 0 0 enp5s0  
default \_gateway 0.0.0.0 UG 101 0 0 eno1  
link-local 0.0.0.0 255.255.0.0 U 1000 0 0 virbr1  
192.168.0.0 0.0.0.0 255.255.255.0 U 100 0 0 enp5s0  
192.168.0.0 0.0.0.0 255.255.255.0 U 101 0 0 eno1  
192.168.122.0 0.0.0.0 255.255.255.0 U 0 0 0 virbr0  
192.168.123.0 0.0.0.0 255.255.255.0 U 0 0 0 virbr1
```

それぞれVnetというブリッジに繋がっているようだ

```bash
$ sudo brctl show  
bridge name bridge id STP enabled interfaces  
virbr0 8000.525400590978 yes virbr0-nic  
vnet0  
vnet1  
vnet2  
vnet4  
vnet6  
vnet8  
virbr1 8000.52540006733a yes virbr1-nic  
vnet3  
vnet5  
vnet7  
vnet9
```

ip a で見てみると…

```bash
$ ip a
5: virbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether 52:54:00:06:73:3a brd ff:ff:ff:ff:ff:ff
inet 192.168.123.1/24 brd 192.168.123.255 scope global virbr1
6: virbr1-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr1 state
7: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
8: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state
9: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN group default qlen 1000
10: vnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN group default qlen 1000
11: vnet2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN group default qlen 1000
12: vnet3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr1 state UNKNOWN group default qlen 1000
13: vnet4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN group default qlen 1000
14: vnet5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr1 state UNKNOWN group default qlen 1000
15: vnet6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN group default qlen 1000
16: vnet7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr1 state UNKNOWN group default qlen 1000
17: vnet8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr0 state UNKNOWN group default qlen 1000
18: vnet9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master virbr1 state UNKNOWN group default qlen 1000
```

vnet\*がvirt-installで作成したVM用の各２個のNICのようだ

### MAASにCLIでログイン

```bash
ubuntu@maas:~$ maas login ubuntu http://192.168.122.99/MAAS/

ubuntu@maas:~$ maas list  
ubuntu http://192.168.122.99/MAAS/api/2.0/ j37ek9yRv3JKd946sP:JX2Wbgd6Vx86WdqheY:NcUHEcrc4aX3EYW9GKtcVW6UcwXhJyau
```

### MAASに登録されたVMののIPを探る旅（外部向けNIC名を知りたい）

新しいMAASはComissionに成功すればMachineのページから見られるので、これらの努力は必要ない。

Nodeを指定してMAAS情報を取り出す＝＞出ない

```bash
ubuntu@maas:~$ maas ubuntu nodes read
Success.
Machine-readable output follows:
[{"memory": 4096,
"raids": [],
"status_action": "",
"cpu_count": 2,
"address_ttl": null,
ubuntu@maas:~$ maas ubuntu machines read hostname=node01
Success.
Machine-readable output follows:
[{"virtualblockdevice_set": [],
"storage_test_status": 2,
"min_hwe_kernel": "",
"current_installation_result_id": null,
"tag_names": ["virtual","2nic"],
```

MAAS WEB => Machine => Log

出ない。

KVM経由でDHCPで貸し出されたIPを探す

```bash
@baremetal:~$ virsh list
Id Name State
----------------------------------------------------
1 desktop1804 running
2 node06 running
3 node05 running
4 node04 running
5 node03 running
6 node02 running
7 node01 running
8 maas running
9 juju running
@baremetal:~$ virsh net-list
Name State Autostart Persistent
----------------------------------------------------------
default active yes yes
@baremetal:~$ virsh net-info default
Name: default
UUID: 4357931e-cdd0-4894-9f4a-a6abb4a13ce7
Active: yes
Persistent: yes
Autostart: yes
Bridge: virbr0
@baremetal:~$ virsh net-dhcp-leases default
Expiry Time MAC address Protocol IP address Hostname Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
2020-02-19 18:29:34 52:54:00:70:32:4e ipv4 192.168.122.171/24 desktop1804 -
2020-02-19 18:15:16 52:54:00:bd:aa:71 ipv4 192.168.122.13/24 maas ff:b5:5e:67:ff:00:02:00:00:ab:11:83:f2:9d:8c:6e:be:e5:62
```

出ない。

通常のdhcp.leasesや/var/lib/maas/dhcp/dhcpd.leasesには入っておらず、このConfに記述があった。

```bash
ens3
ubuntu@maas:~$ sudo cat /var/lib/maas/dhcpd.conf
# node01-ens3
host 52-54-00-61-f2-32 {
## Node DHCP snippets
## No DHCP snippets defined for host
hardware ethernet 52:54:00:61:f2:32;
fixed-address 192.168.122.97;
NO IP from WEB because Machines are not Deployed yet!!!
```

ではNodeにログイン

ssh-addすると、MAASに登録されているMachineにパスなしでログインできるが、JujuをBootstrapしたMachineだけはMAASのMachine経由でないとログインできなかった。ただJujuのMachineにJujuコマンドが入っているわけではなく、単にJujuのサーバーなので、通常Jujuコマンドを発行するときはMAASにSSHするかローカル端末から行うので良しか。

```bash
ubuntu@node01:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
link/ether 52:54:00:61:f2:32 brd ff:ff:ff:ff:ff:ff
inet 192.168.122.97/24 brd 192.168.122.255 scope global ens3
```

上記から、IPがアサインされていないens10が外部向けのNICだと判明。

### 何か失敗してJujuControllerを削除したいとき

```bash
ubuntu@maas:~$ juju kill-controller maas-controller
ubuntu@maas:~$ juju cloud
ubuntu@maas:~$ juju remove-cloud maas-cloud
```

### JujuというVMをMAASの登録の中から探してタグ付け（マニュアル）

新しいMAASではGUIで簡単に判明するので必要ない

```bash
ubuntu@maas:~$ maas ubuntu tags create name=bootstrap
ubuntu@maas:~$ maas ubuntu tag nodes virtual | grep hostname
"hostname": "maas",
"hostname": "node01",
"hostname": "node02",
"hostname": "node03",
"hostname": "node05",
"hostname": "node04",
"hostname": "node06",
"hostname": "juju",
ubuntu@maas:~$ maas ubuntu nodes read hostname=juju | grep system.id
"system_id": "rky7dy",
ubuntu@maas:~$ maas ubuntu tag update-nodes bootstrap add=rky7dy
ubuntu@maas:~$ maas ubuntu tag nodes bootstrap
{"netboot": true,"tag_names": ["virtual","bootstrap"],
```

### JujuにUbuntuバージョンを指定してDeployさせる

素のマシンがほしい時に。あまり必要ないと思うが…。

```bash
ubuntu@maas:~$ juju add-machine --series="xenial" node01.maas
created machine 0
ubuntu@maas:~$ juju machines
Machine State DNS Inst id Series AZ Message
0 pending pending xenial
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2020/03/16/%e3%83%9e%e3%82%b7%e3%83%b3%ef%bc%91%e5%8f%b0%e3%81%a7openstack-ubuntu-18-04/).*
