---
layout: post
title: "ssmtpでGmailを使う"
date: 2016-04-10 00:00:00 +0900
lang: ja
---

参考：[http://askubuntu.com/questions/12917/how-to-send-mail-from-the-command-line](http://askubuntu.com/questions/12917/how-to-send-mail-from-the-command-line)

```bash
sudo apt-get install ssmtp
```

```bash
cat /etc/ssmtp/ssmtp.conf
```

```
root=aaa.bbb@gmail.com
mailhub=smtp.gmail.com:587
rewriteDomain=gmail.com
hostname=gmail.com
FromLineOverride=NO
UseTLS=Yes
UseSTARTTLS=Yes
AuthUser=aaa.bbb
AuthPass=password
AuthMethod=LOGIN
FromLineOverride=yes
```

```bash
echo "test" | ssmtp bbb.ccc@gmail.com
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/04/10/ssmtp%e3%81%a7gmail%e3%82%92%e4%bd%bf%e3%81%86/).*
