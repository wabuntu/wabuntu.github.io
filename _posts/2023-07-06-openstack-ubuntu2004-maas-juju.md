---
layout: post
title: "OpenStack with Ubuntu 20.04 by MAAS + Juju"
date: 2023-07-06 00:00:00 +0900
lang: ja
---

## VIRT

```xml
<network>
  <name>virbr0</name>
  <uuid>91ca6d8f-7bfa-42a8-9a0e-9bb83040ed61</uuid>
  <forward dev="enp5s0" mode="nat">
    <nat>
      <port start="1024" end="65535"/>
    </nat>
    <interface dev="enp5s0"/>
  </forward>
  <bridge name="virbr0" stp="on" delay="0"/>
  <mac address="52:54:00:97:6e:14"/>
  <domain name="virbr0"/>
  <ip address="192.168.100.1" netmask="255.255.255.0">
  </ip>
</network>


<network>
  <name>virbr1</name>
  <uuid>50066c80-71d1-47ad-a930-ef93e0bbb4ac</uuid>
  <bridge name="virbr1" stp="on" delay="0"/>
  <mac address="52:54:00:45:91:d0"/>
  <domain name="virbr1"/>
  <ip address="192.168.101.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.101.128" end="192.168.101.254"/>
    </dhcp>
  </ip>
</network>
```

## MAAS

```bash
NAME="maas"
virt-install --name ${NAME} --vcpus 2 --ram 4096 \
--boot hd,menu=on --graphics spice \
--os-type linux --os-variant ubuntu22.04 \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}-sda.qcow2,size=40,format=qcow2,bus=scsi \
--network bridge=virbr0,mac=52:54:00:aa:aa:11,model=virtio \
--cdrom=/var/lib/libvirt/images/ubuntu-22.04.2-live-server-amd64.iso
```

```bash
sudo snap install maas
sudo snap install maas-test-db
sudo maas init region+rack --database-uri maas-test-db:///
```

```bash
ubuntu@maas:~$ sudo maas createadmin
Username: admin
Password: 
Again: 
Email: wabuntu@wabuntu.com
Import SSH keys [] (lp:user-id or gh:user-id): 
```

Access the dashboard at `http://192.168.100.2:5240/MAAS/`

- Upload your id_rsa.pub
- Enable DHCP at http://192.168.100.2:5240/MAAS/r/vlan/

### Storage Issue Resolution

Filesystem allocation was only half the available size:

```bash
ubuntu@maas:~$ sudo df -h
[sudo] password for ubuntu: 
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              392M  1.2M  391M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   19G  6.7G   11G  38% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/vda2                          2.0G  130M  1.7G   8% /boot
tmpfs                              392M  4.0K  392M   1% /run/user/1000
```

Extend the logical volume:

```bash
ubuntu@maas:~$ sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  Size of logical volume ubuntu-vg/ubuntu-lv changed from <19.00 GiB (4863 extents) to <38.00 GiB (9727 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
ubuntu@maas:~$ sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Verify the expansion:

```bash
ubuntu@maas:~$ sudo df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              392M  1.2M  391M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   38G  6.7G   29G  19% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/vda2                          2.0G  130M  1.7G   8% /boot
tmpfs                              392M  4.0K  392M   1% /run/user/1000
```

Generate API key:

```bash
ubuntu@maas:~$ sudo maas apikey --username admin > api.key
```

## NODES

```bash
#!/bin/bash

if [ $# -ne 2 ]; then
  echo "usage: $0 NAME NUMBER" 1>&2
  exit 1
fi
NAME=${1}
NODE=${2}

date;

virt-install --name ${NAME}${NODE} --vcpus 3 --ram 6144 \
--os-type=linux --os-variant ubuntu22.04 \
--cpu host-model \
--pxe --boot network,hd,menu=on --graphics spice \
--controller scsi,model=virtio-scsi,index=0 \
--disk path=/var/lib/libvirt/images/${NAME}${NODE}-vda.qcow2,size=32,format=qcow2 \
--disk path=/var/lib/libvirt/images/${NAME}${NODE}-vdb.qcow2,size=64,format=qcow2 \
--network bridge=virbr0,mac=52:54:00:00:86:${NODE},model=virtio \
--network bridge=virbr1,mac=52:54:00:00:87:${NODE},model=virtio
```

Create VMs:

```bash
sudo ./createvm.sh node 01
```

### Machine Configuration

Add Machine in MAAS:
- MAC: 52:54:00:00:86:01
- Power Type: Virsh
- Address: qemu+ssh://wabuntu@192.168.100.1/system

Commission the machine.

Repeat for node02, node03, and node04.

Add the "bootstrap" tag to machines in MAAS.

## JUJU

Reference: https://juju.is/docs/olm/maas

Add MAAS cloud:

```bash
ubuntu@juju:~$ juju add-cloud --local
Select cloud type: maas
Enter a name for your maas cloud: maas-cloud
Enter the API endpoint url: http://192.168.100.2:5240/MAAS
```

View available clouds:

```bash
ubuntu@juju:~$ juju clouds --local
Cloud       Regions  Default    Type  Credentials  Source    Description
localhost   1        localhost  lxd   0            built-in  LXD Container Hypervisor
maas-cloud  1        default    maas  0            local     Metal As A Service
```

Add credentials:

```bash
ubuntu@juju:~$ juju add-credential maas-cloud
Select region [any region, credential is not region specific]: 
Using auth-type "oauth1".
Enter maas-oauth: 
```

View credentials:

```bash
ubuntu@juju:~$ juju credentials
Cloud       Credentials
maas-cloud  maasapikey
```

Bootstrap the controller:

```bash
ubuntu@juju:~$ juju bootstrap maas-cloud
```

Check machines:

```bash
ubuntu@juju:~$ juju machines
Machine  State  Address  Inst id  Series  AZ  Message
```

Check status:

```bash
ubuntu@juju:~$ juju status
Model    Controller          Cloud/Region        Version  SLA          Timestamp
default  maas-cloud-default  maas-cloud/default  2.9.43   unsupported  05:46:14Z
```

Access the GUI:

```bash
ubuntu@juju:~$ juju gui
Dashboard 0.8.1 for controller "maas-cloud-default" is enabled at:
  https://192.168.100.4:17070/dashboard
Your login credential is:
  username: admin
  password: 
```

Reference: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-juju.html

Create and switch to a model:

```bash
juju add-model penstack
ubuntu@maas:~$ juju switch openstack
ubuntu@maas:~$ juju download openstack-base
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/07/06/openstack-with-ubuntu-20-04-by-maas-juju/).*
