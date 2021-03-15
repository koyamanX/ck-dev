---
title: "Next Synthesis Language"
date: 2021-03-15T19:00:21+09:00
author: "@koyamanX"
categories: ["CPU"]
tags: ["RISC-V", "CPU", "NSL", "FPGA"]
draft: false
---

## 高位合成言語NSL

NSLで自作のRISC-Vプロセッサを開発している。
- [NSLリファレンス](http://www.overtone.co.jp/release_data/documents/reference/NSL_Language_Reference_ver1.5.pdf)
- [NSLチュートリアル](http://nshimizu.com/parthenon/NSLTUTORIAL.pdf)

開発には高位合成言語NSLを用いている。
NSLでは、Verilog HDL, VHDL, System-Cに変換でき、ステートマシンやパイプラインなどCPU開発に向いた専用の構文が用意されている。
単相同期を前提に設計されているため、クロックを明示する必要がない(m\_clockとp\_resetが自動で生成される)。
宗教上の理由よりNSLを使い始めたわけであるが、シンプルで可読性も高く、かなり使いやすいので、Verilog HDLなどで開発する気が起きない。
ライセンスは非商用の教育用途のものを用いており、2000行の制限がある。
商用には使わないし、
分割コンパイルをすれば、2000行を超えることはほぼないので、十分である。
<!--more-->
## 開発環境
開発環境の構築は開発者の先生の公開されている動画を参照するとわかりやすい。[YouTubeリンク](https://youtu.be/Tz8np5EqjVQ?t=844)
## おわり
自分の開発しているCPUや周辺回路などの実装については、メモがてらまとめるかもしれない。


