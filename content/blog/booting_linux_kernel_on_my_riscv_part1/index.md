---
title: "Booting linux kernel on my RISC-V part1"
date: 2021-03-16T04:01:13+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V", "CPU"]
draft: false
---

## 概要
[前回]({{<ref "/blog/build_linux_for_my_riscv_processor.md" >}})ビルドしたLinuxを用いてシミュレーションを行った。
しかし、例外が発生し、処理が進まない。
今回は、原因を追求してみようと思う。
<!--more-->

## 現状
まずは、現状確認がてら実行した様子を示す。

{{<figure src="./image01.png" >}}

`status: 00000100 badaddr: dead4ead cause: 0000000c epc: c044da24`
がCSRの情報である。
`cause: 0xc`なので、Instruction page faultである様子。
どんな命令があるのか見てみる。\

```bash
riscv32-unknown-linux-gnu-objdump linux/vmlinux |less
```
でディスアセンブルする。
`c044da24:`で検索すると以下の命令であることがわかる。
{{<figure src="./image02.png" >}}
`__printk_safe_flush`関数の命令の一部である。
シミュレータで`0ef725af amoswap.w.aqrl a1, a5, (a4)`が実行される際のレジスタをダンプする。
{{<figure src="./image03.png" >}}
`a4: c140c9c4 a5: 00000000`であることがわかる。
よって、`a1 <- *(0xc140c9c4), *(0xc140c9c4) <- 0x00000000`となる。
また、シーケンシャルコンシステンシーで、ワードサイズのオペレーションである。
というか、Instruction page faultのはずが、Store amo address misalignedとなっている。
おかしい。
- おそらく、Store amo address misalignedが正しい
- Store amo address misalignedのときのscauceがおかしい？
	- mcauseは？
	- OpenSBIがM-ModeのトラップをS-Modeへredirectする。
- Address misalignedの検出回路がおかしい？
	- inst: 0x0ef725af
	- a1: 0xc140c9c4
	- byteen: MEM\_WORD

### Store amo address misalignedのときのscauceがおかしい？
まずは、mcauseの値を見てみる。
シミュレータ上では、Store amo address misalignedとなっているので、mcauseは正しいはず。
`sbi_trap_handler`内(80003b5c)で、csr s1, mcause命令でmcauseをs1レジスタに読み出している。
{{<figure src="./image04.png" >}}
`sbi_trap_handler`はOpenSBIの一部であり、mtvecアドレスがにセットされている。
よって、トラップ時に実行される。S-Modeで対処可能なトラップはredirectされる。
mcauseは0x6となっているのがわかる。
0x6はStore AMO address misalignedを示す。
`sbi_trap_redirect`関数の一部で、scauseに0x6をセットしているので、Linuxカーネルには正しく0x6が渡されている。
よって、カーネル内どこかでscauseに0xcがセットされているよう?
ひとまず、これは保留。
{{<figure src="./image05.png" >}}

### Address misalignedの検出回路がおかしい？
検出はアドレス計算をするexecuteステージで行っている。
以下にNSLのコード片を示すが、AMO命令に関してはワード境界に整列されたアドレスに対するワード単位の演算しかサポートしていない。
```
...
#define MEM_WORD 0x2
#define MEM_HALFWORD 0x1
#define MEM_BYTE 0x0
...
			any {
				/* Data address misaligned */
				((alu_q[1:0] != 2'b00) && ({1'b0, DEREG.funct3[1:0]} == MEM_WORD)):			m_misaligned();
				((alu_q[0] != 1'b0) && ({1'b0, DEREG.funct3[1:0]} == MEM_HALFWORD)):		m_misaligned();
				((DEREG.rs1_data[1:0] != 2'b00) && ({1'b0, DEREG.funct3[1:0]} == MEM_WORD)):a_misaligned();
			}
			/* Exceptions in execute stage */
			any {
				(DEREG.illegal_instruction):	illegal_instruction_execute_stage();
				(i_misaligned):					instruction_address_misaligned(target_address);
				(DEREG.load && m_misaligned):	load_address_misaligned(alu_q);
				(DEREG.store && m_misaligned):	store_address_misaligned(alu_q);
				(DEREG.amo && a_misaligned):	store_address_misaligned(DEREG.rs1_data);
			}
```

`0x0ef725af amoswap.w.aqrl a1, a5, (a4)`より、funct3は0x2である。
{{<figure src="./image06.png" >}}
`rs1_data: 0xc140c9c4`なので、ワード境界に整列している。
また、funct3==0x2なので、MEM\_WORDと同値である。
もしかして、rs1\_dataが正しくフォワーディングされていないのでは？
### rs1\_dataが正しくフォワーディングされていないのでは？
フォワーディングが正しくない。
DEREG(Decode/Executeのパイプラインレジスタ)からrs1のデータを読み出しているが、正しくはパイプラインレジスタからではなく、フォワーディングする必要がある。
細かいが、execute\_alu\_aがexecuteステージに向けたrs1用のフォワーディングパスである。
rs1\_dataとして、execute\_alu\_aを用いる。
フォワーディングするかしないかは、命令ごとにalu\_a\_forward\_en端子にて制御する。
AMO命令では、rs1のデータをフォワーディングをする。
以下のように修正して、再度シミュレータをコンパイルした。

```
			any {
				/* Data address misaligned */
				((alu_q[1:0] != 2'b00) && ({1'b0, DEREG.funct3[1:0]} == MEM_WORD)):			m_misaligned();
				((alu_q[0] != 1'b0) && ({1'b0, DEREG.funct3[1:0]} == MEM_HALFWORD)):		m_misaligned();
				((execute_alu_a[1:0] != 2'b00) && ({1'b0, DEREG.funct3[1:0]} == MEM_WORD)):a_misaligned();
			}
			/* Exceptions in execute stage */
			any {
				(DEREG.illegal_instruction):	illegal_instruction_execute_stage();
				(i_misaligned):					instruction_address_misaligned(target_address);
				(DEREG.load && m_misaligned):	load_address_misaligned(alu_q);
				(DEREG.store && m_misaligned):	store_address_misaligned(alu_q);
				(DEREG.amo && a_misaligned):	store_address_misaligned(execute_alu_a);
			}
```
再度、riscv-testsを実施した。正しくパスしている。
{{<figure src="./image07.png" >}}
というか、前までも(すり抜けて)パスしていた。
問題となっているパターンは
```
addi a4, s6, -4
amoswap.w.aqrl a1, a5, (a4)
```
のような先発のrdが後発のrs1となっているパターンである。
そこそこ、限定的な上に、AMO命令なので、気づくのにかなり時間を要した。

### Linuxの再実行
再度、Linuxをシミュレータ上で実行してみる。
{{<figure src="./image08.png" >}}
進展した。しかし、ハングした。
さらなる問題が発生したので、次回はこの問題の原因の追求をする。
仮想メモリのマップ(Virtual kernel memory layout)はどうやって決めているのだろう。
RV32XSoCにはPCIとかインストールしていないよ?
