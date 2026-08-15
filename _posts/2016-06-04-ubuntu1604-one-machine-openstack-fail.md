---
layout: post
title: "Ubuntu16.04を使ってマシン一台でOpenStack（失敗）"
date: 2016-06-04 00:00:00 +0900
lang: ja
---

失敗に続く失敗にもめげず再挑戦。公式では、一台でやるには８CPU、１２Gのメモリ、１００G以上のSSDが必要となっています。私の使ったPCは下記です。

* CPU：　i5-4460、４コア４スレッド
* メモリ：　１６GB
* SSD:　１２８GB
* OS：　ubuntu-server 16.04
  * 基本+SSHサーバ
  * lvmなし
  * home暗号化なし
  * 初期ユーザーはubuntu(おそらくこれ大事)
* NIC:　オンボード＋USB-LAN（オンボードの方に固定IP(192.168.1.100)を、USB-LANは何も設定せず）

上記プラス、Webインターフェースを使うためにラップトップを使用しています（じゃあ実質２台やんけ）。下記のサイトを参考にインストールを進めます。

* https://help.ubuntu.com/lts/clouddocs/installer/en/single-install.html
* https://wiki.ubuntu.com/OpenStack/Installer/debugging/single-install

## インストール

```shell
sudo apt-add-repository ppa:cloud-installer/stable
sudo apt-get update
sudo apt-get install openstack
sudo openstack-install
```

ここでコマンドが無いと言われるので下記をインストール、実行

```shell
sudo apt-get install conjure-up
conjure-up openstack
```

下記のようなエラーがでるため、conjure-upの新しいのを入れる

```
UnboundLocalError: local variable 'creds' referenced before assignment
```

```shell
sudo apt-add-repository ppa:conjure/ppa
sudo apt-get update
sudo apt-get upgrade
```

もう一度実行

```shell
conjure-up openstack
```

![Screenshot from 2016-06-03 13:41:50](https://wabuntu.wordpress.com/wp-content/uploads/2016/06/screenshot-from-2016-06-03-134150.png?w=440&h=420)

ただのOpenStackを選択。次の画面でlocalhostをスペースで選択。するとBootstrapping…という画面が表示され、２，３分でBundleEditorというのが出る。追加はできるが変更はできないので（マシンが少ないのを選びたいのに！）、とりあえずCommitを押し、Deploy。失敗した場合は、Ctrl+Cで一旦終了して、~/.local/shareを消すと元に戻せるようだ（一旦Deployしちゃうとアウト）。

![Screenshot from 2016-06-03 11:42:30](https://wabuntu.wordpress.com/wp-content/uploads/2016/06/screenshot-from-2016-06-03-114230.png?w=445&h=394)

１，２分すると、おなじみの画面が表示される。ストレージ(radosgw)はデフォがcephになったようですね。

![Screenshot from 2016-06-03 11:45:29](https://wabuntu.wordpress.com/wp-content/uploads/2016/06/screenshot-from-2016-06-03-114529.png?w=443&h=392)

５台目ぐらいでだんまりに・・・。正直他のソフトのラッパーなのにブラックボックスなので、失敗すると原因究明や修正が非常に難しい・・・。誰かボスケテ。

## 後日談

実はNUCでやると、上記画面がかなりのところまでスムーズに進みました。おそらくconjure-upやopenstack-installerでは、HDDの速度がかなりのキモで、そのためSSDで試した時よりM2搭載のNUCのほうが良かったのではないかと。ちなみにCPUやメモリはかなり余裕がありました。

ただしNUCでやった場合でも、ceph-osdの３台が"No block devices detected using current configuration"で止まります（他はreadyまで行く）。既知のバグでext4の場合のファイル名の長さに引っかかるから、というのがありましたが、そのfixはすでに適用されていました。

あとは、どうもceph-osdがデフォルトでsdbを使おうとする点が怪しいのですが、bundleは上記の方法ではいじれないし、ディスクも追加してみたりしたけどダメでした。

どなたか成功した方いたら教えてください。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/06/04/ubuntu16-04を使ってマシン一台でopenstack/).*
