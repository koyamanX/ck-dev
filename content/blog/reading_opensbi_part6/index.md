---
title: "Reading OpenSBI part6"
date: 2021-03-18T09:33:30+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---

[前回]({{<ref "/blog/reading_opensbi_part5/index.md" >}})。
今回は、`_start_warm`を読んでいく。
<!--more-->

### \_start\_warm

`firmware/fw_base.S`
```asm
...
_start_warm:
	/* Reset all registers for non-boot HARTs */
	li	ra, 0
	call	_reset_regs

	/* Disable and clear all interrupts */
	csrw	CSR_MIE, zero
	csrw	CSR_MIP, zero

	/* Find HART count and HART stack size */
	la	a4, platform
#if __riscv_xlen == 64
	lwu	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lwu	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#else
	lw	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lw	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#endif
	REG_L	s9, SBI_PLATFORM_HART_INDEX2ID_OFFSET(a4)

	/* Find HART id */
	csrr	s6, CSR_MHARTID

	/* Find HART index */
	beqz	s9, 3f
	li	a4, 0
1:
#if __riscv_xlen == 64
	lwu	a5, (s9)
#else
	lw	a5, (s9)
#endif
	beq	a5, s6, 2f
	add	s9, s9, 4
	add	a4, a4, 1
	blt	a4, s7, 1b
	li	a4, -1
2:	add	s6, a4, zero
3:	bge	s6, s7, _start_hang
...
```
`_reset_regs`では、fence.iをしてから、汎用レジスタおよび、mscratchレジスタを0で初期化する。
その後、`mip`、`mie`をクリアする。
`struct sbi_platform`へのポインタを`a4`レジスタへ格納する。
`a4`レジスタを用いて`sbi_platform`構造体のフィールドから情報を取得する。
`s7`にhart count、`s8`にスタックサイズ、`s9`に`hart_index2id`テーブルのポインタをそれぞれ格納する。
`s6`レジスタに`mhartid`CSRの値を取り出す。

`hart_index2id`テーブルを索引することで、indexに対応するhartidを取り出す。
使われていないインデックスには-1を返す。使われているインデックスにはhartidを返す。
なお、`hart_index2id == NULL`の場合はidentity mappingであり、`index == hartid`となる。
hart idおよびhart idについての制限は以下のとおりである。
- hart index < sbi\_platform hart\_count
- hart id < SBI\_HARTMASK\_MAX\_BITS

まずは、`s9`がNULLか調べ、NULLの場合はラベル`3`へ分岐する。
NULL出ない場合は、`a4`レジスタ(index)を0で初期化する。
ラベル`1`にて`a5`レジスタに`hart_index2id`の索引結果を入れる。
`a5` == `s6`(mhartid)の場合はラベル`2`へ分岐する。
`add s9, s9, 4`がわからん。(`s9`は`hart_index2id`(配列)の先頭アドレスを持っているはず。)
`a4`(index)を+1する。
`a4`(index) < `s7`(hart count)の場合は、ラベル`1`へ戻る。
それ以外は`a4`レジスタを-1にする。(使われていないindex)
ラベル`2`において、`s6`レジスタ(mhartid)に`a4`レジスタの値を入れる。
ラベル`3`では、`s6`(hartid) <= `s7`(hart count)の場合は、`_start_hang`へ分岐し、ハングする。

### hartのscratch spaceの準備
```asm
...
	/* Find the scratch space based on HART index */
	la	tp, _fw_end
	mul	a5, s7, s8		/* hart count * stack size */
	add	tp, tp, a5		/* last address of scratch area of all harts */
	mul	a5, s8, s6		/* stack size * hartid */
	sub	tp, tp, a5		/* scratch space of hartid */
	li	a5, SBI_SCRATCH_SIZE
	sub	tp, tp, a5		/* first address of scratch space of hartid */

	/* update the mscratch */
	csrw	CSR_MSCRATCH, tp

	/* Setup stack */
	add	sp, tp, zero
...
```

次に、scratch spaceを見つける。
コード片にコメントを挿入した。`_fw_end`の後に`n-hartid-1`のscratch spaceがあり、`_fw_end + stack size * hart count`にhart0のscratch spaceがある。
`tp`はhartのscratch spaceの先頭アドレスを示す。
hartのscratch spaceの先頭アドレスを`mscratch`CSRに格納し、`sp`をscratch spaceにする。

{{<figure src="./image00.png" >}}

### トラップハンドラーの設定
```asm
...
	/* Setup trap handler */
	la	a4, _trap_handler
#if __riscv_xlen == 32
	csrr	a5, CSR_MISA
	srli	a5, a5, ('H' - 'A')
	andi	a5, a5, 0x1
	beq	a5, zero, _skip_trap_handler_rv32_hyp
	la	a4, _trap_handler_rv32_hyp
_skip_trap_handler_rv32_hyp:
#endif
	csrw	CSR_MTVEC, a4

#if __riscv_xlen == 32
	/* Override trap exit for H-extension */
	csrr	a5, CSR_MISA
	srli	a5, a5, ('H' - 'A')
	andi	a5, a5, 0x1
	beq	a5, zero, _skip_trap_exit_rv32_hyp
	la	a4, _trap_exit_rv32_hyp
	csrr	a5, CSR_MSCRATCH
	REG_S	a4, SBI_SCRATCH_TRAP_EXIT_OFFSET(a5)
_skip_trap_exit_rv32_hyp:
#endif
...
```
RV32の場合、`misa`レジスタを確認し、H-extensionが有効であれば、H-mode用のトラップハンドラー(`trap_handler_rv32_hyp`)を設定する。
それ以外は、`mtvec`に`_trap_handler`を設定する。
なお、トラップハンドラーは8-byte境界配置されており、下位3ビットは常に`000`である。よってDirect modeによるトラップが行われる。
H-extensionが有効の場合は、`struct sbi_scratch`の`trap_exit`を`trap_exit_rv32_hyp`で上書きする。

```asm
...	
.section .entry, "ax", %progbits
	.align 3
	.globl _trap_handler
_trap_handler:
	TRAP_SAVE_AND_SETUP_SP_T0
	TRAP_SAVE_MEPC_MSTATUS 0
	TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0
	TRAP_CALL_C_ROUTINE
	TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0
	TRAP_RESTORE_MEPC_MSTATUS 0
	TRAP_RESTORE_SP_T0
	mret
...
#if __riscv_xlen == 32
	.section .entry, "ax", %progbits
	.align 3
	.globl _trap_handler_rv32_hyp
_trap_handler_rv32_hyp:
	TRAP_SAVE_AND_SETUP_SP_T0
	TRAP_SAVE_MEPC_MSTATUS 1
	TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0
	TRAP_CALL_C_ROUTINE
	TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0
	TRAP_RESTORE_MEPC_MSTATUS 1
	TRAP_RESTORE_SP_T0
	mret
	.section .entry, "ax", %progbits
	.align 3
	.globl _trap_exit_rv32_hyp
_trap_exit_rv32_hyp:
	add	sp, a0, zero
	TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0
	TRAP_RESTORE_MEPC_MSTATUS 1
	TRAP_RESTORE_SP_T0
	mret
#endif
...
```
### \_trap\_handler

#### TRAP\_SAVE\_AND\_SETUP\_SP\_T0
`TRAP_SAVE_AND_SETUP_SP_T0`について調べていく。
`mscratch`レジスタに`struct sbi_scratch`のポインタが入っている。
`tp`と`mscratch`をスワップする。
`tmp0`フィールドはscratch spaceとして扱える。そこに`t0`レジスタを格納する。
`t0`レジスタにスタックポインタを作成する。
`((mstatus.mpp < PRIV_M) ? 1 : 0) - 1`により、M-modeからのトラップであれば、`-1`、それ以外は`0`とする。
`tp ^ (((mstatus.mpp < PRIV_M) ? 1 : 0) - 1) & (SP ^ TP))`にて、M-modeであれば、SP、それ以外はTPをスタックポインタとして`t0`へ格納する。
`t0`から`SBI_TRAP_REGS_SIZE`引き、`struct sbi_trap_regs`をスタックに確保し、`sp`を`struct sbi_trap_regs`の`sp`フィールドに格納する。
`sp`に`t0 - SBI_TRAP_REGS_SIZE`をスタックポインタとして格納する。
`t0`レジスタを`struct sbi_scratch`の`tmp0`フィールドから取り出し(`tp`をスタックポインタにしていることに注意)、`struct sbi_trap_regs`の`t0`フィールドに格納する。
`tp`と`mscratch`をスワップする。
これ以降、`t0`レジスタは自由に使え、`sp`を通して、例外スタックにアクセスできる。

```asm
...
.macro	TRAP_SAVE_AND_SETUP_SP_T0
	/* Swap TP and MSCRATCH */
	csrrw	tp, CSR_MSCRATCH, tp

	/* Save T0 in scratch space */
	REG_S	t0, SBI_SCRATCH_TMP0_OFFSET(tp)

	/*
	 * Set T0 to appropriate exception stack
	 *
	 * Came_From_M_Mode = ((MSTATUS.MPP < PRV_M) ? 1 : 0) - 1;
	 * Exception_Stack = TP ^ (Came_From_M_Mode & (SP ^ TP))
	 *
	 * Came_From_M_Mode = 0    ==>    Exception_Stack = TP
	 * Came_From_M_Mode = -1   ==>    Exception_Stack = SP
	 */
	csrr	t0, CSR_MSTATUS
	srl	t0, t0, MSTATUS_MPP_SHIFT
	and	t0, t0, PRV_M
	slti	t0, t0, PRV_M
	add	t0, t0, -1
	xor	sp, sp, tp
	and	t0, t0, sp
	xor	sp, sp, tp
	xor	t0, tp, t0

	/* Save original SP on exception stack */
	REG_S	sp, (SBI_TRAP_REGS_OFFSET(sp) - SBI_TRAP_REGS_SIZE)(t0)

	/* Set SP to exception stack and make room for trap registers */
	add	sp, t0, -(SBI_TRAP_REGS_SIZE)

	/* Restore T0 from scratch space */
	REG_L	t0, SBI_SCRATCH_TMP0_OFFSET(tp)

	/* Save T0 on stack */
	REG_S	t0, SBI_TRAP_REGS_OFFSET(t0)(sp)

	/* Swap TP and MSCRATCH */
	csrrw	tp, CSR_MSCRATCH, tp
.endm
...
```
#### TRAP\_SAVE\_MEPC\_MSTATUS
次に、`TRAP_SAVE_MEPC_MSTATUS have_mstatush`を調べていく。
`sp`は例外スタックを指しており、`struct sbi_trap_regs`にレジスタを保存していく。
`t0`レジスタはすでに保存済みであり、自由に使える。
`mepc`、`mstatus`レジスタを`struct sbi_trap_regs`の`mepc`、`mstatus`フィールドにそれぞれ保存する。
`have_mstatush`が`1`の場合は、`mstatush`を`mstatush`フィールドに保存する。
それ以外は、`0`を`mstatush`フィールドに保存する。

```asm
...
.macro	TRAP_SAVE_MEPC_MSTATUS have_mstatush
	/* Save MEPC and MSTATUS CSRs */
	csrr	t0, CSR_MEPC
	REG_S	t0, SBI_TRAP_REGS_OFFSET(mepc)(sp)
	csrr	t0, CSR_MSTATUS
	REG_S	t0, SBI_TRAP_REGS_OFFSET(mstatus)(sp)
	.if \have_mstatush
	csrr	t0, CSR_MSTATUSH
	REG_S	t0, SBI_TRAP_REGS_OFFSET(mstatusH)(sp)
	.else
	REG_S	zero, SBI_TRAP_REGS_OFFSET(mstatusH)(sp)
	.endif
.endm
...
```
#### TRAP\_SAVE\_GENERAL\_REGS\_EXCEPT\_SP\_T0
次に、`TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0`を見ていく。
```asm
...
.macro	TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0
	/* Save all general regisers except SP and T0 */
	REG_S	zero, SBI_TRAP_REGS_OFFSET(zero)(sp)
	REG_S	ra, SBI_TRAP_REGS_OFFSET(ra)(sp)
	REG_S	gp, SBI_TRAP_REGS_OFFSET(gp)(sp)
	REG_S	tp, SBI_TRAP_REGS_OFFSET(tp)(sp)
	REG_S	t1, SBI_TRAP_REGS_OFFSET(t1)(sp)
	REG_S	t2, SBI_TRAP_REGS_OFFSET(t2)(sp)
	REG_S	s0, SBI_TRAP_REGS_OFFSET(s0)(sp)
	REG_S	s1, SBI_TRAP_REGS_OFFSET(s1)(sp)
	REG_S	a0, SBI_TRAP_REGS_OFFSET(a0)(sp)
	REG_S	a1, SBI_TRAP_REGS_OFFSET(a1)(sp)
	REG_S	a2, SBI_TRAP_REGS_OFFSET(a2)(sp)
	REG_S	a3, SBI_TRAP_REGS_OFFSET(a3)(sp)
	REG_S	a4, SBI_TRAP_REGS_OFFSET(a4)(sp)
	REG_S	a5, SBI_TRAP_REGS_OFFSET(a5)(sp)
	REG_S	a6, SBI_TRAP_REGS_OFFSET(a6)(sp)
	REG_S	a7, SBI_TRAP_REGS_OFFSET(a7)(sp)
	REG_S	s2, SBI_TRAP_REGS_OFFSET(s2)(sp)
	REG_S	s3, SBI_TRAP_REGS_OFFSET(s3)(sp)
	REG_S	s4, SBI_TRAP_REGS_OFFSET(s4)(sp)
	REG_S	s5, SBI_TRAP_REGS_OFFSET(s5)(sp)
	REG_S	s6, SBI_TRAP_REGS_OFFSET(s6)(sp)
	REG_S	s7, SBI_TRAP_REGS_OFFSET(s7)(sp)
	REG_S	s8, SBI_TRAP_REGS_OFFSET(s8)(sp)
	REG_S	s9, SBI_TRAP_REGS_OFFSET(s9)(sp)
	REG_S	s10, SBI_TRAP_REGS_OFFSET(s10)(sp)
	REG_S	s11, SBI_TRAP_REGS_OFFSET(s11)(sp)
	REG_S	t3, SBI_TRAP_REGS_OFFSET(t3)(sp)
	REG_S	t4, SBI_TRAP_REGS_OFFSET(t4)(sp)
	REG_S	t5, SBI_TRAP_REGS_OFFSET(t5)(sp)
	REG_S	t6, SBI_TRAP_REGS_OFFSET(t6)(sp)
.endm
...
```
単に、`sp`、`t0`レジスタ以外の汎用レジスタを`struct sbi_trap_regs`の対応するフィールドに保存していく。

#### TRAP\_CALL\_C\_ROUTINE
次は、`TRAP_CALL_C_ROUTINE`を見ていく。
```asm
...
.macro	TRAP_CALL_C_ROUTINE
	/* Call C routine */
	add	a0, sp, zero
	call	sbi_trap_handler
.endm
...
```
`sp`を第一引数(`a0`に格納)して`sbi_trap_handler`(C関数(`lib/sbi/sbi_trap.c`))を呼び出す。
ここから、C言語で処理を行う。

#### 復帰処理
`TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0`、`TRAP_RESTORE_MEPC_MSTATUS 0`、`TRAP_RESTORE_SP_T0`、`mret`の復帰のシーケンスは
`_trap_exit`と同じである。

### \_trap\_handler\_rv32\_hyp
とりあえず、スキップ
### \_trap\_exit\_rv32\_hyp
とりあえず、スキップ


### sbi\_initの実行(C関数)

```asm
...
	/* Initialize SBI runtime */
	csrr	a0, CSR_MSCRATCH
	call	sbi_init
	/* We don't expect to reach here hence just hang */
	j	_start_hang
...
```
`sbi_init`や`sbi_trap_handler`などを呼び出す前に、`mscratch`には`struct sbi_scratch`のアドレスが格納されている必要がある。
また、`sp`はhartのかぶっていないスタックを指す。
- [Constrains on OpenSBI](https://github.com/riscv/opensbi/blob/master/docs/library_usage.md)

C言語で書かれた`sbi_init`を呼び出す。
なお、`sbi_init`は復帰しない。

次は`sbi_trap_handler`と`sbi_init`からPAYLOADへ制御の引き渡しを見ていく。
