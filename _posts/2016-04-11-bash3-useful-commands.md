---
layout: post
title: "bash3の便利コマンド"
date: 2016-04-11 00:00:00 +0900
lang: ja
---

Bashも新しくなってずいぶん自分に時代遅れ感が出ております・・・

```bash
stringZ=abcABC123ABCabc

echo ${#stringZ}         #15　長さを返す
echo ${stringZ:7}        # 23ABCabc　抽出
echo ${stringZ:7:3}      # 23A　長さ指定で抽出

echo ${stringZ#a\*C}      # 123ABCabc　前方から最短マッチを消す
echo ${stringZ##a\*C}     # abc　前方から最長マッチを消す

echo ${stringZ%A\*C}      # abcABC123　後方から最短マッチを消す
echo ${stringZ%%A\*C}     # abc　後方から最長マッチを消す

echo ${stringZ/abc/xyz}  # xyzABC123ABCabc　置き換え（一個）
echo ${stringZ//abc/xyz} # xyzABC123ABCxyz　置き換え（全部）
echo ${stringZ/#abc/XYZ} # XYZABC123ABCabc　前方から
echo ${stringZ/%abc/XYZ} # abcABC123ABCXYZ　後方から
```

## test、いわゆるifについて

* `[`はtestと同義である
* `[[ ]]`はBash2.02から追加されたもので、コーディングミスが少ないそうな
* `[[ $a -lt $b ]]`はexit statusを返す

## ちなみにMatchはこんなに便利になったみたい

```bash
if [[ "$stringZ" =~ A.\*A ]] ; then
  echo ${BASH_REMATCH[0]}  # ABC123A
fi
```

## 参考サイト

文字列操作：

* http://www.tldp.org/LDP/abs/html/string-manipulation.html#STRINGMANIP
* http://www.tldp.org/LDP/abs/html/parameter-substitution.html#PARAMSUBREF

配列：

* http://www.tldp.org/LDP/abs/html/arrays.html

Match：

* http://dqn.sakusakutto.jp/2013/06/bash_rematch_regexp.html


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2016/04/11/bash3%e3%81%ae%e4%be%bf%e5%88%a9%e3%82%b3%e3%83%9e%e3%83%b3%e3%83%89/).*
