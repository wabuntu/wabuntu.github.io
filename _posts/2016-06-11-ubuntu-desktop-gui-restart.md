---
layout: post
title: "UbuntuデスクトップのGUI環境を再起動"
date: 2016-06-11 00:00:00 +0900
lang: ja
---

Ctrl+Alt+F（2など）でCUIに移動後、下記実行

```shell
sudo service lightdm stop
sudo service lightdm start
```

下記でも可

```shell
sudo restart lightdm
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/06/11/ubuntuデスクトップのgui環境を再起動/).*
