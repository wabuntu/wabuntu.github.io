---
layout: post
title: "USB自体にLinuxをインストール"
date: 2016-03-10 00:00:00 +0900
lang: ja
---

ラップトップのHDD故障→交換→再インストール→電源無限入切発生→メーカー修理→再インストール

こんな経験から、LinuxをUSBキーにインストールすることにしました。個人的なメリットとしては

- ディスクごとバックアップが取りやすい
- PCが壊れても他のPCで使える
- VirtualBoxと比べてWebカメラ等の動作確率が良い

こんなものがあります。なお大きなデメリットとしては

- USBそのものがHDDだと、動作がかなりもたつく

というのがありますので、今のところこの方法がベストかは何とも言えません・・・。

参考：

[http://bellbind.tumblr.com/post/45757405213/usb-hdd%E3%81%ABmbrefi%E4%B8%A1%E3%83%96%E3%83%BC%E3%83%88%E5%8F%AF%E8%83%BD%E3%81%AAubuntu%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E4%BD%9C%E3%82%8B%E6%96%B9%E6%B3%95](http://bellbind.tumblr.com/post/45757405213/usb-hdd%E3%81%ABmbrefi%E4%B8%A1%E3%83%96%E3%83%BC%E3%83%88%E5%8F%AF%E8%83%BD%E3%81%AAubuntu%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%82%92%E4%BD%9C%E3%82%8B%E6%96%B9%E6%B3%95)

## 準備するもの

- 作業用PC（LinuxDesktop可）
- USBキー

極力速いものを選んだほうがいいみたい。私はこれにしたけど、結構もたつく。

なお、このセットアップはVirtualBoxを使うことを絶対おすすめします。私は作業PCから直接やったらそっちのMBRだかgrubまでおかしくなってUSBも作業PCも起動しなくなりました・・・。

## VirtualBoxインストール

- Ubuntuのソフトウエアセンターからのものより、本家サイトからダウンロードしたものの方が良かった（Extention Packが入らなかったりした）
- インストール後はsudo virtualboxで起動したらすんなりUSBも出てきた（rootの方がいいという事？）

## VirtualBox仮想マシン作成

- タイプ: Linux Ubuntu 64bit「仮想ハードドライブを追加しない」で作成
- システム: 「EFIを有効化」にチェックを入れない(MBRブートさせる)
- ストレージ: IDE「CD/DVDドライブ」の「仮想ディスクファイル」として、インストーラISOファイル
- ネットワーク: アダプタ1「NAT」で高度で「ケーブル接続」にチェックを入れておく
- USB: 「USBデバイスフィルタ」にインストールするUSB

## VirtualBox仮想マシン起動

そのままUbuntuインストールに進み、手動でディスク構成をする

私の環境ではsdaは作業PCに入ってるHDD、sdbはUSBになってた。

- /dev/sdb1 fat16 100MB
- /dev/sdb2 ext4 /
- /dev/sdb3 swap 1.5G
- ブートローダーは/dev/sdbに

あとは普通にインストール。私の場合ホームディレクトリを暗号化した（LVM等はなし）

## USBキーから起動する

```bash
sudo chmod -x /etc/grub.d/30_os-prober
sudo apt-get update; sudo apt-get -u dist-upgrade
sudo apt-get install grub-efi-amd64-bin

grub-mkimage -d /usr/lib/grub/x86_64-efi/ -o BOOTx64.EFI -O x86_64-efi -p "" \
part_gpt part_msdos ntfs ntfscomp hfsplus fat ext2 \
normal chain boot configfile linux multiboot

cd; mkdir efi;
sudo mount /dev/sdb1 efi;
sudo mkdir -p efi/EFI/BOOT;
sudo cp BOOTx64.EFI efi/EFI/BOOT/;
sudo cp -r /usr/lib/grub/x86_64-efi efi/EFI/BOOT/;

sudo vi efi/EFI/BOOT/grub.cfg;
# configfile (hd0,msdos2)/boot/grub/grub.cfg

sudo umount efi
```

## スピード対策を施す

体感としてはかなりもたつきが軽減されたように感じます

参考：

[http://smiyaz.cocolog-nifty.com/blog/2015/01/usbde.html](http://smiyaz.cocolog-nifty.com/blog/2015/01/usbde.html)

```bash
sudo vi /etc/NetworkManager/dnsmasq.d/cache
#cache-size=100

sudo vi /etc/fstab
#tmpfs /tmp tmpfs defaults,noatime 0 0
#tmpfs /var/tmp tmpfs defaults,noatime 0 0

sudo vi /etc/sysctl.conf
#vm.swappiness = 0

sudo vi /etc/fstab
#/dev/sdb2 / ext4 defaults,async,noatime,nodiratime,data=writeback,barrier=0 1 1

tune2fs -o journal_data_writeback /dev/sdb2
lsblk
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/10/usb%e8%87%aa%e4%bd%93%e3%81%ablinux%e3%82%92%e3%82%a4%e3%83%b3%e3%82%b9%e3%83%88%e3%83%bc%e3%83%ab/).*
