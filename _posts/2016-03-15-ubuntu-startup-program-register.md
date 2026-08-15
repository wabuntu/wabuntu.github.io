---
layout: post
title: "Ubuntuでスタートアップのプログラム登録"
date: 2016-03-15 00:00:00 +0900
lang: ja
---

## 自動起動プログラムを登録する方法

GoldenDictのショートカット（Windowsで言うところの）はここに保存されています。Windowsキーを押して検索できる、アイコンが出てくるものはどうやらこのファイル（.desktop）がここに登録されているから、のようだ。

```bash
$ cat /usr/share/applications/goldendict.desktop 
[Desktop Entry]
Type=Application
Terminal=false
Categories=Office;Dictionary;Education;Qt
Name=GoldenDict
GenericName=Multiformat Dictionary
Comment=GoldenDict
Encoding=UTF-8
Icon=/usr/share/pixmaps/goldendict.png
Exec=goldendict
```

ここのExec=goldendictというのがいわゆるコマンドな様子。正確にはこれ？

```bash
$ which goldendict 
/usr/bin/goldendict
```

Widnowsキーを押すか、X端末から下記を呼び出して、GoldenDictを登録すればよろし

```bash
$ gnome-session-properties
```

おそらくここに.desktopファイルをコピーすればいいような気がするが、定かではない

```bash
$ sudo locate autostart
/etc/xdg/autostart
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/15/ubuntu%e3%81%a7%e3%82%b9%e3%82%bf%e3%83%bc%e3%83%88%e3%82%a2%e3%83%83%e3%83%97%e3%81%ae%e3%83%97%e3%83%ad%e3%82%b0%e3%83%a9%e3%83%a0%e7%99%bb%e9%8c%b2/).*
