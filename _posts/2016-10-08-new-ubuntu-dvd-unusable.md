---
layout: post
title: "何か新しいUbuntuでDVD使えないよね"
date: 2016-10-08 00:00:00 +0900
lang: ja
---

要はUbuntu１５．０４からLibdvdcss2が使えないみたい。

```bash
sudo apt-get install libdvd-pkg
sudo dpkg-reconfigure libdvd-pkg
```

あとはBraseroで

## 参考：

https://help.ubuntu.com/community/RestrictedFormats/PlayingDVDs

なぜVLCに限ってのみ再生できるかというと、VLCにはLibdvdcss2が内蔵されているからということらしい


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/10/08/何か新しいubuntuでdvd使えないよね/).*
