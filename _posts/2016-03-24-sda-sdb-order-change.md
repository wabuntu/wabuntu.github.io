---
layout: post
title: "sdaとかsdbの順序って変えられるの？"
date: 2016-03-24 00:00:00 +0900
lang: ja
---

本当はメインのHDDをsdaにしたいのに何故か勝手に変わる。HDDを一本だけにしてインストールしたあとでHDDを追加してもダメ。なんとかならんのか。

いろいろ調べてみたけど世間的には「もうそんなのやめてblkidと/dev/disk/by-uuidを使ったら？」という流れのようだが、未だに/dev/sd\*を使わないといけない機会はある気がする。

そこで見つけたのが下記のサイト。「udevでできんじゃね？」という緻密な実験をして下さっている。

http://d.hatena.ne.jp/incarose86/20150125/1422211309

残念ながらudevではできなそうで、結局カーネルが認識した順番で自動で決まってしまうようだ。そしてシンボリックリンクを使うという方法に落ち着いた模様。


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/03/24/sda%e3%81%a8%e3%81%8bsdb%e3%81%ae%e9%a0%86%e5%ba%8f%e3%81%a3%e3%81%a6%e5%a4%89%e3%81%88%e3%82%89%e3%82%8c%e3%82%8b%e3%81%ae%ef%bc%9f/).*
