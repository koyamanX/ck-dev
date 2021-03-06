---
title: "My RISC-V debug feature part2"
date: 2021-04-17T11:21:32+09:00
author: "@koyamanX"
categories: ["RISC-V"]
tags: ["RISC-V", "JTAG", "FPGA"]
draft: false
---

今回はIntel FPGAのVJTAGのIPを試してみる。
FPGAボードはDE10 Liteを使用する。
<!--more-->

### JTAG(Joint Test Action Group)
JTAGについては以下のサイトがわかりやすい。

- [12月1日 JTAGとは何か](http://www.tokudenkairo.co.jp/jtag/adv2018/01.php)
- [12月2日 JTAGの信号線と波形](http://www.tokudenkairo.co.jp/jtag/adv2018/02.php)
- [12月3日 データレジスタと命令レジスタ](http://www.tokudenkairo.co.jp/jtag/adv2018/03.php)
- [12月4日 TAPステートマシン](http://www.tokudenkairo.co.jp/jtag/adv2018/04.php)
- [12月5日 データレジスタの切り替えと、プライベート命令](http://www.tokudenkairo.co.jp/jtag/adv2018/05.php)
- [JTAG – A technical overview and Timing](https://www.cnblogs.com/shangdawei/p/4753682.html)

JTAGを用いると、ICの端子を操作したり、ICの内部と通信ができるようになる。
もともと、JTAGはICの端子を操作するバウンダリースキャンのために作られたようである。
IEEE1149.1は4本の信号線でIC内部と通信するための、プロトコルを定めた規格である。
今回は、バウンダリースキャンは行わない。プロセッサの内部の状態をriscv-debug-specに沿って操作することを目標とする。
なお、riscv-debug-specでは、Debug Transport(情報のやり取り)には規定はないが、JTAGの例が乗っている。
例に乗っ取り、今回は、JTAGを用いる。

JTAGはIRレジスタとDRレジスタがある。
これらはシフトレジスタであり、Shift-IR, Shift-DRにてシフトする。
IRは主にDRの命令(アドレス)を保持する。
IRは最低2ビット必要である。

#### 実装の必須な命令は以下である。
- BYPASS
- SAMPLE
- PRELOAD
- EXTEST

#### オプションな命令
- IDCODE

#### 今回使用する命令
- BYPASS
- IDCODE
なお、バウンダリースキャンはサポートしない。

なお、IRレジスタを読み出すときは、前回の値でなくステータスが帰る。
ステータスは下位２ビットが`01`であること以外は実装定義である。
DRはIRによって選択する。
DRの長さは実装定義で、制御用のソフトウエアはシフト回数を適切に行う必要がある。
なお、仕様で規定されている必須のDRが存在する。
- IDCODE(IR: 任意)
	- JTAGのIDコードを保持する(32 bit)
- BYPASS(IR: すべてのビットが１)
	- 1 bitのレジスタ
	- 実装していない命令などはこちらへ書き込むことが推奨されている

### JTAG信号線
- TCK (Test Clock)
	- クロック信号
- TMS (Test Mode Select)
	- 次のステートを決める
	- 5回以上1を入力するとリセットすることができる
	- TCKの立ち上がりエッジ
- TDI (Test Data-In)
	- データ入力
	- TCKの立ち上がりエッジ
- TDO (Test Data-Out)
	- データ出力
	- TCKの立ち下がりエッジ
### State machine of JTAG TAP
JTAGの状態はTAP(Test Access Port)が管理する。
16のステートを持ち、現在の状態とTMSにより次のステートが決まる。
ステートマシン図を以下に示す。
{{<figure src="./image01.png" >}}

#### Select
Select\_IRの場合は、IRをオペレーション対象とする。
Select\_DRの場合は、DR Chainを選択し、IRにて、各DRを選択する。

#### Shift
Shift\_DR, Shift\_IRにてシフトレジスタをシフトする。
Shift\_DRの場合は、IRの値に応じて、DRが選択され、シフトされる。
TDIはMSBへ、LSBはTDOへ接続される。
Shift\_IRはIRをシフトする。
TDIはMSBへ、ステータス(IRの値ではないことに注意)のLSBはTDOへ接続される。
#### Capture
Capture\_DRでは、IRで選択する、DRにパラレルで値を代入する。
Capture\_IRでは、IRへパラレルで値を代入する。
処理の結果やステート、初期値などをこのステートでDR/IRへ転送する。

#### Update
Updateでは、並列でDRもしくはIRの値を内部ロジックへコピーする。
なにか、処理をする場合はこのステートで実行する。(RISC-V DebugのDTMにおけるDMI Write・Readなど)

重要なステートはUpdate, Capture, Shift、Selectである。
なお、アプリケーションはDRの長さや現在の値、ステートについてしっかり把握して制御する必要がある。
ステートに関して、どのステートからでも、TMSを５回以上1にしておくことで初期状態へ戻ることができる。

### Virtual JTAG
今回はVJTAGのIPを使用する。
VJTAGを用いると、Download Cable(USB-Blaster)を用いて、FPGAとPC間のJTAG通信ができる。
VJTAGとJTAGの違いは、あまりなく、実際はJTAGと同じ制御をしている。
しかし、VJTAGのインスタンスは最大255個作成できるため、これらを識別する機能が必要である。
各インスタンスにはユニークなインデックス(アドレス)が割り当てられ(ユーザが指定可能、指定しない場合はQuartusが合成時に自動割当）
これを用いて、識別する。
FPGAのデザイン内で、通信のルーティングを行う仕組みとして、sld\_hubがぞんざいする。
各VJTAGのインスタンスはsld\_nodeとなり、sld\_hubに接続される。
以下にブロック図を示す。
[ここより](https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/ug/ug_virtualjtag.pdf)引用
{{<figure src="./image00.png" >}}
よって、アプリケーションはsld\_hubを介して、ユーザロジックのVJTAGインスタンス(sld\_node)へアクセスする。
なお、１つしかインスタンスが存在しない場合でも、sld\_hubは生成される。
また、sld\_node, sld\_hubはLEs(Logic Element)を消費する。
JTAG TAPは物理ハードウェアとして実装されている。
以下に、sld\_hubおよびsld\_nodeの内部ブロック図を図示する。


IPのドキュメントは以下のリンクより閲覧できる。
- [Virtual JTAG Intel® FPGA IP Core User Guide](https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/ug/ug_virtualjtag.pdf)
- [仮想JTAG（altera_virtual_jtag)IP コアのユーザーガイド](https://www.intel.co.jp/content/www/jp/ja/programmable/documentation/bhc1411109490717.html)

#### Virtual JTAGのインスタンスの作成
Quartus Megafunctionを使って、Virtual JTAGのインスタンスファイル？を作る。


```verilog
// vjtag.v

// Generated using ACDS version 20.1 711

`timescale 1 ps / 1 ps
module vjtag (
		output wire       tdi,                // jtag.tdi
		input  wire       tdo,                //     .tdo
		output wire [4:0] ir_in,              //     .ir_in
		input  wire [4:0] ir_out,             //     .ir_out
		output wire       virtual_state_cdr,  //     .virtual_state_cdr
		output wire       virtual_state_sdr,  //     .virtual_state_sdr
		output wire       virtual_state_e1dr, //     .virtual_state_e1dr
		output wire       virtual_state_pdr,  //     .virtual_state_pdr
		output wire       virtual_state_e2dr, //     .virtual_state_e2dr
		output wire       virtual_state_udr,  //     .virtual_state_udr
		output wire       virtual_state_cir,  //     .virtual_state_cir
		output wire       virtual_state_uir,  //     .virtual_state_uir
		output wire       tms,                //     .tms
		output wire       jtag_state_tlr,     //     .jtag_state_tlr
		output wire       jtag_state_rti,     //     .jtag_state_rti
		output wire       jtag_state_sdrs,    //     .jtag_state_sdrs
		output wire       jtag_state_cdr,     //     .jtag_state_cdr
		output wire       jtag_state_sdr,     //     .jtag_state_sdr
		output wire       jtag_state_e1dr,    //     .jtag_state_e1dr
		output wire       jtag_state_pdr,     //     .jtag_state_pdr
		output wire       jtag_state_e2dr,    //     .jtag_state_e2dr
		output wire       jtag_state_udr,     //     .jtag_state_udr
		output wire       jtag_state_sirs,    //     .jtag_state_sirs
		output wire       jtag_state_cir,     //     .jtag_state_cir
		output wire       jtag_state_sir,     //     .jtag_state_sir
		output wire       jtag_state_e1ir,    //     .jtag_state_e1ir
		output wire       jtag_state_pir,     //     .jtag_state_pir
		output wire       jtag_state_e2ir,    //     .jtag_state_e2ir
		output wire       jtag_state_uir,     //     .jtag_state_uir
		output wire       tck                 //  tck.clk
	);

	sld_virtual_jtag #(
		.sld_auto_instance_index ("YES"),
		.sld_instance_index      (0),
		.sld_ir_width            (5)
	) virtual_jtag_0 (
		.tdi                (tdi),                // jtag.tdi
		.tdo                (tdo),                //     .tdo
		.ir_in              (ir_in),              //     .ir_in
		.ir_out             (ir_out),             //     .ir_out
		.virtual_state_cdr  (virtual_state_cdr),  //     .virtual_state_cdr
		.virtual_state_sdr  (virtual_state_sdr),  //     .virtual_state_sdr
		.virtual_state_e1dr (virtual_state_e1dr), //     .virtual_state_e1dr
		.virtual_state_pdr  (virtual_state_pdr),  //     .virtual_state_pdr
		.virtual_state_e2dr (virtual_state_e2dr), //     .virtual_state_e2dr
		.virtual_state_udr  (virtual_state_udr),  //     .virtual_state_udr
		.virtual_state_cir  (virtual_state_cir),  //     .virtual_state_cir
		.virtual_state_uir  (virtual_state_uir),  //     .virtual_state_uir
		.tms                (tms),                //     .tms
		.jtag_state_tlr     (jtag_state_tlr),     //     .jtag_state_tlr
		.jtag_state_rti     (jtag_state_rti),     //     .jtag_state_rti
		.jtag_state_sdrs    (jtag_state_sdrs),    //     .jtag_state_sdrs
		.jtag_state_cdr     (jtag_state_cdr),     //     .jtag_state_cdr
		.jtag_state_sdr     (jtag_state_sdr),     //     .jtag_state_sdr
		.jtag_state_e1dr    (jtag_state_e1dr),    //     .jtag_state_e1dr
		.jtag_state_pdr     (jtag_state_pdr),     //     .jtag_state_pdr
		.jtag_state_e2dr    (jtag_state_e2dr),    //     .jtag_state_e2dr
		.jtag_state_udr     (jtag_state_udr),     //     .jtag_state_udr
		.jtag_state_sirs    (jtag_state_sirs),    //     .jtag_state_sirs
		.jtag_state_cir     (jtag_state_cir),     //     .jtag_state_cir
		.jtag_state_sir     (jtag_state_sir),     //     .jtag_state_sir
		.jtag_state_e1ir    (jtag_state_e1ir),    //     .jtag_state_e1ir
		.jtag_state_pir     (jtag_state_pir),     //     .jtag_state_pir
		.jtag_state_e2ir    (jtag_state_e2ir),    //     .jtag_state_e2ir
		.jtag_state_uir     (jtag_state_uir),     //     .jtag_state_uir
		.tck                (tck)                 //  tck.clk
	);
endmodule
```

NSLから使えるようにヘッダーファイルを書く。 
なお、クロック供給などは不要で、合成時にうまくやってくれる。
ちなみに、JTAGのコントローラーは別に実装されているらしく、クロックやtdi, tdoなどの信号は
sld\_hubなどに供給されるようである。
しかし、VJTAGのインスタンス(sld\_node)や(sld\_hub)はロジックエレメントを消費する。
`jtag_*`の信号はデバック用途であり、使用不可である。
この信号はJTAGのコントローラーから来ている。
`virtual_*`の信号はVJTAGのステートマシンの状態である。

```c
declare vjtag interface {
	output		tdi;
	input		tdo;
	output		ir_in[5];
	input		ir_out[5];
	func_out	virtual_state_cdr;
	func_out	virtual_state_sdr;
	func_out	virtual_state_e1dr;
	func_out	virtual_state_pdr;
	func_out	virtual_state_e2dr;
	func_out	virtual_state_udr;
	func_out	virtual_state_cir;
	func_out	virtual_state_uir;
	output		tms;
	func_out	jtag_state_tlr;
	func_out	jtag_state_rti;
	func_out	jtag_state_sdrs;
	func_out	jtag_state_cdr;
	func_out	jtag_state_sdr;
	func_out	jtag_state_e1dr;
	func_out	jtag_state_pdr;
	func_out	jtag_state_e2dr;
	func_out	jtag_state_udr;
	func_out	jtag_state_sirs;
	func_out	jtag_state_cir;
	func_out	jtag_state_sir;
	func_out	jtag_state_e1ir;
	func_out	jtag_state_pir;
	func_out	jtag_state_e2ir;
	func_out	jtag_state_uir;
	output		tck;
}
```
### VJTAGとの通信
VJTAGのインスタンスと通信するには、まずsld\_hubと通信をする必要がある。
sld\_hubには２つのオペレーションが定義されている。
なお、sld\_hubのIRはインスタンスによらず、10bitである。
sld\_hubに接続されているインスタンスの情報を取り出すことができるが、今回は省略する。
ドキュメントおよび`riscv-openocd/src/target/riscv/riscv_tap_vjtag.c`を見てほしい。
なお、VJTAGのインスタンスのIR/DRはsld\_hubのサブセットという形な模様。
そのため、それぞれVIR(Virtual IR)およびVDR(Virtual DR)と呼ばれている。
ブロック図を以下に示す(仕様から書き起こしたので、もしかしたら間違いがあるかもしれない）。
{{<figure src="./image02.png" >}}

なお、sld\_hubには、２つの命令がある。
これは、Shift\_IRを用いてsld\_hubのIR(10 bit)へ書き込むことで命令を発行できる。
- USER0
VDRのパスを選択する。
- USER1
VIRのパスを選択する。

OpenOCDでは、or1kのみVJTAGに対応している。
次からは、OpenOCDのRISC-VターゲットにVJTAGサポートを行う。
その後、実際にOpenOCDからVJTAG on DE10-Liteに通信ができるかテストをする。

