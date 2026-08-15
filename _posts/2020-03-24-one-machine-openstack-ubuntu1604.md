---
layout: post
title: "一台でOpenStack（Ubuntu16.04)"
date: 2020-03-24 00:00:00 +0900
lang: ja
---

Xenialも一台構成でOpenStackインストールをしてみました。

## KVMネットワーク設定

### virbr0

1. 初期画面でQEMU/KVMを右クリック→詳細→仮想ネットワーク
2. 既存のやつを削除して新しくdefaultを作り直す
3. 新規作成のステップ２でDHCPv4をOFF、サブネットは192.168.122.0にした
4. IPv6は無視
5. ステップ４の物理ネットワークにフォワードで、いずれかのデバイス、NATを選ぶ

### virbr1

1. 上記同様にvirbr1名義で追加のネットワークを作成（VMはNICが２つずつあるので、もう一個用だが実際には使わない）。物理ネットワークへの接続は隔離された仮想ネットワークを選択。

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

## MAAS

### VMを作成

```bash
cd /var/lib/libvirt/images
wget ubuntu-16.04.6-server-amd64.iso

NAME="maas"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--boot hd,menu=on --graphics spice \
--os-type linux --os-variant ubuntu16.04 \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=16,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:aa,model=virtio \
--cdrom=ubuntu-16.04.6-server-amd64.iso
```

ipアドレスは192.168.122.2に。

今回何も予備設定しないでやってみる

```bash
$ sudo apt install maas
```

いきなりここにアクセスしてみたら、Admin作れと表示

[http://192.168.122.2/MAAS/](http://192.168.122.2/MAAS/)

```bash
ubuntu@maas:~$ sudo maas createadmin
Username: maasadmin
Password: 
Again: 
Email: 
Import SSH keys [] (lp:user-id or gh:user-id):
```

作ったらそのユーザーでログイン。  
イメージはデフォで16.04が選ばれているのでDNSだけ1.1.1.1にして続行。  
Subnetタブからuntaggedを選んでProvide DHCP。

```bash
$ sudo apt install libvirt-bin
```

## Juju Controller

Jujuは仕様で最低3584MBのメモリが必要

```bash
NAME="controller"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--boot network,hd,menu=on --graphics spice --pxe \
--os-type linux --os-variant ubuntu16.04 \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=16,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:bb,model=virtio
```

コマンド終了とともにVMが起動、勝手にインストールが始まる。  
ちなみに–os-variantをつけないとパフォーマンス低下するよと怒られた。  
一度目にmaas-enlistingみたいなプロンプトが出て、cloud-initが走り、再起動してもう一度インストールが走る。  
その後maasに自動登録された名前で一瞬プロンプトが出て自動で電源が切れる。  
MAASのNodesに新しいマシンが追加されるので、Overviewから下記を設定。

どうもComissionが失敗する状態が続いた。  
Virt managerの画面を見ていると、http://archive.ubuntu.com/ubuntuのどこかのアクセスで403　Forbiddenが出ているようだ。  
手動でアクセスしてみても特にエラーはないので、MAASのSettingからProxyでDon't use a proxyを選んだら成功した。

- Power type Virsh
- address qemu+ssh://wabuntu@192.168.0.5/system
- Virsh password (optional)
- Virsh VM ID controller

maasにlibvirt-binが入ってないと言われたのでインストール。  
名前をcontroller.maas、タグにbootstrapをつける。  
そしてComission。登録時と同様にcloud-initが走る

MAASに戻ってJujuのツールを導入

```bash
$ sudo apt install juju

ubuntu@maas:~$ cat maas-cloud.yaml 
clouds: # clouds key is required.
maas-cloud: # cloud's name
type: maas
auth-types: [oauth1]
endpoint: http://192.168.122.2:5240/MAAS

ubuntu@maas:~$ juju add-cloud maas-cloud maas-cloud.yaml
ubuntu@maas:~$ juju clouds
Cloud Regions Default Type Description
....
maas-cloud 0 maas Metal As A Service

ubuntu@maas:~$ juju add-credential maas-cloud
Enter credential name: maas-cloud-creds
Using auth-type "oauth1".
Enter maas-oauth: 
Credentials added for cloud maas-cloud.

ubuntu@maas:~$ juju credentials
Cloud Credentials
maas-cloud maas-cloud-creds

$ juju bootstrap maas-cloud maas-controller

ubuntu@maas:~$ juju gui
GUI 2.15.0 for model "admin/default" is enabled at:
https://192.168.122.3:17070/gui/u/admin/default
Your login credential is:
username: admin
password: f2db6624bbcc2dacf08e71819a8a374a
ubuntu@maas:~$ juju change-user-password admin
new password: 
type new password again: 
Your password has been changed.
```

上記アドレスでJujuのGUIにログインできる（パスワードは簡単なものに変更）

そしてBootstrap。

## ノード

下記を01から04まで実行

```bash
NODE="01"
virt-install --name node${NODE} --vcpus 3 --ram 6144 --cpu host \
--boot network,hd,menu=on --graphics spice --pxe \
--os-type linux --os-variant ubuntu16.04 \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/node${NODE}-sda.qcow2,size=24,format=qcow2,bus=scsi \
--disk path=/var/lib/libvirt/images/node${NODE}-sdb.qcow2,size=24,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:${NODE},model=virtio \
--network bridge=virbr1,mac=52:54:00:00:02:${NODE},model=virtio
```

sdbを24GBにしているので、実際のボリュームは20GBぐらいまでしか作れない点に注意。  
４台ともComissionしてReady状態にする。

## OpenStack

### デプロイ

```bash
sudo apt install charm
ubuntu@maas:~$ charm pull ~openstack-charmers-next/bundle/openstack-base-xenial-mitaka ~/bundle
ubuntu@maas:~$ cd bundle/
ubuntu@maas:~/bundle$ vi bundle.yaml
```

```yaml
neutron-gateway:
annotations:
gui-x: '0'
gui-y: '0'
charm: cs:~openstack-charmers-next/xenial/neutron-gateway
num_units: 1
options:
bridge-mappings: physnet1:br-ex
data-port: br-ex:ens4
worker-multiplier: 0.25
to:
- '0'

nova-cloud-controller:
annotations:
gui-x: '0'
gui-y: '500'
charm: cs:~openstack-charmers-next/xenial/nova-cloud-controller
num_units: 1
options:
network-manager: Neutron
worker-multiplier: 0.25
console-access-protocol: spice
to:
- lxd:2
```

```bash
ubuntu@maas:~/bundle$ juju switch default
ubuntu@maas:~/bundle$ juju deploy ./bundle.yaml
$ watch -c juju status --color
```

MAASで確保しているのは.122の後ろの５０個ぐらいだが、  
Jujuの各サービスは.122の前半のIPを使うようだ。  
マシンに一番若いIPを当てて、それ以降をLXDが取っていく感じで.4から.19まで。  
同じサブネットにはDHCPは一つ（MAAS）だろうから、Jujuは固定でIPをふっているという事か。

Horizonのパスワードはこれで

```bash
juju run –unit keystone/0 leader-get admin_passwd
```

### インスタンスの作成

どうもOcataのリポは必要

```bash
sudo add-apt-repository cloud-archive:ocata -y
sudo apt install python-novaclient python-keystoneclient python-glanceclient \
python-neutronclient python-openstackclient -y

ubuntu@maas:~/bundle$ source openrc
ubuntu@maas:~/bundle$ openstack catalog list
+----------+--------------+-------------------------------------------------------------------------------+
| Name | Type | Endpoints |
+----------+--------------+-------------------------------------------------------------------------------+
| nova | compute | RegionOne |
| | | publicURL: http://192.168.122.16:8774/v2/bd50a4d466984fe5b4c9991b6ea9529a |
| | | internalURL: http://192.168.122.16:8774/v2/bd50a4d466984fe5b4c9991b6ea9529a |
| | | adminURL: http://192.168.122.16:8774/v2/bd50a4d466984fe5b4c9991b6ea9529a |

curl http://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img | \
openstack image create --public --container-format=bare --disk-format=qcow2 xenial

ubuntu@maas:~/bundle$ ./neutron-ext-net --network-type flat -g 192.168.123.1 -c 192.168.123.0/24 -f 192.168.123.100:192.168.123.200 ext_net
INFO:root:Configuring external network 'ext_net'
INFO:root:Creating new external network definition: ext_net
INFO:root:New external network created: 11c206ef-dc90-41f2-86f0-156e9a694250
INFO:root:Creating new subnet for ext_net
INFO:root:New subnet created: ed6c4287-b546-4213-b27b-97e39427309c
INFO:root:Creating provider router for external network access
INFO:root:New router created: ade2dea6-398d-4be8-97e6-9988d765f22a
INFO:root:Plugging router into ext_net
INFO:root:Router connected to ext_net
```

ルーターまで作ってくれるらしい。インターフェースも繋がっている。

```bash
ubuntu@maas:~/bundle$ openstack network list
+--------------------------------------+---------+--------------------------------------+
| ID | Name | Subnets |
+--------------------------------------+---------+--------------------------------------+
| 11c206ef-dc90-41f2-86f0-156e9a694250 | ext_net | ed6c4287-b546-4213-b27b-97e39427309c |
+--------------------------------------+---------+--------------------------------------+
ubuntu@maas:~/bundle$ openstack subnet list
+--------------------------------------+----------------+--------------------------------------+------------------+
| ID | Name | Network | Subnet |
+--------------------------------------+----------------+--------------------------------------+------------------+
| ed6c4287-b546-4213-b27b-97e39427309c | ext_net_subnet | 11c206ef-dc90-41f2-86f0-156e9a694250 | 192.168.123.0/24 |
+--------------------------------------+----------------+--------------------------------------+------------------+
ubuntu@maas:~/bundle$ openstack router list
+--------------------------------------+-----------------+--------+-------+-------------+-------+----------------------------------+
| ID | Name | Status | State | Distributed | HA | Project |
+--------------------------------------+-----------------+--------+-------+-------------+-------+----------------------------------+
| ade2dea6-398d-4be8-97e6-9988d765f22a | provider-router | ACTIVE | UP | False | False | bd50a4d466984fe5b4c9991b6ea9529a |
+--------------------------------------+-----------------+--------+-------+-------------+-------+----------------------------------+
ubuntu@maas:~/bundle$ openstack router show provider-router
+-------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field | Value |
+-------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up | UP |
| availability_zone_hints | |
| availability_zones | nova |
| description | |
| distributed | False |
| external_gateway_info | {"network_id": "11c206ef-dc90-41f2-86f0-156e9a694250", "enable_snat": true, "external_fixed_ips": [{"subnet_id": "ed6c4287-b546-4213-b27b-97e39427309c", "ip_address": "192.168.123.100"}]} |
| ha | False |
| id | ade2dea6-398d-4be8-97e6-9988d765f22a |
| name | provider-router |
| routes | [] |
| status | ACTIVE |
| tenant_id | bd50a4d466984fe5b4c9991b6ea9529a |
+-------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

これで内部と外部のネットワークもつながる

```bash
ubuntu@maas:~/bundle$ ./neutron-tenant-net -t admin -r provider-router \
> internal 10.5.5.0/24
INFO:root:Creating tenant network: internal
INFO:root:Creating subnet for internal
INFO:root:Adding interface from provider-router to internal_subnet
```

フレーバーはすでにある。SmallのDiskを８GBに変更した。

```bash
ubuntu@maas:~/bundle$ openstack flavor list
+----+-----------+-------+------+-----------+-------+-----------+
| ID | Name | RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+-----------+-------+------+-----------+-------+-----------+
| 1 | m1.tiny | 512 | 1 | 0 | 1 | True |
| 2 | m1.small | 2048 | 20 | 0 | 1 | True |
| 3 | m1.medium | 4096 | 40 | 0 | 2 | True |
| 4 | m1.large | 8192 | 80 | 0 | 4 | True |
| 5 | m1.xlarge | 16384 | 160 | 0 | 8 | True |
+----+-----------+-------+------+-----------+-------+-----------+
```

キーを作成して登録。親マシン（KVMホスト）からの接続になるのでそちらをつかう。

```bash
nova keypair-add --pub-key ~/.ssh/authorized_key mykey
```

### インスタンス作成

```bash
nova boot --image xenial --flavor m1.small --key-name mykey \
--nic net-id=$(neutron net-list | grep internal | awk '{ print $2 }') \
xenial-test
```

セキュリティグループの作成（２つ出てくるので２回やること）

```bash
$ neutron security-group-list
$ neutron security-group-rule-create --protocol icmp \
--direction ingress f188f73a-4d3e-499b-86d8-56c24c5ad565
$ neutron security-group-rule-create --protocol tcp \
--port-range-min 22 --port-range-max 22 \
--direction ingress f188f73a-4d3e-499b-86d8-56c24c5ad565
```

フローティングIPの作成とアタッチ

（古い方法？）

```bash
ubuntu@maas:~$ neutron floatingip-create ext_net
ubuntu@maas:~$ neutron floatingip-list
+--------------------------------------+------------------+---------------------+--------------------------------------+
| id | fixed_ip_address | floating_ip_address | port_id |
+--------------------------------------+------------------+---------------------+--------------------------------------+
| 148900f2-cce4-4f71-95b6-807d97a13bdd | 10.5.5.3 | 192.168.123.101 | 2e4e63ef-efd5-4f0c-a559-6c406a661d0d |
+--------------------------------------+------------------+---------------------+--------------------------------------+
ubuntu@maas:~$ neutron port-list 
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+
| id | name | mac_address | fixed_ips |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+
| 203a31ea-2c6b-4e88-99e7-b1bd70f5a44d | | fa:16:3e:0e:c9:db | {"subnet_id": "ed6c4287-b546-4213-b27b-97e39427309c", "ip_address": "192.168.123.100"} |
| 2e4e63ef-efd5-4f0c-a559-6c406a661d0d | | fa:16:3e:4a:09:e7 | {"subnet_id": "4dd4728f-e9c6-4d80-a64d-1487ce6ad275", "ip_address": "10.5.5.3"} |
| 3816e15f-a027-4771-8d79-fd6686c18519 | | fa:16:3e:2d:59:9f | {"subnet_id": "4dd4728f-e9c6-4d80-a64d-1487ce6ad275", "ip_address": "10.5.5.2"} |
| 7edcd94f-f7cb-4532-8de6-fab3fa76f078 | | fa:16:3e:8b:f2:a7 | {"subnet_id": "4dd4728f-e9c6-4d80-a64d-1487ce6ad275", "ip_address": "10.5.5.1"} |
| f720900e-4c1e-4afb-a040-7a4d5950092c | | fa:16:3e:31:8f:2f | {"subnet_id": "ed6c4287-b546-4213-b27b-97e39427309c", "ip_address": "192.168.123.101"} |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------+

ubuntu@maas:~$ neutron floatingip-associate 148900f2-cce4-4f71-95b6-807d97a13bdd 2e4e63ef-efd5-4f0c-a559-6c406a661d0d
Associated floating IP 148900f2-cce4-4f71-95b6-807d97a13bdd
```

（新しい方法？）

```bash
ubuntu@maas:~$ openstack floating ip create ext_net
ubuntu@maas:~$ openstack server add floating ip xenial_test 192.168.123.102
```

あとは.123へのルートがある親（KVMホスト）からSSH

```bash
$ ssh ubuntu@192.168.123.102
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2020/03/24/%e4%b8%80%e5%8f%b0%e3%81%a7openstack%ef%bc%88ubuntu16-04/).*
