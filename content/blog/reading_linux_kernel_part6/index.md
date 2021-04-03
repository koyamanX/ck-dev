---
title: "Reading linux kernel part6"
date: 2021-04-02T13:59:10+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: false
---
`handle_exception`の続きを読んでいく。
<!--more-->

## handle\_exception

global pointerのロードから。

`arch/riscv/kernel/entry.S`
```asm
	/* Load the global pointer */
.option push
.option norelax
	la gp, __global_pointer$
.option pop

#ifdef CONFIG_TRACE_IRQFLAGS
	call trace_hardirqs_off
#endif

#ifdef CONFIG_CONTEXT_TRACKING
	/* If previous state is in user mode, call context_tracking_user_exit. */
	li   a0, SR_PP
	and a0, s1, a0
	bnez a0, skip_context_tracking
	call context_tracking_user_exit
skip_context_tracking:
#endif

	/*
	 * MSB of cause differentiates between
	 * interrupts and exceptions
	 */
	bge s4, zero, 1f

	la ra, ret_from_exception

	/* Handle interrupts */
	move a0, sp /* pt_regs */
	la a1, handle_arch_irq
	REG_L a1, (a1)
	jr a1
1:
#ifdef CONFIG_TRACE_IRQFLAGS
	call trace_hardirqs_on
#endif
	/*
	 * Exceptions run with interrupts enabled or disabled depending on the
	 * state of SR_PIE in m/sstatus.
	 */
	andi t0, s1, SR_PIE
	beqz t0, 1f
	csrs CSR_STATUS, SR_IE
```

`gp`をロードする。global pointerはリロケーション済みになる。
トラッキングについては省く。
次に、`s4`レジスタ(`scause`)の最上位ビットを調べる。
最上位ビットが立っているときは割込みである。
`bge s4, zero, 1f`で確かめている。
例外の場合は、`1:`である。

まずは、例外の処理を見ていく。
### 例外の処理

sstatus(`s1`)にpie(Previous Interrupt Enable)が立っているか調べる。
例外発生時に割込みが有効であったか調べる。
もし、`sstatus.pie`が0であれば、`sstatus.sie`をセットする。

次にリターンアドレスをセットする。(`ret_from_exception`)
`s4`(`scause`)が`EXC_SYSCALL`であるか調べる。
その場合は、システムコールなので`handle_syscall`へ分岐する。
その他の例外の場合は、`excp_vect_table`を索引する。
`s4`(`scause`)を右に二回(RV32の場合は、`RISCV_LGPTR`は`2`、`arch/riscv/include/asm/asm.h`)左にシフトする。
`a0`に`pt_regs`のポインタをセットし、`excp_vect_table`〜`excp_vect_table_end`の間にあるかをみて、索引する。
その後、レジスタ相対ジャンプを行う。

`excp_vect_table`〜`excp_vect_table_end`の間にインデックス(`t0`)が収まらない場合は、`1:`へ分岐し、`do_trap_unknown`を実行する。

#### excp\_vect\_table
システムコール以外の、各例外のハンドラは`excp_vect_table`に登録されている。
索引する際に、範囲外であれば、`do_trap_unknown`となる。
`arch/riscv/kernel/entry.S`
```asm
#ifndef CONFIG_MMU
#define do_page_fault do_trap_unknown
#endif

	.section ".rodata"
	/* Exception vector table */
ENTRY(excp_vect_table)
	RISCV_PTR do_trap_insn_misaligned
	RISCV_PTR do_trap_insn_fault
	RISCV_PTR do_trap_insn_illegal
	RISCV_PTR do_trap_break
	RISCV_PTR do_trap_load_misaligned
	RISCV_PTR do_trap_load_fault
	RISCV_PTR do_trap_store_misaligned
	RISCV_PTR do_trap_store_fault
	RISCV_PTR do_trap_ecall_u /* system call, gets intercepted */
	RISCV_PTR do_trap_ecall_s
	RISCV_PTR do_trap_unknown
	RISCV_PTR do_trap_ecall_m
	RISCV_PTR do_page_fault   /* instruction page fault */
	RISCV_PTR do_page_fault   /* load page fault */
	RISCV_PTR do_trap_unknown
	RISCV_PTR do_page_fault   /* store page fault */
excp_vect_table_end:
END(excp_vect_table)
```

#### handle\_syscall
`arch/riscv/kernel/entry.S`
```asm
handle_syscall:
#if defined(CONFIG_TRACE_IRQFLAGS) || defined(CONFIG_CONTEXT_TRACKING)
	/* Recover a0 - a7 for system calls */
	REG_L a0, PT_A0(sp)
	REG_L a1, PT_A1(sp)
	REG_L a2, PT_A2(sp)
	REG_L a3, PT_A3(sp)
	REG_L a4, PT_A4(sp)
	REG_L a5, PT_A5(sp)
	REG_L a6, PT_A6(sp)
	REG_L a7, PT_A7(sp)
#endif
	 /* save the initial A0 value (needed in signal handlers) */
	REG_S a0, PT_ORIG_A0(sp)
	/*
	 * Advance SEPC to avoid executing the original
	 * scall instruction on sret
	 */
	addi s2, s2, 0x4
	REG_S s2, PT_EPC(sp)
	/* Trace syscalls, but only if requested by the user. */
	REG_L t0, TASK_TI_FLAGS(tp)
	andi t0, t0, _TIF_SYSCALL_WORK
	bnez t0, handle_syscall_trace_enter
```

`a0`レジスタを`pt_regs->orig_a0`に保存する。
`sepc`(`s2`)を+4して、`pt_regs->epc`に保存する。

次に、`thread_info->flags`を取り出し、システムコールをトレースが必要か調べる。

`arch/riscv/include/asm/thread_info.h`
```asm
/*
 * thread information flags
 * - these are process state flags that various assembly files may need to
 *   access
 * - pending work-to-be-done flags are in lowest half-word
 * - other flags in upper half-word(s)
 */
#define TIF_SYSCALL_TRACE	0	/* syscall trace active */
#define TIF_NOTIFY_RESUME	1	/* callback before returning to user */
#define TIF_SIGPENDING		2	/* signal pending */
#define TIF_NEED_RESCHED	3	/* rescheduling necessary */
#define TIF_RESTORE_SIGMASK	4	/* restore signal mask in do_signal() */
#define TIF_MEMDIE		5	/* is terminating due to OOM killer */
#define TIF_SYSCALL_TRACEPOINT  6       /* syscall tracepoint instrumentation */
#define TIF_SYSCALL_AUDIT	7	/* syscall auditing */
#define TIF_SECCOMP		8	/* syscall secure computing */

#define _TIF_SYSCALL_TRACE	(1 << TIF_SYSCALL_TRACE)
#define _TIF_NOTIFY_RESUME	(1 << TIF_NOTIFY_RESUME)
#define _TIF_SIGPENDING		(1 << TIF_SIGPENDING)
#define _TIF_NEED_RESCHED	(1 << TIF_NEED_RESCHED)
#define _TIF_SYSCALL_TRACEPOINT	(1 << TIF_SYSCALL_TRACEPOINT)
#define _TIF_SYSCALL_AUDIT	(1 << TIF_SYSCALL_AUDIT)
#define _TIF_SECCOMP		(1 << TIF_SECCOMP)

#define _TIF_SYSCALL_WORK \
	(_TIF_SYSCALL_TRACE | _TIF_SYSCALL_TRACEPOINT | _TIF_SYSCALL_AUDIT | \
	 _TIF_SECCOMP)

```

トレースが必要な場合は`handle_syscall_trace_enter`に分岐する。
このパスは`ptrace`に用いられる。

`arch/riscv/kernel/entry.S`
```asm
/* Slow paths for ptrace. */
handle_syscall_trace_enter:
	move a0, sp
	call do_syscall_trace_enter
	move t0, a0
	REG_L a0, PT_A0(sp)
	REG_L a1, PT_A1(sp)
	REG_L a2, PT_A2(sp)
	REG_L a3, PT_A3(sp)
	REG_L a4, PT_A4(sp)
	REG_L a5, PT_A5(sp)
	REG_L a6, PT_A6(sp)
	REG_L a7, PT_A7(sp)
	bnez t0, ret_from_syscall_rejected
	j check_syscall_nr
handle_syscall_trace_exit:
	move a0, sp
	call do_syscall_trace_exit
	j ret_from_exception
```

この場合でも`check_syscall_nr`を呼びだす。
よって、このパスはとりあえず無視する。

`arch/riscv/kernel/entry.S`
```asm
check_syscall_nr:
	/* Check to make sure we don't jump to a bogus syscall number. */
	li t0, __NR_syscalls
	la s0, sys_ni_syscall
	/*
	 * Syscall number held in a7.
	 * If syscall number is above allowed value, redirect to ni_syscall.
	 */
	bge a7, t0, 1f
	/*
	 * Check if syscall is rejected by tracer, i.e., a7 == -1.
	 * If yes, we pretend it was executed.
	 */
	li t1, -1
	beq a7, t1, ret_from_syscall_rejected
	blt a7, t1, 1f
	/* Call syscall */
	la s0, sys_call_table
	slli t0, a7, RISCV_LGPTR
	add s0, s0, t0
	REG_L s0, 0(s0)
1:
	jalr s0
```
`__NR_syscalls`はシステムコール番号の最大値である。

`arch/riscv/kernel/syscall_table.c`
```c
#undef __SYSCALL
#define __SYSCALL(nr, call)	[nr] = (call),

void *sys_call_table[__NR_syscalls] = {
	[0 ... __NR_syscalls - 1] = sys_ni_syscall,
#include <asm/unistd.h>
};
```

`sys_ni_syscall`は実装されていないシステムコールの処理である。
`kernel/sys_ni.c`
```c
/*  we can't #include <linux/syscalls.h> here,
    but tell gcc to not warn with -Wmissing-prototypes  */
asmlinkage long sys_ni_syscall(void);

/*
 * Non-implemented system calls get redirected here.
 */
asmlinkage long sys_ni_syscall(void)
{
	return -ENOSYS;
}
```

システムコール発行の引数(`a7`)のシステムコール番号が`__NR_syscalls`より以上の場合は実装されていない。(`1:`)

システムコール番号(`a7`)が`-1`の場合は`ret_from_syscall_rejected`へ分岐する。

次にシステムコール番号(`a7`)で`sys_call_table`を索引し、ジャンプする。


#### 例外からの復帰
`arch/riscv/kernel/entry.S`
```c
ret_from_syscall:
	/* Set user a0 to kernel a0 */
	REG_S a0, PT_A0(sp)
	/*
	 * We didn't execute the actual syscall.
	 * Seccomp already set return value for the current task pt_regs.
	 * (If it was configured with SECCOMP_RET_ERRNO/TRACE)
	 */
ret_from_syscall_rejected:
	/* Trace syscalls, but only if requested by the user. */
	REG_L t0, TASK_TI_FLAGS(tp)
	andi t0, t0, _TIF_SYSCALL_WORK
	bnez t0, handle_syscall_trace_exit

ret_from_exception:
	REG_L s0, PT_STATUS(sp)
	csrc CSR_STATUS, SR_IE
#ifdef CONFIG_TRACE_IRQFLAGS
	call trace_hardirqs_off
#endif
#ifdef CONFIG_RISCV_M_MODE
	/* the MPP value is too large to be used as an immediate arg for addi */
	li t0, SR_MPP
	and s0, s0, t0
#else
	andi s0, s0, SR_SPP
#endif
	bnez s0, resume_kernel
```

次に、システムコールの戻り値を`pt_regs->a0`に保存し、割込みを無効にする。
次に移るモード(`sstatus.spp`)をマスクする。
このモードが`0`の場合はユーザーモードなので、`resume_userspace`を実行する。
それ以外は`resume_kernel`を実行する。


```asm
#if IS_ENABLED(CONFIG_PREEMPTION)
resume_kernel:
	REG_L s0, TASK_TI_PREEMPT_COUNT(tp)
	bnez s0, restore_all
	REG_L s0, TASK_TI_FLAGS(tp)
	andi s0, s0, _TIF_NEED_RESCHED
	beqz s0, restore_all
	call preempt_schedule_irq
	j restore_all
#endif
```
`CONFIG_PREEMPTION`を定義すると、プロセスコンテキストにおいてカーネル内のコードを実行中であっても他のプロセスにCPUを手放す。

- [プリエンプション](https://wiki.bit-hive.com/linuxkernelmemo/pg/%E3%83%97%E3%83%AA%E3%82%A8%E3%83%B3%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3)

今回は、`CONFIG_PREEMPTION`を定義していないので、`resume_kernel`の実体は`restore_all`となる。

`arch/riscv/kernel/entry.S`
```asm
#if !IS_ENABLED(CONFIG_PREEMPTION)
.set resume_kernel, restore_all
#endif
```

`arch/riscv/kernel/entry.S`
```asm
restore_all:
#ifdef CONFIG_TRACE_IRQFLAGS
	REG_L s1, PT_STATUS(sp)
	andi t0, s1, SR_PIE
	beqz t0, 1f
	call trace_hardirqs_on
	j 2f
1:
	call trace_hardirqs_off
2:
#endif
```

`CONFIG_TRACE_IRQFLAGS`は定義していないので、スキップする。

`arch/riscv/kernel/entry.S`
```asm
	REG_L a0, PT_STATUS(sp)
	/*
	 * The current load reservation is effectively part of the processor's
	 * state, in the sense that load reservations cannot be shared between
	 * different hart contexts.  We can't actually save and restore a load
	 * reservation, so instead here we clear any existing reservation --
	 * it's always legal for implementations to clear load reservations at
	 * any point (as long as the forward progress guarantee is kept, but
	 * we'll ignore that here).
	 *
	 * Dangling load reservations can be the result of taking a trap in the
	 * middle of an LR/SC sequence, but can also be the result of a taken
	 * forward branch around an SC -- which is how we implement CAS.  As a
	 * result we need to clear reservations between the last CAS and the
	 * jump back to the new context.  While it is unlikely the store
	 * completes, implementations are allowed to expand reservations to be
	 * arbitrarily large.
	 */
	REG_L  a2, PT_EPC(sp)
	REG_SC x0, a2, PT_EPC(sp)

	csrw CSR_STATUS, a0
	csrw CSR_EPC, a2

	REG_L x1,  PT_RA(sp)
	REG_L x3,  PT_GP(sp)
	REG_L x4,  PT_TP(sp)
	REG_L x5,  PT_T0(sp)
	REG_L x6,  PT_T1(sp)
	REG_L x7,  PT_T2(sp)
	REG_L x8,  PT_S0(sp)
	REG_L x9,  PT_S1(sp)
	REG_L x10, PT_A0(sp)
	REG_L x11, PT_A1(sp)
	REG_L x12, PT_A2(sp)
	REG_L x13, PT_A3(sp)
	REG_L x14, PT_A4(sp)
	REG_L x15, PT_A5(sp)
	REG_L x16, PT_A6(sp)
	REG_L x17, PT_A7(sp)
	REG_L x18, PT_S2(sp)
	REG_L x19, PT_S3(sp)
	REG_L x20, PT_S4(sp)
	REG_L x21, PT_S5(sp)
	REG_L x22, PT_S6(sp)
	REG_L x23, PT_S7(sp)
	REG_L x24, PT_S8(sp)
	REG_L x25, PT_S9(sp)
	REG_L x26, PT_S10(sp)
	REG_L x27, PT_S11(sp)
	REG_L x28, PT_T3(sp)
	REG_L x29, PT_T4(sp)
	REG_L x30, PT_T5(sp)
	REG_L x31, PT_T6(sp)

	REG_L x2,  PT_SP(sp)

#ifdef CONFIG_RISCV_M_MODE
	mret
#else
	sret
#endif
```
`SC`(Store conditional)を発行して、`reservation`をクリアする。
`reservation`(`load reserved`によりセット)はプロセッサのステートなので、復元が必要。（復元の代わりにSCを発行することでreservationをクリアする。
任意の大きさの場合は`AMO fault`となるはず。)
`sstatus`、`sepc`、汎用レジスタを復元する。
システムコールの戻り値は`pt_regs->a0`に保存済みである。
最後に`sp`を復元していることに注意。
その後、`sret`を行い、例外元のカーネルへ復帰する。(`sstatus.spp == S-mode`)


次にユーザーモードへ戻る場合を見てみる。

```asm
resume_userspace:
	/* Interrupts must be disabled here so flags are checked atomically */
	REG_L s0, TASK_TI_FLAGS(tp) /* current_thread_info->flags */
	andi s1, s0, _TIF_WORK_MASK
	bnez s1, work_pending

#ifdef CONFIG_CONTEXT_TRACKING
	call context_tracking_user_enter
#endif

	/* Save unwound kernel stack pointer in thread_info */
	addi s0, sp, PT_SIZE_ON_STACK
	REG_S s0, TASK_TI_KERNEL_SP(tp)

	/*
	 * Save TP into the scratch register , so we can find the kernel data
	 * structures again.
	 */
	csrw CSR_SCRATCH, tp
```

`thread_info->flags`を見て、シグナル、コールバック、リスケジュールが必要な際には`work_pending`を実行する。
それ以外の場合は、`s0`にスタックポインタを戻したものを作成し、`task_struct`の`kernel sp`へ保存する。
`tp`を`scratch`とスワップし、カーネルのデータ構造へのポインタを`sscratch`へ保存する。
その後は`restore_all`と同じである。ただし、`sstatus->spp == U-mode`なので、ユーザースペースへ復帰する。

`arch/riscv/include/asm/thread_info.h`
```asm
/*
 * thread information flags
 * - these are process state flags that various assembly files may need to
 *   access
 * - pending work-to-be-done flags are in lowest half-word
 * - other flags in upper half-word(s)
 */
#define TIF_SYSCALL_TRACE	0	/* syscall trace active */
#define TIF_NOTIFY_RESUME	1	/* callback before returning to user */
#define TIF_SIGPENDING		2	/* signal pending */
#define TIF_NEED_RESCHED	3	/* rescheduling necessary */
#define TIF_RESTORE_SIGMASK	4	/* restore signal mask in do_signal() */
#define TIF_MEMDIE		5	/* is terminating due to OOM killer */
#define TIF_SYSCALL_TRACEPOINT  6       /* syscall tracepoint instrumentation */
#define TIF_SYSCALL_AUDIT	7	/* syscall auditing */
#define TIF_SECCOMP		8	/* syscall secure computing */

#define _TIF_SYSCALL_TRACE	(1 << TIF_SYSCALL_TRACE)
#define _TIF_NOTIFY_RESUME	(1 << TIF_NOTIFY_RESUME)
#define _TIF_SIGPENDING		(1 << TIF_SIGPENDING)
#define _TIF_NEED_RESCHED	(1 << TIF_NEED_RESCHED)
#define _TIF_SYSCALL_TRACEPOINT	(1 << TIF_SYSCALL_TRACEPOINT)
#define _TIF_SYSCALL_AUDIT	(1 << TIF_SYSCALL_AUDIT)
#define _TIF_SECCOMP		(1 << TIF_SECCOMP)


#define _TIF_WORK_MASK \
	(_TIF_NOTIFY_RESUME | _TIF_SIGPENDING | _TIF_NEED_RESCHED)

```

これで、おおよそトラップ時の動作を見た。
トラップの処理自体を詳しく理解するには、`excp_vect_table`、`sys_call_table`、`do_trap_unknown`、`handle_arch_irq`を読む必要がある。
また、`work_pending`や`ptrace`関連のパスは読んでいないが、後に必要になるかもしれない。
