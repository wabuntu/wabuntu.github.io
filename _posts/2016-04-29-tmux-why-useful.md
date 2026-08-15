---
layout: post
title: "tmuxって何が便利なの？"
date: 2016-04-29 00:00:00 +0900
lang: ja
---

正直私は「gnome-terminalじゃダメなの？」と思っていたクチです。「ウインドウ複数立ち上げられるし、タブもできるじゃん」と。しかし実際使ってみると・・・

* ssh中にネットワークが**切断されても、強制終了しなくていい**（デタッチという操作で終了できる）
* おまけにネットワーク復旧後に**再度接続（アタッチ）すると、ログイン後の状態を保持**してるし、ログも残ってる
* セッション（一連の画面分割とかsshの接続先）とかが、**アプリ終了後も保存**されてる
* **別のPCから**tmuxを使ってたPCにsshすれば、そのセッションがそのまま使える
* 他の人にセッションに同時に入ってもらって、**作業を見せたりできる**

## 起動します。

単純にtmuxと打ってもいいです。

```shell
$ tmux new-session -s test0        #test0というセッション名を指定して起動。
```

画面下に緑色のバーが出て、`[test0]`とセッション名が出ると思います。

* Ctrl+bのあとに"を押すと（つまりShift+2）、上下にペインが分割します。
* Ctrl+bのあとに%を押すと（つまりShift+5）、左右にペインが分割します。
* Ctrl+bのあとに、上下左右キーでペインを移動できます。
* ウインドウはCtrl+bのあとにcを入れると作れて、Ctrl+bのあとにpやnを入れると移動できます。

Ctrl+bはprefix keyというそうで、これは設定で変えられます。

* 終了はCtrl+bのあとdを押すか、gnome-terminalをそのまま閉じればいいです。

## セッションのリストはこのように出せます。

```shell
$ tmux ls
0: 1 windows (created Thu Apr 28 22:16:49 2016) [100x49]
test: 1 windows (created Fri Apr 29 11:19:48 2016) [100x49]
test0: 1 windows (created Fri Apr 29 11:27:41 2016) [100x49]
test1: 1 windows (created Fri Apr 29 11:28:39 2016) [100x49]
```

下記の３つのどれかで、閉じた（デタッチした）セッションに再接続（アタッチ）できます。

```shell
$ tmux attach -t test0
$ tmux a -t test0        #aは省略形
$ tmux a                 #指定しないと最後に使っていたセッションになる
```

起動すると、分割されたペインも、sshもログインした状態で残ってますよね？（すげぇ）

他の人とセッションを共有したい場合は、セッション名を教えてあげて上記のように起動してもらえばOKです。

## その他のコマンド

```shell
$ tmux kill-session -t test0
$ tmux rename-session -t OLDNAME NEWNAME
```

今でもvimでなくviを使っている私ですが、tmuxは今後使っていきたい感じです。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/04/29/tmux%e3%81%a3%e3%81%a6%e4%bd%95%e3%81%8c%e4%be%bf%e5%88%a9%e3%81%aa%e3%81%ae%ef%bc%9f/).*
