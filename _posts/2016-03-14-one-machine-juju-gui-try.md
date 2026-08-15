---
layout: post
title: "一台構成でjuju-guiを使ってみる"
date: 2016-03-14 00:00:00 +0900
lang: ja
---

とにかくJujuを試してみたい、Openstackがどういうものなのか触ってみたい。しかし家にはサーバーマシンはなく、あるのはデスクトップのみ・・・そんな状態でどこまでできるか試してみました

## マシン構成

### juju用 Desktop

割と一般家庭にあるようなレベルのPCを使いました

- マシン名: tt
- M/B: ASUSTeK Intel Z97 MAXIMUS VII RANGER（ありがとう台湾！）
- CPU: Core-i5-4460 BX80646I54460
- Mem: 8G
- Disk: 128GSSD + 2THDD
- NIC: 一個

### 作業マシン

- マシン名: nuc

## 初期設定

### BIOS

- 基本的にデフォ（UEFIはおそらく効いてる）
- Intel Virtualization Technology: Enabled
- VT-d: Enabled（このマザボは上記に追加してこれもあった）

### サーバーのインストール

- OS: 14.04.4 LTS ※インストール中にCDROMが見つからないというエラーが出たが、インストール用のUSBを純粋にddコマンドで作ったらできた
- SSD 128G: 120G:/ ext4 + 8G:Swap
- HDD 2T: /data xfs
- Service: SSH serverのみ

### インストールコマンド例

```bash
dd if=*.iso of=/dev/sdb1 bs=16M
```

## 参考ページ

- https://jujucharms.com/docs/stable/getting-started
- https://jujucharms.com/docs/stable/config-local
- https://jujucharms.com/docs/stable/config-LXC
- https://github.com/juju/cheatsheet

## 諸注意

- Note: If your home directory is **encrypted you cannot point** $JUJU_HOME or root-dir to a location within it. Use locations outside of it.
- Juju cannot be run **under sudo** because it needs to manage permission as the real user.
- **lxc-clone** option in environments.yaml. The default is "true" for Trusty and above

## JujuGUIセットアップ

### まずはインストール

```bash
sudo add-apt-repository ppa:juju/stable
sudo apt-get update && sudo apt-get install juju-core
juju generate-config
```

### メインのディレクトリを変えたい場合

```bash
vi .juju/environments.yaml
```

```yaml
local:
 type: local
 root-dir: /xxx/.juju/local
```

```bash
sudo chmod -R 777 /xx/
```

### そして起動

```bash
sudo apt-get install juju-local
juju switch local
juju bootstrap            # 一番親になるプロセスです
juju deploy juju-gui      # jujuのWEBからアクセスできるサービスをデプロイ
juju status juju-gui      # いま何をやってるところか確認
                          # agent-state: started になるまで約20分
juju expose juju-gui      # exposeすることで、WEBにアクセスが可能になります
```

### 他の参考コマンド（失敗した時用）

```bash
juju remove-service
juju remove-unit
juju remove-machine
```

## アクセス設定

すると下記のように、Juju-GuiへのアクセスのIPが出てきます

```bash
wabuntu@tt:~/.juju$ juju status
...
 open-ports:
 - 80/tcp
 - 443/tcp
 public-address: 10.0.3.49
```

10.0.3.*のIPは、自動で作られた下記のNICです

```bash
wabuntu@tt:~/.juju$ ip a
...
3: lxcbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
 inet 10.0.3.1/24 brd 10.0.3.255 scope global lxcbr0
```

作業用マシンは192.168.1.*しか持っていないので、ルーティングを実施

```bash
wabuntu@nuc:~$ sudo route add -net 10.0.3.0 netmask 255.255.255.0 gw 192.168.1.100 dev em eth0
wabuntu@nuc:~$ ping 10.0.3.49
```

すると下記のWEBページにアクセスできるようになります

https://10.0.3.49/

### ログイン情報

ログインのユーザーは下記のように調べることができる

```bash
wabuntu@tt:~/.juju$ juju api-info --password
user: admin
password: 28a3b6a2918xxxxxxxxxxxxxxxx
```

### イメージキャッシュ

イメージのキャッシュもいつの間にかできています

```bash
$ juju cached-images list
Cached images:
- kind: lxc
  series: trusty
  arch: amd64
  source-url: https://cloud-images.ubuntu.com/server/releases/trusty/release-20160222/ubuntu-14.04-server-cloudimg-amd64-root.tar.gz
  created: Mon, 14 Mar 2016 10:32:57 JST
```

見た目は大変素敵ですが、なくても作業はできます ＞ juju-gui


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/14/%e4%b8%80%e5%8f%b0%e6%a7%8b%e6%88%90%e3%81%a7juju%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%a6%e3%81%bf%e3%82%8b/).*
