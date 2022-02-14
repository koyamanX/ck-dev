---
title: "Design of branch instruction issue for RISC-V OoO"
date: 2022-02-15T00:41:52+09:00
author: "@koyamanX"
categories: ["RISCV"]
tags: ["CPU"]
draft: false
---

趣味でRISC-Vのアウト・オブ・オーダープロセッサを実装している。
Tomasulo algorithmに基づいている。
今回は分岐命令の発行部分の検討をしていく。
Commit: df27bdc1b9382573458b5bf571ca32a9fa93325c
<!--more-->

# BRU pipe
今回の実装では分岐命令はBRU(BRanch Unit) pipeで実行する。2-wayのデコードのスーパスカラな実装で、現状はALUパイプは2つある。
ここにBRUパイプとリザベーションステーションを追加する。

# Branch命令の実行
今回はBranch命令を以下のように分類する。

- Branch(opcode == BRANCH)
- Jump(opcode == JAL || opcode == JALR)

現状の実装では、Jump命令は分岐のコンディションを計算する必要ないため、ALUパイプへ発行している。分岐パイプが実装できたら、こちらへ発行するようにする。


## BRU RS
分岐パイプ用のリザベーションステーションは１つとする。（エントリではない）
フェッチパケット(命令フェッチの単位、今回は２命令同時フェッチ）内に分岐命令が２つある場合、デコードステージを２回行う。
一回目では先行の命令のみを発行をする。次のサイクルでは次の命令を発行する。

issueステージでは、分岐命令が１つしか発行されないことを前提する。
issueステージでは、BRU用のリザベーションステーションへ発行する。

オペランドが揃った時点で、BRUパイプへDispatchする。

実行ステージでは、分岐命令の成否とターゲットアドレスを計算する。
つまり、必要なオペランドは以下のものとなる。

- PC
- Imm
- rs1
- rs2

ターゲットアドレスはissueステージで計算可能なので、事前にissueステージでpc+immを計算し、ROB.Targetへ格納しておく。
実行ステージではrs1 Branch Op rs2を計算し、write resultステージで、ROB.ValueへCDBを通してブロードキャストする
Jump命令の場合は常に、1を格納する。


## BRU Commit
コミット時には、以下の条件で分岐をする。

```c
if((opcode == JAL || opcode == JALR || opcode == BRANCH)) {
	if(opcode == JAL || opcode == JALR) {
		GPR[rd] = PC4;
	}
	if(ROB.Value[0]) {
		Redirect(ROB.Target);
		// Feedback to Branch prediction unit if it presents
		// Also check prediction and actual result
	}
} else {
	// Normal commit
}
```


