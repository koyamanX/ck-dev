---
title: "Booting linux kernel on my RISC-V part2"
date: 2021-03-22T10:16:42+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V", "CPU"]
draft: false
---
[前回]({{<ref "/blog/booting_linux_kernel_on_my_riscv_part1/index.md" >}})で起きた、`riscv-int`の初期化中?のハングの原因を見ていく。
<!--more-->

## 問題点
前回のLinuxの実行ログ。

{{<figure src="./image00.png" >}}
{{<figure src="./image01.png" >}}

`riscv-int`の出力のあと、シミュレータがハングしているようである。

## 原因に目星をつける。

`riscv-int`が32 local interruptsを持っていると認識したということは、FDTを読んで、デバイスドライバの選択をし、初期化を始めたはず。
考えられる原因として、

- 割込みコントローラーのロード・ストアアクセスフォルト(アドレスのレンジを超えた？)
- デバイスツリーの記述ミス

あたりかなと思う。

一先ず、Linux kernelが`riscv-int`に対して何をしてるのか調べてみることから始める。
