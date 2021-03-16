---
title: "Reading OpenSBI part1"
date: 2021-03-16T09:14:55+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---
## Reading OpenSBI
RISC-VのSBI実装の一つである、OpenSBIを読んでいく。
今開発しているシステムでは、OpenSBIからLinux Kernelをロードし、制御を移しているため、理解しておきたい。
<!--more-->

## OpenSBIとは
Open Source Supervisor Binary Interfaceの略である。\
- [OpenSBI](https://github.com/riscv/opensbi)
### SBIとは
Supervisor Execution Environment(SEE)とSupervisorのインターフェースである。
つまり、Supervisorがより高位のモードの機能を使うためのインタフェースとなっている。
Supervisorモードで、ecallを行うことでM-Modeへ遷移し、Supervisor callの処理を行う。
Unix-like OSでは、OSはS-Modeで動作し、OpenSBIはM-Modeで動作しファームウェアのような(UEFIのような？)立ち位置で存在する。\
OpenSBI自体はstatic libraryである。
RISC-V Foundationに管理されており、SBI自体は以下で定義されている。
- [SBI documentation](https://github.com/riscv/riscv-sbi-doc)

## Firmware types
- [OpenSBI FW](https://github.com/riscv/opensbi/blob/master/docs/firmware/fw.md)

ファームウェアのタイプとしては大きく分けて２つある。
OpenSBIから次段のブートローダーもしくはクライアントプログラム(OSやアプリケーション)を実行する際の違いがある。
違いは雑に言えば、引数の渡しかたと初期化フローの違いくらいである。
また、プラットフォームが違っても初期化のフローは基本的に同じである。

- [FW\_JUMP](https://github.com/riscv/opensbi/blob/master/docs/firmware/fw_jump.md)
	OpenSBIの次に実行されれるプログラムは、固定番地にロードされていると想定する。
	OpenSBIの初期化終了後には、固定番地へジャンプする。
	OpenSBIに直接リンクをしないため、ファームウェアの実行形式のサイズを小さくできる。
	OpenSBIより前段のブートローダーがOpenSBIとクライアントプログラムをロードすることができる場合は使える。
	例えば、前段のブートローダーが、SDカードからOpenSBIおよび次段のクライアントプログラムを固定番地へロードできる場合。
- [FW\_PAYLOAD](https://github.com/riscv/opensbi/blob/master/docs/firmware/fw_payload.md)
	次段のクライアントプログラムのバイナリをOpenSBI自体にリンクする。
	初期ブートローダーがアプリケーションをロードできない場合などに便利である。
	直接リンクするため、実行形式のサイズは大きくなる。
	PAYLOADとしてクライアントプログラムを指定する。
- [FW\_DYNAMIC](https://github.com/riscv/opensbi/blob/master/docs/firmware/fw_dynamic.md)
	次段のクライアントプログラムの情報を前段ブートローダーから受け取る。
	前段がOpenSBIと後段のクライアントプログラムをロードできる場合便利である。

## パラメータ
- FW\_TEXT\_ADDR
	OpenSBIの実行アドレス。
- FW\FDT\_PATH
	FDTへのパス。OpenSBIの.rodataセクションにバイナリを組込み、次段のクライアントプログラムへ渡す。
- FW\FDT\_PADDING
	FDTをゼロ埋めする。

## OpenSBI static library
2種類のライブラリが存在する。
今回は、`libplatsbi.a`を使用する。そこで、OpenSBIをRV32XSoCに移植する。
- libsbi.a

	SBIを提供するプラットフォーム非依存のライブラリである。プラットフォーム依存の処理フックはこのライブラリにリンクされるアプリケーションにより、
提供する必要がある。`struct sbi_platform`のインスタンスにより、フラットフォーム依存処理のフックを設定する。
- libplatsbi.a

	OpenSBIにより、サポートされるプラットフォームに依存のライブラリ。
	- libsbiutils.a

		OpenSBIによりサポートされるプラットフォームの共通処理のライブラリ。

詳しくはこちらを見るとよい。
- [library usage](https://github.com/riscv/opensbi/blob/master/docs/library_usage.md)

## OpenSBIの移植
### requirementsの確認
OpenSBIに必要なRISC-Vの機能は以下にまとめられている。
- [platform requirements](https://github.com/riscv/opensbi/blob/master/docs/platform_requirements.md)
### RV32XSoC
- rv32ima
- one hart/one core(hartid is 0)
- support S-mode
- support MTVEC direct mode

最低限の機能を満たしているため、OpenSBIの移植が可能である。
また、今回のSoCでは、UARTはオリジナルの実装である。
タイマー、ソフトウエア割込み源としてCLINT、外部割込みコントローラーとしてPLIC(M-mode, S-mode)を採用する。
なお、ソフトウエア割込みはプロセッサ間の割込み(inter-processor interrupt)に用いられる。

### 移植
OpenSBIの移植方法は以下にまとめられている。
- [platform guide](https://github.com/riscv/opensbi/blob/master/docs/platform_guide.md)

基本的に、テンプレートのままでOKである(platform/template)。
- platform/template/config.mk
- platform/template/object.mk
- platform/template/platform.c

config.mkにはOpenSBIのビルドオプションを記述する。\
object.mkはそのまま使う。\
platform.cにプラットフォーム依存のコードを書く。\


UART関連および割込みコントローラー系ははSoC用に書く必要がある。\
雑に言えば、`struct sbi_platform_operations`と`struct sbi_platform`の2つの構造体を提供すればよい。
非依存のコードはこの構造体のインスタンスを通して、処理を実行したり、リソースへアクセスを行う。
定義は`include/sbi/sbi_platform.h`にある。
```c
const struct sbi_platform_operations platform_ops = {
	.early_init		= rv32xsoc_early_init,
	.final_init		= rv32xsoc_final_init,
	.console_putc		= rv32xsoc_console_putc,
	.console_getc		= rv32xsoc_console_getc,
	.console_init		= rv32xsoc_console_init,
	.irqchip_init		= rv32xsoc_irqchip_init,
	.ipi_send		= rv32xsoc_ipi_send,
	.ipi_clear		= rv32xsoc_ipi_clear,
	.ipi_init		= rv32xsoc_ipi_init,
	.timer_value		= rv32xsoc_timer_value,
	.timer_event_stop	= rv32xsoc_timer_event_stop,
	.timer_event_start	= rv32xsoc_timer_event_start,
	.timer_init		= rv32xsoc_timer_init,
	.system_reset_check	= rv32xsoc_system_reset_check,
	.system_reset		= rv32xsoc_system_reset
};
const struct sbi_platform platform = {
	.opensbi_version	= OPENSBI_VERSION,
	.platform_version	= SBI_PLATFORM_VERSION(0x0, 0x00),
	.name			= "rv32xsoc-name",
	.features		= SBI_PLATFORM_DEFAULT_FEATURES,
	.hart_count		= 1,
	.hart_stack_size	= SBI_PLATFORM_DEFAULT_HART_STACK_SIZE,
	.platform_ops_addr	= (unsigned long)&platform_ops
};
```
プラットフォーム依存で書く必要のありそうなコードは
- `console_putc`
- `console_getc`
- `console_init`
- `irqchip_init`
- `ipi_init`

である。
タイマー関連、ソフトウエア割込み関連はジェネリックなものを使用する。
Sifiveの仕様に沿ったCLINTを用いているため、問題はないはず。
また、システムリセットはサポートしない。

コンソール系の関数は、単にブロッキングで入出力をすれば良い。
```c
static int rv32xsoc_console_init(void) {
	return rv32xsoc_uart_init();
}
static void rv32xsoc_console_putc(char ch) {
	rv32xsoc_uart_putchar(ch);
}
static int rv32xsoc_console_getc(void) {
	return rv32xsoc_uart_getchar();
}
```
`rv32x_soc_uart`系の関数たちは`opensbi/lib/utils/serial/rv32xsoc_uart.c`に記載する。
ヘッダーファイルは`include/sbi_utils/serial/rv32xsoc_uart.h`に作成する。
```c
#include <sbi_utils/serial/rv32xsoc_uart.h>

int rv32xsoc_uart_init(void) {
	return 0;
}
int rv32xsoc_uart_putchar(int ch) {

	while(RV32XSOC_UART_TX_GET_STAT_FULL())
		asm volatile("nop");

	*RV32XSOC_UART_TX_BUF = ch;
	RV32XSOC_UART_TX_SET_EN(1);

	return (unsigned int) ch;
}

int rv32xsoc_uart_getchar(void) {
	int ch = -1;

	RV32XSOC_UART_RX_SET_EN(1);

	/* Blocking */
	while(RV32XSOC_UART_RX_GET_STAT_EMPTY()) {
		asm volatile("nop");
	}
	ch = *RV32XSOC_UART_RX_BUF;
	return ch;
}
void __attribute__((weak)) rv32xsoc_uart_rx_interrupt_handler(void) {
}
void __attribute__((weak)) rv32xsoc_uart_tx_interrupt_handler(void) {
}
```

```h
#ifndef RV32XSOC_UART_H
#define RV32XSOC_UART_TX_BUF ((volatile unsigned int *) 0x40000000)
#define RV32XSOC_UART_TX_STAT ((volatile unsigned int *) 0x40000004)

#define RV32XSOC_UART_RX_BUF ((volatile unsigned int *) 0x40000010)
#define RV32XSOC_UART_RX_STAT ((volatile unsigned int *) 0x40000014)

#define RV32XSOC_UART_TX_GET_STAT_FULL() (((*(RV32XSOC_UART_TX_STAT)) & 0x00000008) >> 3)
#define RV32XSOC_UART_TX_GET_STAT_EMPTY() (((*(RV32XSOC_UART_TX_STAT)) & 0x00000004) >> 2)
#define RV32XSOC_UART_TX_GET_STAT_BUSY() (((*(RV32XSOC_UART_TX_STAT)) & 0x00000002) >> 1)
#define RV32XSOC_UART_TX_GET_STAT_EN() (((*(RV32XSOC_UART_TX_STAT)) & 0x00000001))
#define RV32XSOC_UART_TX_SET_EN(en) (*(RV32XSOC_UART_TX_STAT) = *(RV32XSOC_UART_TX_STAT) | ((en) & 0x1));

#define RV32XSOC_UART_RX_GET_STAT_FULL() (((*(RV32XSOC_UART_RX_STAT)) & 0x00000008) >> 3)
#define RV32XSOC_UART_RX_GET_STAT_EMPTY() (((*(RV32XSOC_UART_RX_STAT)) & 0x00000004) >> 2)
#define RV32XSOC_UART_RX_GET_STAT_BUSY() (((*(RV32XSOC_UART_RX_STAT)) & 0x00000002) >> 1)
#define RV32XSOC_UART_RX_GET_STAT_EN() (((*(RV32XSOC_UART_RX_STAT)) & 0x00000001))
#define RV32XSOC_UART_RX_SET_EN(en) (*(RV32XSOC_UART_RX_STAT) = *(RV32XSOC_UART_RX_STAT) | ((en) & 0x1));

typedef union {
	struct {
		unsigned int en : 1;	
		unsigned int busy : 1;	
		unsigned int empty : 1;
		unsigned int full : 1;
		unsigned int unused: 28;
	} stat;
	unsigned int val;
} rv32xsoc_uart_tx_stat_t;

typedef rv32xsoc_uart_tx_stat_t rv32xsoc_uart_rx_stat_t;

int rv32xsoc_uart_init(void);
int rv32xsoc_uart_putchar(int ch);
int rv32xsoc_uart_getchar(void);

extern __attribute__((weak)) void rv32xsoc_uart_rx_interrupt_handler(void);
extern __attribute__((weak)) void rv32xsoc_uart_tx_interrupt_handler(void);
#endif
```

割込み系の関数を用意する。
`irq`系がPLIC、`ipi`系がCLINTである。
割込みソースの数やhartの数などはそれぞれ`struct plic_data`および`struct clint_data`にて記述する。
```c
#define RV32XSOC_PLIC_ADDR		0xc000000
#define RV32XSOC_PLIC_NUM_SOURCES	31
#define RV32XSOC_HART_COUNT		1
#define RV32XSOC_CLINT_ADDR		0x2000000
static struct plic_data plic = {
	.addr = RV32XSOC_PLIC_ADDR,
	.num_src = RV32XSOC_PLIC_NUM_SOURCES,
};
static struct clint_data clint = {
	.addr = RV32XSOC_CLINT_ADDR,
	.first_hartid = 0,
	.hart_count = RV32XSOC_HART_COUNT,
	.has_64bit_mmio = FALSE,
};
```
初期化やレジスタ参照はこの構造体をもとにジェネリックな処理を行う。

これで移植は完了である。

## ビルドする
FW\_PAYLOAD、FW\_DYNAMICとしてビルドする。
PAYLOADはビルド時に指定する。未指定にして、デフォルトのサンプルプログラムを使用する。
ビルド時のパラメータは`opensbi/platform/rv32xsoc/config.mk`にある。
以下に抜粋する。
```
platform-cppflags-y =
platform-cflags-y =
platform-asflags-y =
platform-ldflags-y =
PLATFORM_RISCV_XLEN = 32
PLATFORM_RISCV_ABI = ilp32
PLATFORM_RISCV_ISA = rv32ima
PLATFORM_RISCV_CODE_MODEL = medany
FW_TEXT_START=0x80000000
FW_DYNAMIC=y
FW_JUMP=n
FW_PAYLOAD=y
FW_PAYLOAD_OFFSET=0x400000
```
FW\_PAYLOAD\_OFFSETでPAYLOADのロードアドレスをFW\_TEXT\_STARTからのオフセットとして指定する。今回は`0x80000000+0x400000`番地にPAYLOADがロードされることとなる。
ちなみに、OpenSBIは(ファームウェアなので)ベアメタル環境を前提としているので、ツールチェーンとしては、ベアメタル用の`riscv32-unknown-elf-`が必要となる。
ツールのプリフィックスはビルド状況や環境によって違うが、`riscv32-unknown-linux-gnu-`でないことに注意。
```bash
make CROSS_COMPILE=riscv32-unknown-elf- PLATFORM=rv32xsoc
```
## 実行
```bash
clear;./rv32x_simulation fw_payload.elf --no-sim-exit --no-log 2>/dev/null
```

{{<figure src="./image00.png" >}}

正しく動作した。
次回はソースコードを読んでいく。
