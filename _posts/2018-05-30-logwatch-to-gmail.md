---
layout: post
title: "logwatchの結果をGmailに送りたい"
date: 2018-05-30 00:00:00 +0900
lang: ja
---

普段の作業用のマシン、rootに届いたメールをあまりチェックしないので、Gmailに送りたいのです。簡単にできないものかと思ったらこんなページがありました。

https://qiita.com/chikoboo/items/77d0030c49e0d8b7fd4f

まず、ssmpをインストールして設定します。

```bash
$ sudo apt-get install ssmtp
$ sudo vi /etc/ssmtp/ssmtp.conf
```

```
root=[YOU]@gmail.com
mailhub=smtp.gmail.com:587
rewriteDomain=gmail.com
hostname=gmail.com
AuthUser=[YOU]@gmail.com
AuthPass=[PASS]
AuthMethod=LOGIN
UseSTARTTLS=YES
FromLineOverride=YES
```

テストメール送信で動作確認します。

```bash
$ echo "hello" | sendmail [YOU]@gmail.com
```

これで届くことが確認できました。

次にLogwatchの設定を確認します。

```bash
$ sudo vi /etc/logwatch/conf/logwatch.conf
```

```
mailer = "/usr/sbin/sendmail -t"
MailTo = [YOU]@gmail.com
```

LogwatchはCronの設定でこのように毎日動くようになっています。

```bash
$ ls /etc/cron.daily/
00logwatch

$ cat /etc/cron.daily/00logwatch 
/usr/sbin/logwatch --output mail
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2018/05/30/logwatch%e3%81%ae%e7%b5%90%e6%9e%9c%e3%82%92gmail%e3%81%ab%e9%80%81%e3%82%8a%e3%81%9f%e3%81%84/).*
