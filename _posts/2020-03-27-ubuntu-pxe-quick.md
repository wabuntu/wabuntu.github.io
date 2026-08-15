---
layout: post
title: "UbuntuでざっくりPXE"
date: 2020-03-27 00:00:00 +0900
lang: ja
---

このブログ投稿は、Ubuntuでの基本的なPXE（Preboot Execution Environment）セットアップについて説明しています。

## セットアップ手順

```bash
ubuntu@ubuntu:~$ sudo apt install isc-dhcp-server
```

DHCP設定ファイルは次のようになります：

```conf
subnet 192.168.122.0 netmask 255.255.255.0 {
   range 192.168.122.100 192.168.122.199;
   option domain-name-servers 192.168.122.1;
   option domain-name "ubuntu.com";
   option routers 192.168.122.1;
   option broadcast-address 192.168.122.255;
   filename "/pxelinux.0";
}
```

TFTPサーバーのインストールと設定：

```bash
ubuntu@ubuntu:~$ sudo apt install tftpd-hpa
```

```conf
# /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure -vvv"
RUN_DAEMON="yes"
```

ネットブートイメージの取得とサービス開始：

```bash
ubuntu@ubuntu:~$ sudo systemctl restart tftpd-hpa.service
ubuntu@ubuntu:~$ wget http://archive.ubuntu.com/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/netboot.tar.gz
ubuntu@ubuntu:~$ sudo systemctl start isc-dhcp-server.service
```

NFSサーバーのセットアップ：

```bash
ubuntu@ubuntu:~$ sudo apt install nfs-kernel-server
ubuntu@ubuntu:~$ wget http://ftp.riken.jp/Linux/ubuntu-releases/18.04.4/ubuntu-18.04.4-desktop-amd64.iso
ubuntu@ubuntu:~$ sudo mkdir /tmp/nfs
ubuntu@ubuntu:~$ sudo mount -t iso9660 -o loop ubuntu-18.04.4-desktop-amd64.iso /tmp/nfs
ubuntu@ubuntu:~$ systemctl restart nfs-kernel-server
```

PXEブートメニュー設定ファイル：

```cfg
default live
label live
   menu label ^Livelinux
   menu default
   kernel ubuntu-installer/amd64/linux
   append root=/dev/nfs boot=casper netboot=nfs nfsroot=192.168.122.200 vga=788 initrd=ubuntu-installer/amd64/initrd.gz quiet splash ---
label install
	menu label ^Install
	menu default
	kernel ubuntu-installer/amd64/linux
	append vga=788 initrd=ubuntu-installer/amd64/initrd.gz --- quiet 
label cli
	menu label ^Command-line install
	kernel ubuntu-installer/amd64/linux
	append tasks=standard pkgsel/language-pack-patterns= pkgsel/install-language-support=false vga=788 initrd=ubuntu-installer/amd64/initrd.gz --- quiet
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2020/03/27/ubuntu%e3%81%a7%e3%81%96%e3%81%a3%e3%81%8f%e3%82%8apxe/).*
