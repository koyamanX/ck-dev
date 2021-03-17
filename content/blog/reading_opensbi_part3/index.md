---
title: "Reading OpenSBI part3"
date: 2021-03-17T07:35:38+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---

[前回]({{<ref "/blog/reading_opensbi_part2/index.md" >}})は、`_relocate_done`まで読んだ。
続きから読み進めていく。
<!--more-->

# fw\_base.S

`firmware/fw_base.S`
```asm
...
	li	ra, 0
	call	_reset_regs

	/* Zero-out BSS */
	la	s4, _bss_start
	la	s5, _bss_end
_bss_zero:
	REG_S	zero, (s4)
	add	s4, s4, __SIZEOF_POINTER__
	blt	s4, s5, _bss_zero

	/* Setup temporary trap handler */
	la	s4, _start_hang
	csrw	CSR_MTVEC, s4

	/* Setup temporary stack */
	la	s4, _fw_end
	li	s5, (SBI_SCRATCH_SIZE * 2)
	add	sp, s4, s5

	/* Allow main firmware to save info */
	MOV_5R	s0, a0, s1, a1, s2, a2, s3, a3, s4, a4
	call	fw_save_info
	MOV_5R	a0, s0, a1, s1, a2, s2, a3, s3, a4, s4

#ifdef FW_FDT_PATH
	/* Override previous arg1 */
	la	a1, fw_fdt_bin
#endif
...
```

```asm
...
_reset_regs:
	fence.i
	li sp, 0
	li gp, 0
	li tp, 0
	li t0, 0
	li t1, 0
	li t2, 0
	li s0, 0
	li s1, 0
	li a3, 0
	li a4, 0
	li a5, 0
	li a6, 0
	li a7, 0
	li s2, 0
	li s3, 0
	li s4, 0
	li s5, 0
	li s6, 0
	li s7, 0
	li s8, 0
	li s9, 0
	li s10, 0
	li s11, 0
	li t3, 0
	li t4, 0
	li t5, 0
	li t6, 0
	csrw CSR_MSCRATCH, 0
	ret
...
```
まず、`_reset_regs`を実行する。
`_reset_regs`では、まず、命令ストリームをメモリと同期する。
これは、リロケーション先にジャンプしているので、必要である。リロケーションはメモリ書き込み命令にて行うため、命令キャッシュと同期していない場合がある。
その後、`ra, a0, a1, a2`レジスタ以外を`0`に初期化する。
加えて、`mscratch`レジスタも初期化する。
次に、bssセクションを`0`で初期化する。
`_bss_start`はページ境界配置されている。
`_bss_end`は8バイト境界配置されている。
`__SIZEOF_POINTER__`サイズ(4 byte for RV32, 8 byte for RV64)でストアを行う。
次に、一時的なトラップハンドラーを設定する。　
`_start_hang`は8バイト境界配置されている。このアドレスをそのまま`mtvec`レジスタへ代入しているため、下位2ビットは常に`00`である。
よって、`Direct mode`として割込み時トラップを行う。
例外のトラップについては常に`Direct mode`である。

`firmware/fw_base.S`
```asm
...
	.section .entry, "ax", %progbits
	.align 3
	.globl _start_hang
_start_hang:
	wfi
	j	_start_hang
...
```

次に、作業用の一時的なスタックを用意する。
`_fw_end`より、`SBI_SCRATCH_SIZE*2`(8KB)分確保する。
計算したアドレスを`sp`レジスタへ入れる。
Stack grows downwardなので、スタックの終わりアドレスである。

`include/sbi/sbi_scratch.h`
```c
#define SBI_SCRATCH_SIZE	(0x1000)
```
{{<figure src="./image00.png" >}}

次に、レジスタの退避を行う。
`(s0, s1, s2, s3, s4) <- (a0, a1, a2, a3, a4)`\
`fw_save_info`関数を呼び出す。(ファームウェアの種類によって実装が違う。今回は`FW_PAYLOAD`)

`firmware/fw_payload.S`
```asm
...
fw_save_info:
	ret
...
```
レジスタの復帰を行う。
`(a0, a1, a2, a3, a4) <- (s0, s1, s2, s3, s4)`\

`FW_FDT_PATH`によってFDT(Flattend Device Tree)が指定されている場合は、`a1`レジスタをそのポインタ(`fw_fdt_bin`)で置き換える。

`firmware/fw_base.S`
``` asm
...
#ifdef FW_FDT_PATH
	.section .rodata
	.align 4
	.globl fw_fdt_bin
fw_fdt_bin:
	.incbin FW_FDT_PATH
#ifdef FW_FDT_PADDING
	.fill FW_FDT_PADDING, 1, 0
#endif
#endif
...
```
`.incbin`ディレクティブで、FDTのバイナリを配置している。
`FW_FDT_PADDING`が指定されているときは、`FW_FDT_PADDING`回、1byteの0で埋める。

今回はここまで。次はプラットフォームのイニシャライズから。
