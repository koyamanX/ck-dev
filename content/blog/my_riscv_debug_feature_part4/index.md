---
title: "My RISC-V debug feature part4"
date: 2021-05-09T23:05:20+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "JTAG", "FPGA"]
draft: false
---

これから、DM(Debug Module)の実装をおこない、VJTAGを通して、OpenOCDと情報をやりとりできるかを確認していく。
まずは、かんたんな実装から始める。
参照しているspecのバージョンは`v0.13`である。
<!--more-->


## 機能
- 対象のHartsの選択
- Halt/Resume
- Abstracts commands
- Program buffer
- ステップ実行
- トリガー
- リセット
などが挙げられる。
なお、Debuggerが介入してきた場合は、hartはD-Mode(Debug Mode)で動作を行う。
D-Modeでは、基本的にはM-Modeと同じだが、割り込みが無効になり、例外はデバッガが対処を行う。また、デバッカーから命令が供給されるまで、haltする。

## 全体像
{{<figure src="./image00.png" >}}
実線で囲まれている部分は実装が必須であり、点線で囲まれている部分は実装するかは任意である。
今回は、実線の部分のみ実装する。
なお、DTMに関してはほとんど前回までで実装が完了している。
また、VJTAGおよびDTM、DMはTCK(VJTAGのクロックドメイン)で動作している。
そこで、hart <-> DM間にクロッククロスドメインの機構を用意する。
念の為記しておくが、System busへのアクセスは実装しないが、System busは`m_clock`ドメインで動作している。
{{<figure src="./image01.png" >}}


### CSRs
- dcsr
- dpc
- dscratch(0..N)

うち、`dpc`, `dscratch`はXLEN bitであり、`dcsr`は最長`2**3`ビットである。

## 今回実装する機能
### Ver1
- hartの選択
- (halt中の)GPRsに対するアクセス
- hartのリセット(resetvectorからの再実行)
- いくつかのAbstarct Commands
- halt/resume
### ver2
- ステップ実行
- Triggers
- Program Buffer

とりあえず、ver1を実装する。
Spikeや他のRTLによる実装を参考に実装していく。
次回からは、hartの選択、halt、GPRsへのアクセスを実装する。
内容が薄いが今回はここまで。

## 付録 Spikeのビルド
```bash
git clone --recursive https://github.com/riscv/riscv-isa-sim
cd riscv-isa-sim
mkdir build
cd build
../configure --prefix=$RISCV --enable-commitlog --enable-histogram
make -j $(nproc)
make install
```
これで、デバックモードの機能を扱うことができる。

## 参考
- ドキュメント・スペック
	- [SiFive E31/E51 Coreplex](https://sifive.cdn.prismic.io/sifive/fab000f6-0e07-48d0-9602-e437d5367806_sifive_U54MC_rtl_full_20G1.03.00_manual.pdf)
	- [RISC-V Debug Spec Update](https://riscv.org/wp-content/uploads/2017/05/Wed1445_Debug_WorkingGroup_Wachs.pdf)
	- [https://www.lauterbach.com/pdfnew/debugger\_riscv.pdf](https://www.lauterbach.com/pdfnew/debugger_riscv.pdf)
- 実装
	- [Spike](https://github.com/riscv/riscv-isa-sim)
	- [https://github.com/aquaxis/riscv\_debug](https://github.com/aquaxis/riscv_debug)
