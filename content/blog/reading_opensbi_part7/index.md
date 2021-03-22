---
title: "Reading OpenSBI part7"
date: 2021-03-19T05:17:26+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---

今回から、C言語によるOpenSBIの実装を見ていく。
`sbi_trap_handler`と`sbi_init`を読んでいく。
<!--more-->

### sbi\_trap\_handler
- `mscratch`は`struct sbi_scratch`を指している。
- `regs`は`_trap_handler`(`firmware/fw_base.S`)にてセーブした(`struct sbi_trap_regs`)のポインタである(`sp` in `_trap_handler`)。

`lib/sbi/sbi_trap.c`
```c
...
void sbi_trap_handler(struct sbi_trap_regs *regs)
{
	int rc = SBI_ENOTSUPP;
	const char *msg = "trap handler failed";
	ulong mcause = csr_read(CSR_MCAUSE);
	ulong mtval = csr_read(CSR_MTVAL), mtval2 = 0, mtinst = 0;
	struct sbi_trap_info trap;

	if (misa_extension('H')) {
		mtval2 = csr_read(CSR_MTVAL2);
		mtinst = csr_read(CSR_MTINST);
	}
...
```
トラップの情報はスタック上の(`struct sbi_trap_info`)に保存する。
この情報はtrapをリダイレクトする際に用いる。
H-extensionが有効の場合は、`mtval2`、`mtinst`を取得する。(それ以外の場合は`0`)
`include/sbi/sbi_trap.h`
```c
...
/** Representation of trap details */
struct sbi_trap_info {
	/** epc Trap program counter */
	unsigned long epc;
	/** cause Trap exception cause */
	unsigned long cause;
	/** tval Trap value */
	unsigned long tval;
	/** tval2 Trap value 2 */
	unsigned long tval2;
	/** tinst Trap instruction */
	unsigned long tinst;
};
...
```
### Interruptのtrap
`mcause`の最上位ビットが1のときはInterruptによるトラップである。
ここでは、M-modeのタイマー割込みおよびM-modeのソフトウエア割込みの対処を行う。
それぞれ、`sbi_timer_process`と`sbi_ipi_process`が受け持つ。
M-modeの外部割込みは対処しない。(OpenSBIはM-modeで動作する、OpenSBIはM-modeの外部割込みを使用ない)
```c
...
	if (mcause & (1UL << (__riscv_xlen - 1))) {
		mcause &= ~(1UL << (__riscv_xlen - 1));
		switch (mcause) {
		case IRQ_M_TIMER:
			sbi_timer_process();
			break;
		case IRQ_M_SOFT:
			sbi_ipi_process();
			break;
		default:
			msg = "unhandled external interrupt";
			goto trap_error;
		};
		return;
	}
...
```

#### sbi\_timer\_process
`lib/sbi/sbi_timer.c`
```c
void sbi_timer_process(void)
{
	csr_clear(CSR_MIE, MIP_MTIP);
	csr_set(CSR_MIP, MIP_STIP);
}
```
`sbi_timer_process`では、M-modeのタイマー割込みをS-modeのタイマー割込みとして発生させる。
単に、M-modeのタイマー割込みを停止して(mie.mtip = 0)、S-modeのタイマー割込みを発生させる(mie.stip = 1)。
現在はM-modeのトラップなので、低位のモードのトラップは遅延する。
後続の`sbi_trap_redirect`にて実際にトラップのリダイレクトを行う。

#### sbi\_ipi\_process

`lib/sbi/sbi_ipi.c`
```c
void sbi_ipi_process(void)
{
	unsigned long ipi_type;
	unsigned int ipi_event;
	const struct sbi_ipi_event_ops *ipi_ops;
	struct sbi_scratch *scratch = sbi_scratch_thishart_ptr();
	const struct sbi_platform *plat = sbi_platform_ptr(scratch);
	struct sbi_ipi_data *ipi_data =
			sbi_scratch_offset_ptr(scratch, ipi_data_off);

	u32 hartid = current_hartid();
	sbi_platform_ipi_clear(plat, hartid);

	ipi_type = atomic_raw_xchg_ulong(&ipi_data->ipi_type, 0);
	ipi_event = 0;
	while (ipi_type) {
		if (!(ipi_type & 1UL))
			goto skip;

		ipi_ops = ipi_ops_array[ipi_event];
		if (ipi_ops && ipi_ops->process)
			ipi_ops->process(scratch);

skip:
		ipi_type = ipi_type >> 1;
		ipi_event++;
	};
}
```
scratch spaceから`struct sbi_ipi_data`を取り出している。
`ipi_data`は`sbi_ipi_init`にてscratch spaceに作成している様子。
`ipi_data`は`sbi_ipi_send`にてリモート(ipi先)のhartのscratch spaceにセットする。

```c
struct sbi_ipi_data {
	unsigned long ipi_type;
};
```

```c
int sbi_ipi_init(struct sbi_scratch *scratch, bool cold_boot)
{
	int ret;
	struct sbi_ipi_data *ipi_data;

	if (cold_boot) {
		ipi_data_off = sbi_scratch_alloc_offset(sizeof(*ipi_data),
							"IPI_DATA");
```

取得した`ipi_data->ipi_type`を用いて`struct sbi_ipi_event_ops ipi_ops_array`を索引し`ipi_ops`を取得、(`ipi_ops->process()`)実行する。
なお、`ipi_type`のビットポジションが0からの添字に対応している。


`include/sbi/sbi_ipi.h`
```c
/** IPI event operations or callbacks */
struct sbi_ipi_event_ops {
	/** Name of the IPI event operations */
	char name[32];

	/**
	 * Update callback to save/enqueue data for remote HART
	 * Note: This is an optional callback and it is called just before
	 * triggering IPI to remote HART.
	 */
	int (* update)(struct sbi_scratch *scratch,
			struct sbi_scratch *remote_scratch,
			u32 remote_hartid, void *data);

	/**
	 * Sync callback to wait for remote HART
	 * Note: This is an optional callback and it is called just after
	 * triggering IPI to remote HART.
	 */
	void (* sync)(struct sbi_scratch *scratch);

	/**
	 * Process callback to handle IPI event
	 * Note: This is a mandatory callback and it is called on the
	 * remote HART after IPI is triggered.
	 */
	void (* process)(struct sbi_scratch *scratch);
};
```

`struct ipi_event_ops ipi_ops_array`は`sbi_ipi_event_create`を用いて`sbi_ipi_init`内で初期化される。

### Exception trap
`sbi_trap_handler`の続き
```c
	switch (mcause) {
	case CAUSE_ILLEGAL_INSTRUCTION:
		rc  = sbi_illegal_insn_handler(mtval, regs);
		msg = "illegal instruction handler failed";
		break;
	case CAUSE_MISALIGNED_LOAD:
		rc = sbi_misaligned_load_handler(mtval, mtval2, mtinst, regs);
		msg = "misaligned load handler failed";
		break;
	case CAUSE_MISALIGNED_STORE:
		rc  = sbi_misaligned_store_handler(mtval, mtval2, mtinst, regs);
		msg = "misaligned store handler failed";
		break;
	case CAUSE_SUPERVISOR_ECALL:
	case CAUSE_MACHINE_ECALL:
		rc  = sbi_ecall_handler(regs);
		msg = "ecall handler failed";
		break;
	default:
		/* If the trap came from S or U mode, redirect it there */
		trap.epc = regs->mepc;
		trap.cause = mcause;
		trap.tval = mtval;
		trap.tval2 = mtval2;
		trap.tinst = mtinst;
		rc = sbi_trap_redirect(regs, &trap);
		break;
	};

trap_error:
	if (rc)
		sbi_trap_error(msg, rc, mcause, mtval, mtval2, mtinst, regs);
}
```
次に、mcauseの最上位ビットが0のパターン(例外によるトラップ)。
Illegal Instruction Exception, Load Address Misaligned exception, Store AMO Address Misalgined Exception, ECALL From S/M modeは
OpenSBIによって、トラップする。その後、各例外に対応したハンドラー(`sbi_illegal_insn_handler`, `sbi_misaligned_load_handler`, `sbi_msialgined_store_handler`, `sbi_ecall_handler`)へ制御を移す。

それ以外の例外(S-modeもしくはU-mode由来)のものは、それぞれのモードへトラップのコンテキストとともにリダイレクト(`sbi_trap_redirect`)する。
なお、delegateされているトラップについては、各モードにてハンドルされ、OpenSBIは感知しない。

`sbi_trap_redirect`について見ていく。

`lib/sbi/sbi_trap.c`
```c
...
	} else {
		/* Update S-mode exception info */
		csr_write(CSR_STVAL, trap->tval);
		csr_write(CSR_SEPC, trap->epc);
		csr_write(CSR_SCAUSE, trap->cause);

		/* Set MEPC to S-mode exception vector base */
		regs->mepc = csr_read(CSR_STVEC);

		/* Set MPP to S-mode */
		regs->mstatus &= ~MSTATUS_MPP;
		regs->mstatus |= (PRV_S << MSTATUS_MPP_SHIFT);

		/* Set SPP for S-mode */
		regs->mstatus &= ~MSTATUS_SPP;
		if (prev_mode == PRV_S)
			regs->mstatus |= (1UL << MSTATUS_SPP_SHIFT);

		/* Set SPIE for S-mode */
		regs->mstatus &= ~MSTATUS_SPIE;
		if (regs->mstatus & MSTATUS_SIE)
			regs->mstatus |= (1UL << MSTATUS_SPIE_SHIFT);

		/* Clear SIE for S-mode */
		regs->mstatus &= ~MSTATUS_SIE;
	}
```
M-modeのトラップコンテキストをS-modeへ移している。
M-modeのトラップハンドラーを`mret`したあとに、S-modeのトラップハンドラへ制御が移る。

## sbi\_init

`lib/sbi/sbi_init.c`
```c
void __noreturn sbi_init(struct sbi_scratch *scratch)
{
	bool next_mode_supported	= FALSE;
	bool coldboot			= FALSE;
	u32 hartid			= current_hartid();
	const struct sbi_platform *plat = sbi_platform_ptr(scratch);

	if ((SBI_HARTMASK_MAX_BITS <= hartid) ||
	    sbi_platform_hart_invalid(plat, hartid))
		sbi_hart_hang();

	switch (scratch->next_mode) {
	case PRV_M:
		next_mode_supported = TRUE;
		break;
	case PRV_S:
		if (misa_extension('S'))
			next_mode_supported = TRUE;
		break;
	case PRV_U:
		if (misa_extension('U'))
			next_mode_supported = TRUE;
		break;
	default:
		sbi_hart_hang();
	}

	/*
	 * Only the HART supporting privilege mode specified in the
	 * scratch->next_mode should be allowed to become the coldboot
	 * HART because the coldboot HART will be directly jumping to
	 * the next booting stage.
	 *
	 * We use a lottery mechanism to select coldboot HART among
	 * HARTs which satisfy above condition.
	 */

	if (next_mode_supported && atomic_xchg(&coldboot_lottery, 1) == 0)
		coldboot = TRUE;

	if (coldboot)
		init_coldboot(scratch, hartid);
	else
		init_warmboot(scratch, hartid);
}
```

`coldboot`を`coldboot_lottery`よりはじめに`atomic_xchg`で0を読み出したhart(特権モード必須)が担当する。
`init_coldboot`は次のレベルのクライアントプログラムにジャンプする。
その他のコアは`init_warmboot`する。

`sbi_init`については、以下のリンク先のサイトがわかりやすい。(Versionが違うことに注意)
- [OpenSBIの内部実装(boot~linux kernelを実行するまで)](https://cstmize.hatenablog.jp/entry/2019/10/21/OpenSBI%E3%81%AE%E5%86%85%E9%83%A8%E5%AE%9F%E8%A3%85%28boot~linux_kernel%E3%82%92%E5%AE%9F%E8%A1%8C%E3%81%99%E3%82%8B%E3%81%BE%E3%81%A7%29)


OpenSBIから制御が移ると、クライアントプログラムからOpenSBIへ移る手段はトラップのみである。
上のリンクにも上がっていたが、どのトラップがトリガーになっているかしっかり調べておく。

## delegate\_traps
`delegate_traps`は`lib/sbi/sbi_hart.c`で定義されている関数で、低位の特権モードへトラップをDelegationを設定する。
`delegate_traps`は`sbi_init`中`sbi_hart_init`で呼ばれる。
Delegateされていないトラップについては、M-modeでOpenSBIにハンドルされることとなる。

`lib/sbi/sbi_hart.c`
```c
static int delegate_traps(struct sbi_scratch *scratch)
{
	const struct sbi_platform *plat = sbi_platform_ptr(scratch);
	unsigned long interrupts, exceptions;

	if (!misa_extension('S'))
		/* No delegation possible as mideleg does not exist */
		return 0;

	/* Send M-mode interrupts and most exceptions to S-mode */
	interrupts = MIP_SSIP | MIP_STIP | MIP_SEIP;
	exceptions = (1U << CAUSE_MISALIGNED_FETCH) | (1U << CAUSE_BREAKPOINT) |
		     (1U << CAUSE_USER_ECALL);
	if (sbi_platform_has_mfaults_delegation(plat))
		exceptions |= (1U << CAUSE_FETCH_PAGE_FAULT) |
			      (1U << CAUSE_LOAD_PAGE_FAULT) |
			      (1U << CAUSE_STORE_PAGE_FAULT);

	/*
	 * If hypervisor extension available then we only handle hypervisor
	 * calls (i.e. ecalls from HS-mode) in M-mode.
	 *
	 * The HS-mode will additionally handle supervisor calls (i.e. ecalls
	 * from VS-mode), Guest page faults and Virtual interrupts.
	 */
	if (misa_extension('H')) {
		exceptions |= (1U << CAUSE_VIRTUAL_SUPERVISOR_ECALL);
		exceptions |= (1U << CAUSE_FETCH_GUEST_PAGE_FAULT);
		exceptions |= (1U << CAUSE_LOAD_GUEST_PAGE_FAULT);
		exceptions |= (1U << CAUSE_VIRTUAL_INST_FAULT);
		exceptions |= (1U << CAUSE_STORE_GUEST_PAGE_FAULT);
	}

	csr_write(CSR_MIDELEG, interrupts);
	csr_write(CSR_MEDELEG, exceptions);

	return 0;
}
```
当たり前だが、S-modeがない場合はdelagetionはできない。
#### 割込みdelegationの設定

- Supervisor Software Interrput
- Supervisor Timer Interrput
- Supervisor External Interrput

#### 例外delegationの設定
- Environment Call From U Mode
- Instruction Address Misaligned
- Environment breakpoint
- Instruction Page Fault	(satpありの場合)
- Load Page Fault			(satpありの場合)	
- Store AMO Page Fault		(satpありの場合)

上にリストしたものに関してはS-mode以下の特権モードにてハンドルされる。

これでなんとなくであるが、OpenSBIを理解した。
今後、OpenSBIとクライアントプログラムのやり取りがわからなくなったら深堀していこうとおもう。
