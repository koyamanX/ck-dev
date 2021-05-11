---
title: "My RISC-V debug feature part3"
date: 2021-04-20T22:55:05+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "JTAG", "FPGA"]
draft: false
---

OpenOCD RISC-VにVJTAGのサポートを行う。
OpenOCDにて、VJTAG on DE10-Liteにアクセスしてみる。
<!--more-->

### OpenOCDのVJTAGサポート状況
or1kはVJTAGに対応している。
この実装を理解して、OpenOCD RISC-Vへ移植をする。

### OpenOCD VJTAG support (or1kの実装を読解する)
まず、or1kでどのようにVJTAGを使っているのか確かめる。
そのために、cfgファイルを読む。

`tcl/target/or1k.cfg`
```tcl
set  _ENDIAN big

if { [info exists CHIPNAME] } {
   set _CHIPNAME $CHIPNAME
} else {
   set _CHIPNAME or1k
}

if { [info exists TAP_TYPE] } {
   set _TAP_TYPE $TAP_TYPE
} else {
   puts "You need to select a tap type"
   shutdown
}

# Configure the target
if { [string compare $_TAP_TYPE "VJTAG"] == 0 } {
	if { [info exists FPGATAPID] } {
	   set _FPGATAPID $FPGATAPID
	} else {
	   puts "You need to set your FPGA JTAG ID"
		shutdown
	}

	jtag newtap $_CHIPNAME cpu -irlen 10 -expected-id $_FPGATAPID

	set _TARGETNAME $_CHIPNAME.cpu
	target create $_TARGETNAME or1k -endian $_ENDIAN -chain-position $_TARGETNAME

	# Select the TAP core we are using
	tap_select vjtag
}
```

`tap_select vjtag`にてVJTAGをTAPとして選択するようである。
`tap_select`コマンドはor1k向けに定義されたものであるため、`target`コマンドでor1kを選択してから
でないと使用することができない。

`tap_select`命令をまず、RISC-V向けに移植したい。

### tap\_select命令の移植

`src/target/openrisc/or1k.c`
```c
static const struct command_registration or1k_hw_ip_command_handlers[] = {
	{
		.name = "tap_select",
		.handler = or1k_tap_select_command_handler,
		.mode = COMMAND_ANY,
		.usage = "name",
		.help = "Select the TAP core to use",
	},

```
`src/target/openrisc/or1k.c`
```c
COMMAND_HANDLER(or1k_tap_select_command_handler)
{
	struct target *target = get_current_target(CMD_CTX);
	struct or1k_common *or1k = target_to_or1k(target);
	struct or1k_jtag *jtag = &or1k->jtag;
	struct or1k_tap_ip *or1k_tap;

	if (CMD_ARGC != 1)
		return ERROR_COMMAND_SYNTAX_ERROR;

	list_for_each_entry(or1k_tap, &tap_list, list) {
		if (or1k_tap->name) {
			if (!strcmp(CMD_ARGV[0], or1k_tap->name)) {
				jtag->tap_ip = or1k_tap;
				LOG_INFO("%s tap selected", or1k_tap->name);
				return ERROR_OK;
			}
		}
	}

	LOG_ERROR("%s unknown, no tap selected", CMD_ARGV[0]);
	return ERROR_COMMAND_SYNTAX_ERROR;
}
```
`tap_select`コマンドは、`or1k_tap_select_command_handler`が対処する。
`tap_select`コマンドの第一引数にあるインターフェースを使用する。
VJTAGを選択した場合を見てみる。

`src/target/openrisc/or1k_tap_vjtag.c`で実装されている。
`or1k_tap_vjtag_init`が初期化用の処理である。

```C
	jtag_add_tlr();
```
まずは、TAPを初期状態にする。

```C
	uint8_t t[4] = { 0 };
	struct scan_field field;
	struct jtag_tap *tap = jtag_info->tap;

	/* Select VIR */
	buf_set_u32(t, 0, tap->ir_length, ALTERA_CYCLONE_CMD_USER1);
	field.num_bits = tap->ir_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_ir_scan(tap, &field, TAP_IDLE);
```
その後、sld\_hubのIRを`USER1`にする。
この操作で、Select\_IRを行い、Shift\_IRにて、sld\_hubのIRに`USER1`のコード(0xE, 10 bit)を書き込む。

次に、DR Shiftを行う。
`USER1`レジスタのフォーマットは以下のようになっている。
{{<figure src="./image00.png" >}}
ただし、nはCEIL(log2(Number of SLD\_nodes + 1))である。
mは各VIRの長さの最大値である。
`sld_hub`には以下のレジスタがある。
- `SLD HUB IP Configuration Register`
	- sld\_hubに接続されているsld\_nodeの情報や、USER1 DRの寸法がわかる。
- 各`sld_hub`の`SLD_NODE_INFO`レジスタ
	- indexとアドレスのマッピングがわかる
`HUB_INFO`命令を使用し、`HUB IP Configuration Register`を読み出す。
また、各sld\_nodeのアドレスマッピングを保持している、`SLD_NODE_INFO`レジスタがある。
ただし、この時点でsld\_hubに接続されたsld\_nodeの数がわからない。
また、n, mについてもわからない状態である。
そこで、USER1 DRをゼロで埋める。
nがわからないが、64回シフトすれば大体の場合は十分である。
```C
	/* Select the SLD Hub */
	field.num_bits = 64;
	field.out_value = NULL;
	field.in_value = NULL;
	jtag_add_dr_scan(tap, 1, &field, TAP_IDLE);
```
次に、Select\_DRにて、DR(`SLD HUB Configuration register`)を選択する。
```C
	/* Select VDR */
	buf_set_u32(t, 0, tap->ir_length, ALTERA_CYCLONE_CMD_USER0);
	field.num_bits = tap->ir_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_ir_scan(tap, &field, TAP_IDLE);

	int retval = jtag_execute_queue();
	if (retval != ERROR_OK)
		return retval;
```
`jtag_execute_queue`にて、JTAGコマンドを実行する。

次に、`SLD HUB Configuration register`を読み出す。
レジスタのフォーマットは以下のとおりである。
{{<figure src="./image01.png" >}}
8つのnibbleのフィールドからなるレジスタであり、nibble単位での読み出しが必須となる。:w
次のnibbleを読み出す前に、UPDATE\_DRをする必要がある。
そこで、読み出しごとに`jtag_execute_queue`を行う。

```C
	uint8_t nibble;
	uint32_t hub_info = 0;

	for (int i = 0; i < 8; i++) {
		field.num_bits = 4;
		field.out_value = NULL;
		field.in_value = &nibble;
		jtag_add_dr_scan(tap, 1, &field, TAP_IDLE);
		retval = jtag_execute_queue();
		if (retval != ERROR_OK)
			return retval;
		hub_info = ((hub_info >> 4) | ((nibble & 0xf) << 28));
	}

	int nb_nodes = NB_NODES(hub_info);
	int m_width = M_WIDTH(hub_info);

	LOG_DEBUG("SLD HUB Configuration register");
	LOG_DEBUG("------------------------------");
	LOG_DEBUG("m_width         = %d", m_width);
	LOG_DEBUG("manufacturer_id = 0x%02" PRIx32, MANUF(hub_info));
	LOG_DEBUG("nb_of_node      = %d", nb_nodes);
	LOG_DEBUG("version         = %" PRIu32, VER(hub_info));
	LOG_DEBUG("VIR length      = %d", guess_addr_width(nb_nodes) + m_width);
```
これで、sld\_nodeの数、m, nのサイズ、VIRのサイズなどがわかった。

次に、インデックス(VJTAGのインスタンス時にユーザもしくはQuartusが割当)からアドレスへのマッピングを調べる。
`SLD_NODE info Register`に情報が格納されており、sld\_nodeの数分存在する。
このレジスタも同様に8つのnibbleからなる。
つまり、`ノードの数 * (8 * nibble)`回シフトが必要である。

{{<figure src="./image02.png" >}}
```C
	int vjtag_node_address = -1;
	int node_index;
	uint32_t node_info = 0;
	for (node_index = 0; node_index < nb_nodes; node_index++) {

		for (int i = 0; i < 8; i++) {
			field.num_bits = 4;
			field.out_value = NULL;
			field.in_value = &nibble;
			jtag_add_dr_scan(tap, 1, &field, TAP_IDLE);
			retval = jtag_execute_queue();
			if (retval != ERROR_OK)
				return retval;
			node_info = ((node_info >> 4) | ((nibble & 0xf) << 28));
		}

		LOG_DEBUG("Node info register");
		LOG_DEBUG("--------------------");
		LOG_DEBUG("instance_id     = %" PRIu32, ID(node_info));
		LOG_DEBUG("manufacturer_id = 0x%02" PRIx32, MANUF(node_info));
		LOG_DEBUG("node_id         = %" PRIu32 " (%s)", ID(node_info),
						       id_to_string(ID(node_info)));
		LOG_DEBUG("version         = %" PRIu32, VER(node_info));

		if (ID(node_info) == VJTAG_NODE_ID)
			vjtag_node_address = node_index + 1;
	}

	if (vjtag_node_address < 0) {
		LOG_ERROR("No VJTAG TAP instance found !");
```
これでアドレスがわかった。
次に、sld\_nodeのインスタンスのVIR, VDRにアクセスをする。

まず、`USER1`命令を発行する。
これにより、IR chainが選択される。
```C
	/* Select VIR */
	buf_set_u32(t, 0, tap->ir_length, ALTERA_CYCLONE_CMD_USER1);
	field.num_bits = tap->ir_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_ir_scan(tap, &field, TAP_IDLE);
```

次に、sld\_nodeのVIRに命令を送る。
この場合はDEBUG命令を転送している。
これは、or1kのデバックモードの命令の様子。
なお、転送サイズは`アドレスのサイズ+VIRのサイズ`である。
VIRの前に、Addressを付与する。
フォーマットを以下に示す。
{{<figure src="./image03.png" >}}

```C
/* Send the DEBUG command to the VJTAG IR */
	int dr_length = guess_addr_width(nb_nodes) + m_width;
	buf_set_u32(t, 0, dr_length, (vjtag_node_address << m_width) | ALT_VJTAG_CMD_DEBUG);
	field.num_bits = dr_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_dr_scan(tap, 1, &field, TAP_IDLE);
```

最後に、`USER0`命令を発行して、DR chainに切り替える。
これにより、後続のShift\_DEは、`USER1`命令でセットした、Addrを用いてsld\_nodeのインスタンスに転送される。
なお、or1kの場合では、VIRをDEBUGから変えることはないようである。
しかし、VIRを変える場合は、`USER1`を発行して、ADDR+VIRを転送し、`USER0`へ切り替えるという手順が必要となる。
この切り替え部分をRISC-VのOpenOCDへ追加する必要がある。
DRのアクセスは切り替えが正しくできていれば、問題ないはずである。
```C
/* Select the VJTAG DR */
	buf_set_u32(t, 0, tap->ir_length, ALTERA_CYCLONE_CMD_USER0);
	field.num_bits = tap->ir_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_ir_scan(tap, &field, TAP_IDLE);

	return jtag_execute_queue();
}
```
### OpenOCD VJTAG support
詳しい追加は[koyamanX/riscv-openocd](https://github.com/koyamanX/riscv-openocd)にある、
`src/target/riscv/riscv_tap_vjtag.c`を確認してほしい。
初期化用のコードは、`or1k_tap_vjtag_init`をそのまま流用する。
関数名を`riscv_tap_vjtag_init`とする。
また、DEBUGレジスタの指定を、DTMCSレジスタの指定に変更する。
これで初期化は十分である。
一部、変数(nb\_nodesなど)を雑にグローバル変数にした。
また、VIRの書き込み用の関数を別に定義した。

`src/target/riscv/riscv_tap_vjtag.c`
```C
int vjtag_vir_scan(struct jtag_tap *tap, uint32_t vir_val)
{
	uint8_t t[4] = { 0 };
	struct scan_field field;

	/* Select VIR chain */
	buf_set_u32(t, 0, tap->ir_length, ALTERA_CYCLONE_CMD_USER1);
	field.num_bits = tap->ir_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_ir_scan(tap, &field, TAP_IDLE);

	/* Set VIR Value to the VIR of sld_node determined by vjtag_node_address */
	int dr_length = guess_addr_width(nb_nodes) + m_width;
	buf_set_u32(t, 0, dr_length, (vjtag_node_address << m_width) | vir_val);
	field.num_bits = dr_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_dr_scan(tap, 1, &field, TAP_IDLE);

	/* Select the VJTAG DR chain */
	buf_set_u32(t, 0, tap->ir_length, ALTERA_CYCLONE_CMD_USER0);
	field.num_bits = tap->ir_length;
	field.out_value = t;
	field.in_value = NULL;
	jtag_add_ir_scan(tap, &field, TAP_IDLE);

	return jtag_execute_queue();
}
```

`src/target/riscv/riscv.c`を変更し、targetの初期化の際に、ついでに`riscv_tap_vjtag_init`を呼び出すことにした。
また、`use_vjtag`フラグで、VJTAGを使用するかをハードコードした。
本来は、このフラグを`tap_select`のような命令で変更できるようにするべきである。
しかし、`target`固有のコマンドは`target`の初期化が終わっていこうに使えるようになる。
ただし、`target`初期化の処理は、`examine`(JTAGでDebug Moduleへアクセスし、dmactiveを取得する)が成功したら
正しく完了したこととなる。
VJTAGを用いたアクセスでないため、必ず失敗する。
そこで、とりあえず、動けばいいという適当な考えで、ハードコードしてしまった。
正しく動くことが確認できたら、しっかりと対応しテストしてPRを出したいね。

### 通信テスト
VJTAGとOpenOCDでやり取りをする。
適当なDebug Moduleのような回路を書く。
コードは[koyamanX/riscv-openocd](https://github.com/koyamanX/riscv-debug)にある。
なお、DM(Debug Module)やDTM(Debug Transport Module)はJTAGのTCKのクロックドメインで動作していることに注意。
プロセッサと接続するためには、適切にCDC(Clock Domain Crossing)を行う必要がある。
また、DMに関しては、まだスタブである。

`src/dtm.nsl`
```C
#ifndef DTM_H
#define DTM_H

/* This module design to run in TCK in JTAG clock domain */
/* m_clock must be connected to TCK of JTAG */

#define IDCODE		5'b00001
#define DTMCS		5'b10000
#define DMI			5'b10001
#define BYPASS		5'b11111

#define DTMCS_VERSION	4'b0001
#define DTMCS_ABITS		6'b100000

#define DMI_NOP			2'b00
#define DMI_READ		2'b01
#define DMI_WRITE		2'b10
#define DMI_RESERVED	2'b11

#define DMI_STAT_SUCCESS	2'b00
#define DMI_STAT_RESERVED	2'b01
#define DMI_STAT_FAILURE	2'b10
#define DMI_STAT_INPROGRESS	2'b11

struct dtmcs_t {
	zero1[14];
	dmihardreset[1];
	dmireset[1];
	zero0[1];
	idle[3];
	dmistat[2];
	abits[6];
	version[4];
};

struct dmi_t {
	addr[32];
	data[32];
	op[2];
};

declare dtm interface {
	input m_clock;
	input p_reset;

	input ir_in[5];
	output ir_out[5];
	input tdi;
	output tdo;
	func_in virtual_state_cdr;
	func_in	virtual_state_sdr;
	func_in	virtual_state_e1dr;
	func_in	virtual_state_pdr;
	func_in	virtual_state_e2dr;
	func_in	virtual_state_udr;
	func_in	virtual_state_cir;
	func_in	virtual_state_uir;

	/* DMI */
	output addr[32];
	input rdata[32];
	output wdata[32];
	func_out read(addr);
	func_out write(addr, wdata);
	func_in ready();

#ifdef DEBUG
	output debug_out[32];
#endif
}

#endif
```

`src/dtm.nsl`

```C
#include "dtm.h"

module dtm {
	reg ir[5] = IDCODE;	/* IDECODE (defined by spec) */
	reg idcode[32] = 0x10e31913;	/* same as SiFive's */
	reg bypass = 0;
	
	/* DTMCS register */
	dtmcs_t reg dtmcs = {14'b00000000000000, 1'b0, 1'b0, 1'b0, 3'b000, 2'b00, DTMCS_ABITS, DTMCS_VERSION};
	/* DTMCS internal register for read via dtmcs(Capture-DR) */
	reg dtmcs_dmistat[2] = 0;
	reg dtmcs_idle[3] = 0;
	reg dtmcs_dmireset = 0;
	reg dtmcs_dmihardreset = 0;
	/* DMI register */
	dmi_t reg dmi = 0;
	/* DMI internal register for read via dmi(Capture-DR) */
	reg dmi_op_stat[2] = 0;
	reg dmi_addr[32] = 0;
	reg dmi_data[32] = 0;

	func virtual_state_uir {
		ir := ir_in;
	}
	func virtual_state_cir {
		ir_out = IDCODE;
	}
	func virtual_state_udr {
		/* At this point, we can issue abstarct command */
		any {
			ir == DTMCS: {
				/* TODO: Implement hardreset */
				if(dtmcs.dmihardreset) {
					dtmcs_dmireset := 0;
				} else if(dtmcs.dmireset) {
					dtmcs_dmireset := 0;
					dmi_op_stat := 0;
				}
			}
			ir == DMI: {
				any {
					dmi.op == DMI_NOP: {dmi_op_stat := DMI_STAT_SUCCESS;}
					dmi.op == DMI_READ: {
						dmi_addr := dmi.addr;
						read(dmi.addr);
					}
					dmi.op == DMI_WRITE: {
						dmi_addr := dmi.addr;
						dmi_data := dmi.data;
						write(dmi.addr, dmi.data);
					}
					dmi.op == DMI_RESERVED: {dmi_op_stat := DMI_STAT_FAILURE;}
				}
			}
		}
	}
	func ready {
		dmi_data := rdata;
		dmi_op_stat := DMI_STAT_SUCCESS;
	}
	func virtual_state_cdr {
		any {
			ir == DTMCS: dtmcs := {14'(1'b0), dtmcs_dmihardreset, dtmcs_dmireset, 1'b0, 
									dtmcs_idle, dtmcs_dmistat, DTMCS_ABITS, DTMCS_VERSION};
			ir == DMI: dmi := {dmi_addr, dmi_data, dmi_op_stat};
			ir == IDCODE: idcode := 0x10e31913;
			ir == BYPASS: bypass := 1'b0;
			else: bypass := 1'b0;
		}
	}
	func virtual_state_sdr {
		any {
			ir == DTMCS: dtmcs := {tdi, dtmcs[31:1]};
			ir == DMI: dmi := {tdi, dmi[65:1]};
			ir == IDCODE: idcode := {tdi, idcode[31:1]};
			ir == BYPASS: bypass := tdi;
			else: bypass := tdi;
		}
	}
	any {
		ir == DTMCS: tdo = dtmcs[0];
		ir == DMI: tdo = dmi[0];
		ir == BYPASS: tdo = bypass;
		ir == IDCODE: tdo = idcode[0];
		else: tdo = bypass;
	}

#ifdef DEBUG
	debug_out = dtmcs;
#endif
}
```


`src/dm.h`
```C
#ifndef DM_H
#define DM_H
/* Runs in same clock domain as DTM and output to hart must be 
	synchronized to hart's clock domain */
declare dm interface {
	input p_reset;
	input m_clock;
	/* DMI */
	input addr[32];
	input wdata[32];
	output rdata[32];
	func_in read(addr);
	func_in write(addr, wdata);
	func_out ready();
#ifdef DEBUG
	output debug_out[32];
#endif
}
#endif
```

`src/dm.nsl`
```C
#include "dm.h"
module dm {
	reg dmcontrol[32] = 0;
	func write {
		any {
			addr == 0x10: dmcontrol := wdata;
		}
		ready();
	}
	func read {
		any {
			addr == 0x10: rdata = dmcontrol;
		}
		ready();
	}

#ifdef DEBUG
	debug_out = dmcontrol;
#endif
}
```

```C
#include "vjtag.h"
#include "dtm.h"
#include "dm.h"

declare DE10 {
	input SW[10];
	output LEDR[10];
}
module DE10 {
	vjtag vjtag0;
	dtm riscv_dtm;
	dm riscv_dm;

	riscv_dtm.m_clock = vjtag0.tck;
	riscv_dtm.p_reset = p_reset;
	riscv_dtm.ir_in = vjtag0.ir_in;
	vjtag0.ir_out = riscv_dtm.ir_out;
	riscv_dtm.tdi = vjtag0.tdi;
	vjtag0.tdo = riscv_dtm.tdo;

	riscv_dm.m_clock = vjtag0.tck;
	riscv_dm.p_reset = p_reset;

	func vjtag0.virtual_state_cdr {
		riscv_dtm.virtual_state_cdr();
	}
	func vjtag0.virtual_state_sdr {
		riscv_dtm.virtual_state_sdr();
	}
	func vjtag0.virtual_state_udr {
		riscv_dtm.virtual_state_udr();
	}
	func vjtag0.virtual_state_cir {
		riscv_dtm.virtual_state_cir();
	}
	func vjtag0.virtual_state_uir {
		riscv_dtm.virtual_state_uir();
	}
	func riscv_dtm.read {
		riscv_dm.read(riscv_dtm.addr);
	}
	func riscv_dtm.write {
		riscv_dm.write(riscv_dtm.addr, riscv_dtm.wdata);
	}
	func riscv_dm.ready {
		riscv_dtm.rdata = riscv_dm.rdata;
		riscv_dtm.ready();
	}

#ifdef DEBUG
	any {
		SW == 0: LEDR = riscv_dtm.debug_out;
		SW == 1: LEDR = riscv_dm.debug_out;
		else:	LEDR = 0xffffffff;
	}
#endif
}

```

### 動作確認
[koyamanX/riscv-openocd](https://github.com/koyamanX/riscv-debug)

#### ツールのビルド
```bash
sudo apt install libftdi1-dev
git clone --recursive https://github.com/koyamanX/riscv-debug #riscv-vjtag branch(by default)
cd riscv-debug/riscv-openocd
git submodule update --init --recursive
./bootstrap
mkdir build
cd build
../configure --prefix=/opt/riscv-openocd
make -j $(nproc)
sudo make install
```

`rv32xsoc.cfg`
```
set _ENDIAN little
set _CHIPNAME riscv
set _FPGATAPID 0x031050dd
set _TARGETNAME $_CHIPNAME.cpu

adapter driver usb_blaster
usb_blaster_lowlevel_driver ftdi

jtag newtap $_CHIPNAME cpu -irlen 10 -expected-id $_FPGATAPID
target create $_TARGETNAME riscv -endian $_ENDIAN -chain-position $_TARGETNAME

#use_vjtag
```

#### FPGA(DE10-Lite)へ実装
```bash
cd riscv-debug/fpga
make all
make download
```

#### テスト
``` bash
sudo /opt/riscv-openocd/bin/openocd -f ../tcl/target/rv32xsoc.cfg
```
#### 実行結果
dmactiveに成功している。(dmcontrolの0ビット目)
dmstatusのリードに失敗しているが、実装していないので期待通り。
{{<figure src="./image04.png" >}}

次からは、DMの仕様読みと実装を行う。
