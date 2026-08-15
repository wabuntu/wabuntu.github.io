---
layout: post
title: "OpenStackの最小ディスク容量"
date: 2020-03-24 00:00:00 +0900
lang: ja
---

OpenStackの全環境をバックアップできるようにしたいので、できるだけディスクの容量を抑えたい。一体各VM最小でどのくらい確保すればいいのか。VCPUは２、メモリは4GBで余裕があるくらいだと思う。

## MAAS

```
/dev/sda2 16G 11G 4.0G 74% /
```

16GBでちょうどいいくらいか

## juju controller

```
/dev/sda1 16G 7.2G 7.8G 49% /
```

おそらく12GBでもいける。

## 各ノード

```
ubuntu@maas:~$ for i in 0 1 2 3; do juju ssh $i df -h; done
/dev/sda1 32G 12G 19G 39% /
/dev/sda1 32G 13G 18G 41% /
/dev/sda1 32G 15G 16G 49% /
/dev/sda1 32G 14G 17G 46% /
```

16GBではギリなので24GB＋Ceph用を必要に応じて（24GB?）ぐらいか。

これらを加味すると、VM作成用コマンドはこのようになる。

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
#ipアドレスは192.168.122.2に。

NAME="controller"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--boot network,hd,menu=on --graphics spice --pxe \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-vda.qcow2,size=12,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:bb,model=virtio

NODE="01"
virt-install --name node${NODE} --vcpus 2 --ram 4096 --cpu host \
--boot network,hd,menu=on --graphics spice --pxe \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/node${NODE}-vda.qcow2,size=24,format=qcow2,bus=scsi \
--disk path=/var/lib/libvirt/images/node${NODE}-vdb.qcow2,size=24,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:00:01:${NODE},model=virtio \
--network bridge=virbr1,mac=52:54:00:00:02:${NODE},model=virtio
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2020/03/24/openstack%e3%81%ae%e6%9c%80%e5%b0%8f%e3%83%87%e3%82%a3%e3%82%b9%e3%82%af%e5%ae%b9%e9%87%8f/).*
