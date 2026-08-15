---
layout: post
title: "postfixのnull clientを試す"
date: 2016-04-10 00:00:00 +0900
lang: ja
---

参考：http://www.postfix.org/STANDARD_CONFIGURATION_README.html#null_client

非常にシンプル。だがUbuntuではまずsendmailを削除することから始めないといけない

```bash
$sudo service stop sendmail

$sudo apt-get remove sendmail

$ sudo lsof -i:25
COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
sendmail- 2320 root 4u IPv4 28717 0t0 TCP localhost:smtp (LISTEN)

$sudo kill -9 2320
```

おわかりいただけただろうか。service stopでは死なないのである・・・。

次にpostfixのインストールと設定

```bash
$sudo apt-get install postfix
```

インターネットサイト、ホスト名はデフォを選択

```bash
$sudo vi /etc/postfix/main.cf
```

```
myhostname = corsair.domain
myorigin = $mydomain
relayhost = $mydomain
inet_interfaces = loopback-only
mydestination =
```

```bash
$echo "test" | mail aaa.bbb@gmail.com
```

しかし、今日びのメールサービスはそんな怪しいところからのメールは受け付けないのであった・・・。

次回、ssmtpとGmail編に続く！


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/04/10/postfix%e3%81%aenull-client%e3%82%92%e8%a9%a6%e3%81%99/).*
