---
layout: post
title: "NICのパケロス等を調べる"
date: 2016-03-29 00:00:00 +0900
lang: ja
---

こんなコマンドが使えるようです。

```shell
wabuntu@nuc:/var/log/maas$ ethtool -S eth0
NIC statistics:
     rx_packets: 4142731
     tx_packets: 3216842
     rx_bytes: 1885950282
     tx_bytes: 307023631
     rx_broadcast: 21589
     tx_broadcast: 3702
     rx_multicast: 6626
     tx_multicast: 1503
     rx_errors: 0
     tx_errors: 0
     tx_dropped: 0
     multicast: 6626
   ....
```

```shell
wabuntu@nuc:/var/log/maas$ ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:        4096
RX Mini:    0
RX Jumbo:    0
TX:        4096
Current hardware settings:
RX:        256
RX Mini:    0
RX Jumbo:    0
TX:        256
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/29/nic%e3%81%ae%e3%83%91%e3%82%b1%e3%83%ad%e3%82%b9%e7%ad%89%e3%82%92%e8%aa%bf%e3%81%b9%e3%82%8b/).*
