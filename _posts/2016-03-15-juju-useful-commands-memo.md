---
layout: post
title: "juju便利コマンドメモ"
date: 2016-03-15 00:00:00 +0900
lang: ja
---

ユニット指定でSSH  
```shell
juju ssh servicename/machine#
```

マシン指定でSSH  
```shell
juju ssh machine#
```

全部のマシンで一気にコマンド実行  
```shell
juju run "uname -a" –all
```

簡単SCP  
```shell
juju scp file.sh servicename/0:/tmp
```

デバッグモード（SSHでTmuxに入れる）  
```shell
juju debug-hooks
```

全部やり直し  
http://askubuntu.com/questions/403618/how-do-i-clean-up-a-machine-after-using-the-local-provider

エラー復旧トライ  
```shell
juju resolved $servicename –retry
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/15/juju%e4%be%bf%e5%88%a9%e3%82%b3%e3%83%9e%e3%83%b3%e3%83%89%e3%83%a1%e3%83%a2/).*
