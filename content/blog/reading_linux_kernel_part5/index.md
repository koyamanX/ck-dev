---
title: "Reading linux kernel part5"
date: 2021-04-01T21:49:28+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: false
---

`setup_vm`、`relocate`を読み終えたので、次は`start_kernel`実行までを読んでいく。
<!--more-->

## setup\_trap\_vector

`arch/riscv/kernel/head.S`
```asm
	call setup_vm
#ifdef CONFIG_MMU
	la a0, early_pg_dir
	call relocate
#endif /* CONFIG_MMU */

	call setup_trap_vector
	/* Restore C environment */
	la tp, init_task
	sw zero, TASK_TI_CPU(tp)
	la sp, init_thread_union + THREAD_SIZE
```
次に呼ばれる関数は`setup_trap_vector`である。

`arch/riscv/kernel/head.S`
```asm
.align 2
setup_trap_vector:
	/* Set trap vector to exception handler */
	la a0, handle_exception
	csrw CSR_TVEC, a0

	/*
	 * Set sup0 scratch register to 0, indicating to exception vector that
	 * we are presently executing in kernel.
	 */
	csrw CSR_SCRATCH, zero
	ret
```
つまり、`handle_exception`のアドレスを`stvec`にダイレクトモードとしてセットする。
`csrw CSR_SCRATCH, zero`の`SSCRATCH`の構造はよくわからないが、コメントによると`sup0`が`0`であると、カーネルモードとするらしい。
`handle_exception`はどうなっているか見てみる。
かなり長い。
`arch/riscv/kernel/entry.S`
```asm
ENTRY(handle_exception)
	/*
	 * If coming from userspace, preserve the user thread pointer and load
	 * the kernel thread pointer.  If we came from the kernel, the scratch
	 * register will contain 0, and we should continue on the current TP.
	 */
	csrrw tp, CSR_SCRATCH, tp
	bnez tp, _save_context

_restore_kernel_tpsp:
	csrr tp, CSR_SCRATCH
	REG_S sp, TASK_TI_KERNEL_SP(tp)
_save_context:
	REG_S sp, TASK_TI_USER_SP(tp)
	REG_L sp, TASK_TI_KERNEL_SP(tp)
	addi sp, sp, -(PT_SIZE_ON_STACK)
	REG_S x1,  PT_RA(sp)
	REG_S x3,  PT_GP(sp)
	REG_S x5,  PT_T0(sp)
	REG_S x6,  PT_T1(sp)
	REG_S x7,  PT_T2(sp)
	REG_S x8,  PT_S0(sp)
	REG_S x9,  PT_S1(sp)
	REG_S x10, PT_A0(sp)
	REG_S x11, PT_A1(sp)
	REG_S x12, PT_A2(sp)
	REG_S x13, PT_A3(sp)
	REG_S x14, PT_A4(sp)
	REG_S x15, PT_A5(sp)
	REG_S x16, PT_A6(sp)
	REG_S x17, PT_A7(sp)
	REG_S x18, PT_S2(sp)
	REG_S x19, PT_S3(sp)
	REG_S x20, PT_S4(sp)
	REG_S x21, PT_S5(sp)
	REG_S x22, PT_S6(sp)
	REG_S x23, PT_S7(sp)
	REG_S x24, PT_S8(sp)
	REG_S x25, PT_S9(sp)
	REG_S x26, PT_S10(sp)
	REG_S x27, PT_S11(sp)
	REG_S x28, PT_T3(sp)
	REG_S x29, PT_T4(sp)
	REG_S x30, PT_T5(sp)
	REG_S x31, PT_T6(sp)

	/*
	 * Disable user-mode memory access as it should only be set in the
	 * actual user copy routines.
	 *
	 * Disable the FPU to detect illegal usage of floating point in kernel
	 * space.
	 */
	li t0, SR_SUM | SR_FS

	REG_L s0, TASK_TI_USER_SP(tp)
	csrrc s1, CSR_STATUS, t0
	csrr s2, CSR_EPC
	csrr s3, CSR_TVAL
	csrr s4, CSR_CAUSE
	csrr s5, CSR_SCRATCH
	REG_S s0, PT_SP(sp)
	REG_S s1, PT_STATUS(sp)
	REG_S s2, PT_EPC(sp)
	REG_S s3, PT_BADADDR(sp)
	REG_S s4, PT_CAUSE(sp)
	REG_S s5, PT_TP(sp)

	/*
	 * Set the scratch register to 0, so that if a recursive exception
	 * occurs, the exception vector knows it came from the kernel
	 */
	csrw CSR_SCRATCH, x0

```
カーネルモードからのトラップの場合は、`SSCRATCH`は`0`を示す。
また、`tp`はカーネルモードのThread pointerとなる。(`_restore_kernel_tpsp`) 
一方、ユーザーモードからのトラップの場合は、`SSCRATCH`はカーネルモードのThread Pointerを示す。
また、`tp`の値はユーザーモードが使用している。
`tp`にカーネルモードのThread Pointerをセットしたら、次にレジスタを保存する。
まず、ユーザーモードの`sp`を保存し、カーネルモードの`sp`をロードする。
次に、スタックを必要分確保し、汎用レジスタを保存する(`sp`、`tp`以外)。
`pt_regs`構造体として保存する。
アセンブリ中ではオフセットは`PT_`で表される。

`arch/riscv/include/asm/ptrace.h`
```asm
struct pt_regs {
	unsigned long epc;
	unsigned long ra;
	unsigned long sp;
	unsigned long gp;
	unsigned long tp;
	unsigned long t0;
	unsigned long t1;
	unsigned long t2;
	unsigned long s0;
	unsigned long s1;
	unsigned long a0;
	unsigned long a1;
	unsigned long a2;
	unsigned long a3;
	unsigned long a4;
	unsigned long a5;
	unsigned long a6;
	unsigned long a7;
	unsigned long s2;
	unsigned long s3;
	unsigned long s4;
	unsigned long s5;
	unsigned long s6;
	unsigned long s7;
	unsigned long s8;
	unsigned long s9;
	unsigned long s10;
	unsigned long s11;
	unsigned long t3;
	unsigned long t4;
	unsigned long t5;
	unsigned long t6;
	/* Supervisor/Machine CSRs */
	unsigned long status;
	unsigned long badaddr;
	unsigned long cause;
	/* a0 value before the syscall */
	unsigned long orig_a0;
};
```
また、`TASK_TI`で表される構造体は(`struct thread_info`)であり、`task_struct`構造体に組み込まれている。

かなり長いので、最初の方だけ。
ちなみに、Linuxのプロセスは`task_struct`にて表現される。
また、カーネルレベルのスレッドの実装にも使われる。

- [■Linux task構造体](http://www.coins.tsukuba.ac.jp/~yas/coins/os2-2013/2013-12-26/)

`include/linux/sched.h`
```asm
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/*
	 * For reasons of header soup (see current_thread_info()), this
	 * must be the first element of task_struct.
	 */
	struct thread_info		thread_info;
#endif
	/* -1 unrunnable, 0 runnable, >0 stopped: */
	volatile long			state;

	/*
	 * This begins the randomizable portion of task_struct. Only
	 * scheduling-critical items should be added above here.
	 */
	randomized_struct_fields_start

	void				*stack;
	refcount_t			usage;
	/* Per task flags (PF_*), defined further below: */
	unsigned int			flags;
	unsigned int			ptrace;

```

`thread_info`構造体は`arch/riscv/include/asm/thread_info.h`にある。
`arch/riscv/include/asm/thread_info.h`
```asm
/*
 * low level task data that entry.S needs immediate access to
 * - this struct should fit entirely inside of one cache line
 * - if the members of this struct changes, the assembly constants
 *   in asm-offsets.c must be updated accordingly
 * - thread_info is included in task_struct at an offset of 0.  This means that
 *   tp points to both thread_info and task_struct.
 */
struct thread_info {
	unsigned long		flags;		/* low level flags */
	int                     preempt_count;  /* 0=>preemptible, <0=>BUG */
	mm_segment_t		addr_limit;
	/*
	 * These stack pointers are overwritten on every system call or
	 * exception.  SP is also saved to the stack it can be recovered when
	 * overwritten.
	 */
	long			kernel_sp;	/* Kernel stack pointer */
	long			user_sp;	/* User stack pointer */
	int			cpu;
};
```

メモリの構造を以下に示す。
{{<figure src="./image00.png" >}}

まず、User modeの`sp`を`task_struct.thread_info.user_sp`に保存し、
`sp`に`task_struct.thread_info.kernel_sp`を入れ、ユーザーモードのスタックからカーネルのスタックに切り替える。
`kernel_sp`は`struct pt_regs`を保持するメモリ領域のポインタとなっている。
`pt_regs`へ`sp`、`tp`を覗いた汎用レジスタすべて保存する。

次に、`t0`にsstatus.SUM、sstatus.FSビットを1にする。
ユーザーモードのスタックポインタを`s0`へ、
`s1`に`sstatus`を読み出し、同時に`t0`レジスタで`sstatus`をクリアする。
この段階で、Supervisor ModeでUビットのたった、ページにアクセスするとフォルトする。また、FPUは使用できない。
`s2`に`sepc`、`s3`に`stval`、`s4`に`scause`、`s5`に`sscratch`を読み込み、次に、`pt_regs`構造体の所定の位置に保存する。
最後に、`sscratch`に`zero`をセットする。
これにて、再帰的なカーネルモードののトラップに対応できる。

次は、グローバルポインタ(global pointer)のロードから読んでいく。
