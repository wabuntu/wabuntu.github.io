---
layout: post
title: "パッチがどのバージョンのカーネルに当たってるか"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

The blog post describes how to determine which kernel version contains a specific patch.

## Steps

Clone the Linux kernel repository:

```shell
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

Check which tags contain the commit:

```shell
git tag –contains baf606d9c9b12517e47e0d1370e8aa9f7323f210
```

The example shows this returns `v4.2`, indicating the patch is included in kernel version 4.2.

The post references a specific commit to the IPv4 socket code via the kernel.org git interface as a practical example of this technique.


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/%e3%83%91%e3%83%83%e3%83%81%e3%81%8c%e3%81%a9%e3%81%ae%e3%83%90%e3%83%bc%e3%82%b8%e3%83%a7%e3%83%b3%e3%81%ae%e3%82%ab%e3%83%bc%e3%83%8d%e3%83%ab%e3%81%ab%e5%bd%93%e3%81%9f%e3%81%a3%e3%81%a6%e3%82%8b/).*
