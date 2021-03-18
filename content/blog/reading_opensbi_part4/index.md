---
title: "Reading OpenSBI part4"
date: 2021-03-17T10:29:12+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---

## 概要
[前回]({{<ref "/blog/reading_opensbi_part3/index.md" >}})。
今回はプラットフォームの初期化に入る。
<!--more-->

`firmware/fw_base.S`
```asm
...
	/*
	 * Initialize platform
	 * Note: The a0 to a4 registers passed to the
	 * firmware are parameters to this function.
	 */
	MOV_5R	s0, a0, s1, a1, s2, a2, s3, a3, s4, a4
	call	fw_platform_init
	add	t0, a0, zero
	MOV_5R	a0, s0, a1, s1, a2, s2, a3, s3, a4, s4
	add	a1, t0, zero

	/* Preload HART details
	 * s7 -> HART Count
	 * s8 -> HART Stack Size
	 */
	la	a4, platform
#if __riscv_xlen == 64
	lwu	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lwu	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#else
	lw	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lw	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#endif

	/* Setup scratch space for all the HARTs*/
	la	tp, _fw_end
	mul	a5, s7, s8
	add	tp, tp, a5
	/* Keep a copy of tp */
	add	t3, tp, zero
	/* Counter */
	li	t2, 1
	/* hartid 0 is mandated by ISA */
	li	t1, 0
_scratch_init:
	add	tp, t3, zero
	mul	a5, s8, t1
	sub	tp, tp, a5
	li	a5, SBI_SCRATCH_SIZE
	sub	tp, tp, a5

	/* Initialize scratch space */
	/* Store fw_start and fw_size in scratch space */
	la	a4, _fw_start
	la	a5, _fw_end
	mul	t0, s7, s8
	add	a5, a5, t0
	sub	a5, a5, a4
	REG_S	a4, SBI_SCRATCH_FW_START_OFFSET(tp)
	REG_S	a5, SBI_SCRATCH_FW_SIZE_OFFSET(tp)
	/* Store next arg1 in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
	call	fw_next_arg1
	REG_S	a0, SBI_SCRATCH_NEXT_ARG1_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Store next address in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
	call	fw_next_addr
	REG_S	a0, SBI_SCRATCH_NEXT_ADDR_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Store next mode in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
	call	fw_next_mode
	REG_S	a0, SBI_SCRATCH_NEXT_MODE_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Store warm_boot address in scratch space */
	la	a4, _start_warm
	REG_S	a4, SBI_SCRATCH_WARMBOOT_ADDR_OFFSET(tp)
	/* Store platform address in scratch space */
	la	a4, platform
	REG_S	a4, SBI_SCRATCH_PLATFORM_ADDR_OFFSET(tp)
	/* Store hartid-to-scratch function address in scratch space */
	la	a4, _hartid_to_scratch
	REG_S	a4, SBI_SCRATCH_HARTID_TO_SCRATCH_OFFSET(tp)
	/* Store trap-exit function address in scratch space */
	la	a4, _trap_exit
	REG_S	a4, SBI_SCRATCH_TRAP_EXIT_OFFSET(tp)
	/* Clear tmp0 in scratch space */
	REG_S	zero, SBI_SCRATCH_TMP0_OFFSET(tp)
	/* Store firmware options in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
#ifdef FW_OPTIONS
	li	a0, FW_OPTIONS
#else
	call	fw_options
#endif
	REG_S	a0, SBI_SCRATCH_OPTIONS_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Move to next scratch space */
	add	t1, t1, t2
	blt	t1, s7, _scratch_init
...
```
### fw\_platform\_init
まずは、レジスタの退避`(s0, s1, s2, s3, s4) <- (a0, a1, a2, a3, a4)`\
`fw_platform_init`の呼び出し。
```asm
...
	.section .entry, "ax", %progbits
	.align 3
	.globl fw_platform_init
	.weak fw_platform_init
fw_platform_init:
	add	a0, a1, zero
	ret
...
```
`fw_platform_init`は`.weak`で定義されている。プラットフォームにて、必要に応じで上書き実装することとなる。今回は、特に実装しないのでこのままでよい。
`t0`レジスタに返り値(`a0`)を保存。\
レジスタの復帰`(a0, a1, a2, a3, a4) <- (s0, s1, s2, s3, s4)`\
`a1`に`t0`を保存。

### hartごとのscratch areaの作成(struct sbi\_scratch)
次に`a4`レジスタに`struct sbi_platform platform`のアドレスを代入する。
`platform`はプラットフォームの情報を持っている構造体であり、`include/sbi/sbi_platform.h`に宣言されている。
この構造体より、hartの数とhartのスタックサイズをそれぞれ`s7`、`s8`レジスタへロードする。

```c
...
/** Offset of hart_count in struct sbi_platform */
#define SBI_PLATFORM_HART_COUNT_OFFSET (0x50)
/** Offset of hart_stack_size in struct sbi_platform */
#define SBI_PLATFORM_HART_STACK_SIZE_OFFSET (0x54)
...
struct sbi_platform {
	/**
	 * OpenSBI version this sbi_platform is based on.
	 * It's a 32-bit value where upper 16-bits are major number
	 * and lower 16-bits are minor number
	 */
	u32 opensbi_version;
	/**
	 * OpenSBI platform version released by vendor.
	 * It's a 32-bit value where upper 16-bits are major number
	 * and lower 16-bits are minor number
	 */
	u32 platform_version;
	/** Name of the platform */
	char name[64];
	/** Supported features */
	u64 features;
	/** Total number of HARTs */
	u32 hart_count;
	/** Per-HART stack size for exception/interrupt handling */
	u32 hart_stack_size;
	/** Pointer to sbi platform operations */
	unsigned long platform_ops_addr;
	/** Pointer to system firmware specific context */
	unsigned long firmware_context;
	/**
	 * HART index to HART id table
	 *
	 * For used HART index <abc>:
	 *     hart_index2id[<abc>] = some HART id
	 * For unused HART index <abc>:
	 *     hart_index2id[<abc>] = -1U
	 *
	 * If hart_index2id == NULL then we assume identity mapping
	 *     hart_index2id[<abc>] = <abc>
	 *
	 * We have only two restrictions:
	 * 1. HART index < sbi_platform hart_count
	 * 2. HART id < SBI_HARTMASK_MAX_BITS
	 */
	const u32 *hart_index2id;
} __packed;
...
```
#### scratch spaceの作成
取り出した情報をもとにすべてのhartに対するscratch spaceを用意する。
`_fw_end`をベースアドレスして、`hart count * hart stack size`をオフセットする。
このアドレスを`tp`レジスタへ入れる。`tp`レジスタから相対でアドレス生成することで、scratch spaceにアクセスする。
`t3`レジスタへ`tp`を保存する。
`t2`レジスタに１、`t1`レジスタに0をいれる。
`t1`レジスタは現在処理しているhartを表し、`t1+t2`は次のhartidを示す。

#### \_scratch\_init
`_scratch_init`はhart count回行い、すべてのhartに対して行う。

`tp`レジスタに`t3`レジスタを代入する。これは`_scratch_init`前に保存した`tp`の(scratch space)のアドレス
次に、現在処理しているhart用のscratch spaceの`tp`からのオフセットを計算する。
`s8 * t1`にて、`hart stack size * current hartid`で計算する。
`tp`からSBI\_SCRATCH\_SIZEを引き去る。
`tp`はhartごとのscratch spaceのベースアドレスを指す。

{{<figure src="./image00.png" >}}
scratch構造体を各hartのscratch spaceへ作成する。

`include/sbi/sbi_scratch.h`
```c
...
/** Offset of fw_start member in sbi_scratch */
#define SBI_SCRATCH_FW_START_OFFSET		(0 * __SIZEOF_POINTER__)
/** Offset of fw_size member in sbi_scratch */
#define SBI_SCRATCH_FW_SIZE_OFFSET		(1 * __SIZEOF_POINTER__)
/** Offset of next_arg1 member in sbi_scratch */
#define SBI_SCRATCH_NEXT_ARG1_OFFSET		(2 * __SIZEOF_POINTER__)
/** Offset of next_addr member in sbi_scratch */
#define SBI_SCRATCH_NEXT_ADDR_OFFSET		(3 * __SIZEOF_POINTER__)
/** Offset of next_mode member in sbi_scratch */
#define SBI_SCRATCH_NEXT_MODE_OFFSET		(4 * __SIZEOF_POINTER__)
/** Offset of warmboot_addr member in sbi_scratch */
#define SBI_SCRATCH_WARMBOOT_ADDR_OFFSET	(5 * __SIZEOF_POINTER__)
/** Offset of platform_addr member in sbi_scratch */
#define SBI_SCRATCH_PLATFORM_ADDR_OFFSET	(6 * __SIZEOF_POINTER__)
/** Offset of hartid_to_scratch member in sbi_scratch */
#define SBI_SCRATCH_HARTID_TO_SCRATCH_OFFSET	(7 * __SIZEOF_POINTER__)
/** Offset of trap_exit member in sbi_scratch */
#define SBI_SCRATCH_TRAP_EXIT_OFFSET		(8 * __SIZEOF_POINTER__)
/** Offset of tmp0 member in sbi_scratch */
#define SBI_SCRATCH_TMP0_OFFSET			(9 * __SIZEOF_POINTER__)
/** Offset of options member in sbi_scratch */
#define SBI_SCRATCH_OPTIONS_OFFSET		(10 * __SIZEOF_POINTER__)
/** Offset of extra space in sbi_scratch */
#define SBI_SCRATCH_EXTRA_SPACE_OFFSET		(11 * __SIZEOF_POINTER__)
/** Maximum size of sbi_scratch (4KB) */
#define SBI_SCRATCH_SIZE			(0x1000)


...
/** Representation of per-HART scratch space */
struct sbi_scratch {
	/** Start (or base) address of firmware linked to OpenSBI library */
	unsigned long fw_start;
	/** Size (in bytes) of firmware linked to OpenSBI library */
	unsigned long fw_size;
	/** Arg1 (or 'a1' register) of next booting stage for this HART */
	unsigned long next_arg1;
	/** Address of next booting stage for this HART */
	unsigned long next_addr;
	/** Priviledge mode of next booting stage for this HART */
	unsigned long next_mode;
	/** Warm boot entry point address for this HART */
	unsigned long warmboot_addr;
	/** Address of sbi_platform */
	unsigned long platform_addr;
	/** Address of HART ID to sbi_scratch conversion function */
	unsigned long hartid_to_scratch;
	/** Address of trap exit function */
	unsigned long trap_exit;
	/** Temporary storage */
	unsigned long tmp0;
	/** Options for OpenSBI library */
	unsigned long options;
} __packed;
...
```

#### fw\_start、fw\_sizeフィールド
scratch spaceに`_fw_start`、およびFWのサイズを格納する。
`mul t0, s7, s8`にてすべてのhartのscratch space合計したサイズを計算する(hart count * hart stack size)。
`_fw_end`に`t0`を加えることで、scratch spaceのベースアドレスを計算する。
そこから、`_fw_start`(`a4`レジスタ)を引き去ることで、FWのサイズを計算する(`a5`レジスタ)。
`a4`、`a5`レジスタをそれぞれ、`struct sbi_scratch`の`fw_start`および`fw_size`フィールドに格納する。


#### next\_arg1フィールド
次に、next\_arg1を格納する。
レジスタを退避する。(s0, s1, s2) <- (a0, a1, a2)
`fw_next_arg1`関数を呼ぶ。(FW\_PAYLOAD)
`FW_PAYLOAD_FDT_ADDR`が定義されているとき、FDTのPAを次段のクライアントプログラムへ`a0`レジスタを通して引き渡す。
定義されていない場合は`0`を渡す。
関数から復帰し、`next_arg1`フィールドへ格納する。
レジスタを復帰する。(a0, a1, a2) <- (s0, s1, s2)

`firmware/fw_payload.S`
```asm
...
	.section .entry, "ax", %progbits
	.align 3
	.global fw_next_arg1
	/*
	 * We can only use a0, a1, and a2 registers here.
	 * The a0, a1, and a2 registers will be same as passed by
	 * previous booting stage.
	 * The next arg1 should be returned in 'a0'.
	 */
fw_next_arg1:
#ifdef FW_PAYLOAD_FDT_ADDR
	li	a0, FW_PAYLOAD_FDT_ADDR
#else
	add	a0, a1, zero
#endif
	ret
...
```

#### next\_addrフィールド
次に、`fw_next_addr`フィールドを格納する。(FW\_PAYLOAD)
レジスタを退避する。(s0, s1, s2) <- (a0, a1, a2)
`fw_next_addr`関数を呼び、PAYLOADの物理アドレスを`a0`に取得し、復帰する。
`next_addr`フィールドに格納し、レジスタを復帰する。(a0, a1, a2) <- (s0, s1, s2)

`firmware/fw_payload.S`
```asm
	.section .entry, "ax", %progbits
	.align 3
	.global fw_next_addr
	/*
	 * We can only use a0, a1, and a2 registers here.
	 * The next address should be returned in 'a0'.
	 */
fw_next_addr:
	la	a0, payload_bin
	ret
```

#### warmboot\_addrフィールド
`_start_warm`のアドレスを取得し、`warmboot_addr`フィールドへ格納する。
`_start_warm`の実装については次回に取っておこう。(`firmware/fw_base.S`)

#### platform\_addrフィールド
`platform`のアドレスを取得し、`platform_addr`フィールドへ格納する。
`platform`は`struct sbi_platform`の実態である。
#### hartid\_to\_scratchフィールド
`_hartid_to_scratch`のアドレスを取得し、`hartid_to_scratch`フィールドへ格納する。
`_hartid_to_scratch`は以下の通り、レジスタを使用する。
`a0`、`a1`は呼び出し元が引数として渡すものとする。
hartidからscratch spaceを求める関数である。
戻り地は`a0`レジスタに格納される。

|Register|meaning|
|---|---|
|a0|Hart ID|
|a1|Hart index|
|t0|Hart stack size|
|t1|Hart stack end|
|t2|Temporary|

`firmware/fw_base.S`
```asm
...
	.section .entry, "ax", %progbits
	.align 3
	.globl _hartid_to_scratch
_hartid_to_scratch:
	/*
	 * a0 -> HART ID (passed by caller)
	 * a1 -> HART Index (passed by caller)
	 * t0 -> HART Stack Size
	 * t1 -> HART Stack End
	 * t2 -> Temporary
	 */
	la	t2, platform
#if __riscv_xlen == 64
	lwu	t0, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(t2)
	lwu	t2, SBI_PLATFORM_HART_COUNT_OFFSET(t2)
#else
	lw	t0, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(t2)
	lw	t2, SBI_PLATFORM_HART_COUNT_OFFSET(t2)
#endif
	sub	t2, t2, a1
	mul	t2, t2, t0
	la	t1, _fw_end
	add	t1, t1, t2
	li	t2, SBI_SCRATCH_SIZE
	sub	a0, t1, t2
	ret
...
```

#### trap\_exitフィールド
`_trap_exit`のアドレスを取得し、`trap_exit`フィールドへ格納する。
`_trap_exit`は基本的にマクロで実現されている。
`SBI_TRAP_REGS_OFFSET`の定義は`include/sbi/sbi_trap.h`に存在する。
トラップ時のコンテキストはスタック内に`struct sbi_trap_regs`として保存される。
`SBI_TRAP_REGS_OFFSET`は引数のレジスタ番号に4(`__SIZEOF_PTR__`)をかけて構造体のオフセットを計算する。
`x0-x31`の場合はそれぞれ0~31が対応する。保存する`CSR`の場合は、それぞれ定数を定義している。
`_trap_exit`は引数(`struct sbi_trap_regs`)を`a0`に取る。
これを`sp`に代入し、`sp`相対アドレスとして、レジスタコンテキストの復帰を行う。
`_TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0`では、`sp`および`t0`以外のレジスタコンテキストを復帰する。
次に、`TRAP_RESTORE_MEPC_MSTATUS 0`により、`mepc`および`mstatus`を復帰する。マクロ引数に0(`have_mstatush`)を指定しているので、`mstatush`は復帰しない。
`TRAP_RESTORE_SP_TO`にて`t0`、`sp`を復帰する。
注意が必要であるが、`sp`は最後に復帰しなければならない。(`sp`相対でアドレスを生成しているため)
`mret`により、トラップ時のPCへ復帰する。

`firmware/fw_base.S`
```asm
...
.macro	TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0
	/* Restore all general regisers except SP and T0 */
	REG_L	ra, SBI_TRAP_REGS_OFFSET(ra)(sp)
	REG_L	gp, SBI_TRAP_REGS_OFFSET(gp)(sp)
	REG_L	tp, SBI_TRAP_REGS_OFFSET(tp)(sp)
	REG_L	t1, SBI_TRAP_REGS_OFFSET(t1)(sp)
	REG_L	t2, SBI_TRAP_REGS_OFFSET(t2)(sp)
	REG_L	s0, SBI_TRAP_REGS_OFFSET(s0)(sp)
	REG_L	s1, SBI_TRAP_REGS_OFFSET(s1)(sp)
	REG_L	a0, SBI_TRAP_REGS_OFFSET(a0)(sp)
	REG_L	a1, SBI_TRAP_REGS_OFFSET(a1)(sp)
	REG_L	a2, SBI_TRAP_REGS_OFFSET(a2)(sp)
	REG_L	a3, SBI_TRAP_REGS_OFFSET(a3)(sp)
	REG_L	a4, SBI_TRAP_REGS_OFFSET(a4)(sp)
	REG_L	a5, SBI_TRAP_REGS_OFFSET(a5)(sp)
	REG_L	a6, SBI_TRAP_REGS_OFFSET(a6)(sp)
	REG_L	a7, SBI_TRAP_REGS_OFFSET(a7)(sp)
	REG_L	s2, SBI_TRAP_REGS_OFFSET(s2)(sp)
	REG_L	s3, SBI_TRAP_REGS_OFFSET(s3)(sp)
	REG_L	s4, SBI_TRAP_REGS_OFFSET(s4)(sp)
	REG_L	s5, SBI_TRAP_REGS_OFFSET(s5)(sp)
	REG_L	s6, SBI_TRAP_REGS_OFFSET(s6)(sp)
	REG_L	s7, SBI_TRAP_REGS_OFFSET(s7)(sp)
	REG_L	s8, SBI_TRAP_REGS_OFFSET(s8)(sp)
	REG_L	s9, SBI_TRAP_REGS_OFFSET(s9)(sp)
	REG_L	s10, SBI_TRAP_REGS_OFFSET(s10)(sp)
	REG_L	s11, SBI_TRAP_REGS_OFFSET(s11)(sp)
	REG_L	t3, SBI_TRAP_REGS_OFFSET(t3)(sp)
	REG_L	t4, SBI_TRAP_REGS_OFFSET(t4)(sp)
	REG_L	t5, SBI_TRAP_REGS_OFFSET(t5)(sp)
	REG_L	t6, SBI_TRAP_REGS_OFFSET(t6)(sp)
.endm

.macro	TRAP_RESTORE_MEPC_MSTATUS have_mstatush
	/* Restore MEPC and MSTATUS CSRs */
	REG_L	t0, SBI_TRAP_REGS_OFFSET(mepc)(sp)
	csrw	CSR_MEPC, t0
	REG_L	t0, SBI_TRAP_REGS_OFFSET(mstatus)(sp)
	csrw	CSR_MSTATUS, t0
	.if \have_mstatush
	REG_L	t0, SBI_TRAP_REGS_OFFSET(mstatusH)(sp)
	csrw	CSR_MSTATUSH, t0
	.endif
.endm

.macro TRAP_RESTORE_SP_T0
	/* Restore T0 */
	REG_L	t0, SBI_TRAP_REGS_OFFSET(t0)(sp)

	/* Restore SP */
	REG_L	sp, SBI_TRAP_REGS_OFFSET(sp)(sp)
.endm
...
	.section .entry, "ax", %progbits
	.align 3
	.globl _trap_exit
_trap_exit:
	add	sp, a0, zero
	TRAP_RESTORE_GENERAL_REGS_EXCEPT_SP_T0
	TRAP_RESTORE_MEPC_MSTATUS 0
	TRAP_RESTORE_SP_T0
	mret
...
```
`include/sbi/sbi_trap.h`

```asm
...
/** Representation of register state at time of trap/interrupt */
#define SBI_TRAP_REGS_zero			0
#define SBI_TRAP_REGS_ra			1
#define SBI_TRAP_REGS_sp			2
#define SBI_TRAP_REGS_gp			3
#define SBI_TRAP_REGS_tp			4
#define SBI_TRAP_REGS_t0			5
#define SBI_TRAP_REGS_t1			6
#define SBI_TRAP_REGS_t2			7
#define SBI_TRAP_REGS_s0			8
#define SBI_TRAP_REGS_s1			9
#define SBI_TRAP_REGS_a0			10
#define SBI_TRAP_REGS_a1			11
#define SBI_TRAP_REGS_a2			12
#define SBI_TRAP_REGS_a3			13
#define SBI_TRAP_REGS_a4			14
#define SBI_TRAP_REGS_a5			15
#define SBI_TRAP_REGS_a6			16
#define SBI_TRAP_REGS_a7			17
#define SBI_TRAP_REGS_s2			18
#define SBI_TRAP_REGS_s3			19
#define SBI_TRAP_REGS_s4			20
#define SBI_TRAP_REGS_s5			21
#define SBI_TRAP_REGS_s6			22
#define SBI_TRAP_REGS_s7			23
#define SBI_TRAP_REGS_s8			24
#define SBI_TRAP_REGS_s9			25
#define SBI_TRAP_REGS_s10			26
#define SBI_TRAP_REGS_s11			27
#define SBI_TRAP_REGS_t3			28
#define SBI_TRAP_REGS_t4			29
#define SBI_TRAP_REGS_t5			30
#define SBI_TRAP_REGS_t6			31
#define SBI_TRAP_REGS_mepc			32
#define SBI_TRAP_REGS_mstatus			33
#define SBI_TRAP_REGS_mstatusH			34
#define SBI_TRAP_REGS_last			35
...
#define SBI_TRAP_REGS_OFFSET(x) ((SBI_TRAP_REGS_##x) * __SIZEOF_POINTER__)
#define SBI_TRAP_REGS_SIZE SBI_TRAP_REGS_OFFSET(last)
...

struct sbi_trap_regs {
	/** zero register state */
	unsigned long zero;
	/** ra register state */
	unsigned long ra;
	/** sp register state */
	unsigned long sp;
	/** gp register state */
	unsigned long gp;
	/** tp register state */
	unsigned long tp;
	/** t0 register state */
	unsigned long t0;
	/** t1 register state */
	unsigned long t1;
	/** t2 register state */
	unsigned long t2;
	/** s0 register state */
	unsigned long s0;
	/** s1 register state */
	unsigned long s1;
	/** a0 register state */
	unsigned long a0;
	/** a1 register state */
	unsigned long a1;
	/** a2 register state */
	unsigned long a2;
	/** a3 register state */
	unsigned long a3;
	/** a4 register state */
	unsigned long a4;
	/** a5 register state */
	unsigned long a5;
	/** a6 register state */
	unsigned long a6;
	/** a7 register state */
	unsigned long a7;
	/** s2 register state */
	unsigned long s2;
	/** s3 register state */
	unsigned long s3;
	/** s4 register state */
	unsigned long s4;
	/** s5 register state */
	unsigned long s5;
	/** s6 register state */
	unsigned long s6;
	/** s7 register state */
	unsigned long s7;
	/** s8 register state */
	unsigned long s8;
	/** s9 register state */
	unsigned long s9;
	/** s10 register state */
	unsigned long s10;
	/** s11 register state */
	unsigned long s11;
	/** t3 register state */
	unsigned long t3;
	/** t4 register state */
	unsigned long t4;
	/** t5 register state */
	unsigned long t5;
	/** t6 register state */
	unsigned long t6;
	/** mepc register state */
	unsigned long mepc;
	/** mstatus register state */
	unsigned long mstatus;
	/** mstatusH register state (only for 32-bit) */
	unsigned long mstatusH;
} __packed;
...
```
#### scratch\_tmp0フィールド
`NULL`をアドレスとして`scratch_tmp0`フィールドに格納する。

#### fw\_options
レジスタを退避。(s0, s1, s2) <- (a0, a1, a2)
`FW_OPTIONS`フラグが定義されているとき、それで`a0`レジスタ上書きする。 
それ以外は`fw_options`を呼び出す。(FW\_PAYLOAD)
`a0`レジスタを`options`フィールドへ格納する。


`firmware/fw_payload.S`
```asm
...
	.section .entry, "ax", %progbits
	.align 3
	.global fw_options
	/*
	 * We can only use a0, a1, and a2 registers here.
	 * The 'a4' register will have default options.
	 * The next address should be returned in 'a0'.
	 */
fw_options:
	add	a0, zero, zero
	ret
...
```
#### hart count回繰り返す。
現在処理中のhartid(`t1`)を+1(+`t2`)する。
hart count(`s7`)以上になるまで、`_scratch_init`を繰り返す。

最終的な、scratch spaceは以下の画像のようになる。
{{<figure src="./image01.png" >}}

次回は、FDTのリロケーションおよび`_start_warm`の実装について調べる。
