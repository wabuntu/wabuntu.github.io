---
layout: post
title: "Linuxはupstartからsystemdに移行しはじめてるらしい"
date: 2016-03-15 00:00:00 +0900
lang: ja
---

たまにCentOSを使ったりすると混乱してきて、私は結局サービスの起動停止は/etc/init.dで直接やるようになりました。そんなわけでちょっと時代に追いついてみようとしましたが・・・。

## そもそもupstartを知らなかった

ssh.confなんかもここに入ってるようだ。・・・しばらくLinux触らない間にこんなことに！

```bash
$ ls /etc/init/
acpid.conf                      mountall-bootclean.sh.conf        rc.conf
apport.conf                     mountall.conf                     rcS.conf
atd.conf                        mountall-net.conf                 rc-sysinit.conf
bootmisc.sh.conf                mountall-reboot.conf              resolvconf.conf
cgmanager.conf                  mountall-shell.conf               rsyslog.conf
cgproxy.conf                    mountall-sh.conf                  setvtrgb.conf
checkfs.sh.conf                 mountdevsubfs.sh.conf             shutdown.conf
checkroot-bootclean.sh.conf     mounted-debugfs.conf              ssh.conf
```

こういう設定でランレベルと連携しているようだ

```bash
$ cat /etc/init/ssh.conf 
description    "OpenSSH server"

start on runlevel \[2345\]
stop on runlevel \[!2345\]
```

使い方はこんな感じらしい

```bash
$ sudo stop ssh
$ sudo start ssh
$ sudo reload ssh
$ sudo restart ssh
$ sudo status ssh
> ssh start/running, process 887
```

以前自分で/etc/rc?.d/にS99とか番号つけてやったりしてましたが、もう過去の遺物なんですね・・・。一覧はこうやって見るそうな

```bash
$ sudo initctl list
```

## そしてSystemdへ

Systemdになると設定ファイルはここになるそうな

```bash
$ cat /lib/systemd/system/ssh.service 
\[Unit\]
Description=OpenBSD Secure Shell server
After=network.target auditd.service
```

キモはこのAfter=で、このターゲットが立ち上がった後で無いとダメよ、という事らしい。じゃあnetwork.targetはここにあるのかな、と思いきや無い。

```bash
$ ls /lib/systemd/system/
acpid.service multi-user.target.wants ssh@.service systemd-udevd.service
acpid.socket polkitd.service ssh.socket systemd-udev-settle.service
atd.service rsync.service sudo.service systemd-udev-trigger.service
dbus.service rsyslog.service sysinit.target.wants udev.service
dbus.socket sockets.target.wants systemd-udevd-control.socket wpa_supplicant.service
dbus.target.wants ssh.service systemd-udevd-kernel.socket
```

おまけに*.target.wantsというよくわからんものもある。Ubuntu16.04ではこの方式に変わるそうなのでもうすこし勉強が必要・・・。

コマンドはこのようなものが使えるようだ

```bash
$ systemctl list-units
$ sudo systemctl enable httpd
$ sudo systemctl disable httpd
$ sudo systemctl start httpd
$ sudo systemctl show httpd
$ sudo systemctl stop httpd
$ sudo systemctl reload httpd
$ sudo systemctl restart httpd
$ sudo systemctl kill httpd
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/15/linux%e3%81%afupstart%e3%81%8b%e3%82%89systemd%e3%81%ab%e7%a7%bb%e8%a1%8c%e3%81%97%e3%81%af%e3%81%98%e3%82%81%e3%81%a6%e3%82%8b%e3%82%89%e3%81%97%e3%81%84/).*
