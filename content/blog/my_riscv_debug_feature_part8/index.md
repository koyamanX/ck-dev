---
title: "My RISC-V debug feature part8(implement suggested DMI signal interface)"
date: 2021-07-20T22:28:05+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "JTAG", "FPGA"]
draft: false
---
独自のDMIインターフェースから推奨インターフェースに変更する。
<!--more-->

riscv-debugによると、以下の信号がインターフェースとして推奨されているようである。

{{<figure src="./suggested_DMI_interface_from_spec.png">}}

ちなみに、今回の実装ではABITSは32 bitとしている。
