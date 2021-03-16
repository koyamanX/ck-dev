---
title: "Reading OpenSBI part2"
date: 2021-03-16T22:06:05+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---

OpenSBIの実装を読んでいこうと思う。
アセンブリはあまり読みたくないので、できるかぎり飛ばしていく。
<!--more-->

`firmware/fw_base.S`が基本となっており、ファームウェアのタイプによって`firmware/fw_payload.S`や`firmware/fw_dynamic.S`、`firmware/fw_jump.S`が
使われる。

## リンカスクリプト
`firmware/fw_base.ldS`
ちなみに、ldSはプリプロセッサにかけられる。
以下に抜粋する。

```asm
	. = FW_TEXT_START;
	PROVIDE(_fw_start = .);
	. = ALIGN(0x1000);
	.text :
 	{
		PROVIDE(_text_start = .);
		*(.entry)
		*(.text)
		. = ALIGN(8);
		PROVIDE(_text_end = .);
	}
	. = ALIGN(0x1000);
	.rodata :
	{
		PROVIDE(_rodata_start = .);
		*(.rodata .rodata.*)
		. = ALIGN(8);
		PROVIDE(_rodata_end = .);
	}
	. = ALIGN(0x1000);
	.data :
	{
		PROVIDE(_data_start = .);

		*(.sdata)
		*(.sdata.*)
		*(.data)
		*(.data.*)
		*(.readmostly.data)
		*(*.data)
		. = ALIGN(8);

		PROVIDE(_data_end = .);
	}
	. = ALIGN(0x1000);
	.bss :
	{
		PROVIDE(_bss_start = .);
		*(.sbss)
		*(.sbss.*)
		*(.bss)
		*(.bss.*)
		. = ALIGN(8);
		PROVIDE(_bss_end = .);
	}
	. = ALIGN(0x1000);
	PROVIDE(_fw_end = .);
```
リンカスクリプトをもとに、メモリマップを作成。
{{<figure src="./image00.png" >}}

OpenSBIの実態はFW\_TEXT\_STARTで指定したアドレスから始まる。

## fw\_base.S
エントリーポイントは`_start`である。
```asm
_start:
	/* a0, a1, a2は前段のブートローダーからの情報を保持しているはず。 */
	/* レジスタを退避(s0, s1, s2) <- (a0, a1, a2) */
	MOV_3R	s0, a0, s1, a1, s2, a2
	/* a0, a1, a2が使える */
	call	fw_boot_hart	/* fw_payload.S(FW_PAYLOADを仮定) */
	/* a0 <- -1 */
	add	a6, a0, zero	/* a6 <- -1 + 0 */
	/* レジスタを復帰(a0, a1, a2) <- (s0, s1, s2) */
	MOV_3R	a0, s0, a1, s1, a2, s2
	li	a7, -1	/* a7 <- -1 */
	beq	a6, a7, _try_lottery	/* always taken with this code */
	bne	a0, a6, _wait_relocate_copy_done
_try_lottery:
	la	a6, _relocate_lottery	/* symbol in bss section (inited to zero) */
	li	a7, 1
	amoadd.w a6, a7, (a6)	/* a6 <- *(a6), *(a6) <- a7 */
	/* 一番最初に_relocate_lottery番地から0を読み出したhartがブートhartになる。*/
	bnez	a6, _wait_relocate_copy_done	/* それ以外のhartはboot hartが終わるのを待つ */
...
```

## fw\_payload.S
```asm
fw_boot_hart:
	li	a0, -1
	ret
```

## \_relocate
`fw_base.S`続き\
`load address != link address`のときに、relocateを行う。\
例えば、`load address = 0x00000000`、`link address = 0x80000000`を考える。\
- リンク時
	- `_start = 0x80000000`
	- `_fw_start = 0x80000000`
- ロード時
	- `_start = 0x00000000`
	- `_fw_start = 0x00000000`


`_link_start`, `_load_start`は`_fw_start`自体の値を持っているため、リンク時の`_fw_start = 0x80000000`となる。\
`la`命令を用いたアドレス生成はauipc, addペアでPC-Relativeでアドレス生成がされるので、実行時のロードアドレスになる。

```asm
	la	t0, _load_start			/* t0 <- _load_start */
	la	t1, _start				/* load address (PC-Relative)*/
	REG_S	t1, 0(t0)			/* *(_load_start) <- _start */
_relocate:
	la	t0, _link_start			/* t0 <- _link_start */
	REG_L	t0, 0(t0)			/* t0 <- 0x80000000(_fw_start) */
	la	t1, _link_end			/* t1 <- _link_end */
	REG_L	t1, 0(t1)			/* t1 <- _fw_reloc_end(Link Address) */
	la	t2, _load_start			/* t2 <- _load_start */
	REG_L	t2, 0(t2)			/* t2 <- _0x00000000(_start) */
	/* calc size in bytes */
	sub	t3, t1, t0				/* t3 <- _fw_reloc_end(Link Address) - _fw_start(Link Address)*/
	/* calc load_end */
	add	t3, t3, t2				/* t3 <- t3 + _start(Load Address) */
	beq	t0, t2, _relocate_done	/* if link address == load address, then _relocate_done */
	la	t4, _relocate_done
	sub	t4, t4, t2				/* offset of _relocate_done from _start(load address) */
	add	t4, t4, t0				/* link address of _relocate_done */
	/* if load address < link address then _relocate_copy_to_upper */
	blt	t2, t0, _relocate_copy_to_upper
_relocate_copy_to_lower:
	ble	t1, t2, _relocate_copy_to_lower_loop
	la	t3, _relocate_lottery
	BRANGE	t2, t1, t3, _start_hang
	la	t3, _boot_status
	BRANGE	t2, t1, t3, _start_hang
	la	t3, _relocate
	la	t5, _relocate_done
	BRANGE	t2, t1, t3, _start_hang
	BRANGE	t2, t1, t5, _start_hang
	BRANGE  t3, t5, t2, _start_hang
_relocate_copy_to_lower_loop:
	REG_L	t3, 0(t2)
	REG_S	t3, 0(t0)
	add	t0, t0, __SIZEOF_POINTER__
	add	t2, t2, __SIZEOF_POINTER__
	blt	t0, t1, _relocate_copy_to_lower_loop
	jr	t4
_relocate_copy_to_upper:
	/* t3: load end, t0: link start */
	/* if t3 <= t0 then _relocate_copy_to_upper_loop */
	ble	t3, t0, _relocate_copy_to_upper_loop
	/* t2 <- _relocate_lottery */
	/* _relocate_lottery == number of hart */
	la	t2, _relocate_lottery
	/* if not _link_start <= _relocate_lotttery && _relocate_lottery < _load_end, jump _start_hang*/
	BRANGE	t0, t3, t2, _start_hang
	la	t2, _boot_status
	/* if not _link_start <= _boot_status && _boot_status < _load_end, jump _start_hang*/
	BRANGE	t0, t3, t2, _start_hang
	la	t2, _relocate
	la	t5, _relocate_done
	/* if not _link_start <= _relocate && _relocate < _load_end, jump _start_hang*/
	BRANGE	t0, t3, t2, _start_hang
	/* if not _link_start <= _relocate_done && _relocate_done < _load_end, jump _start_hang*/
	BRANGE	t0, t3, t5, _start_hang
	/* if not _relocate <= _link_start && _link_start < _relocate_done, jump _start_hang*/
	BRANGE	t2, t5, t0, _start_hang
_relocate_copy_to_upper_loop:
	/* t3: load end, t1: fw_reloc_end(link end), t0: link start */
	add	t3, t3, -__SIZEOF_POINTER__				/* t3 <- load end - 4 */
	add	t1, t1, -__SIZEOF_POINTER__				/* t1 <- link end - 4*/
	REG_L	t2, 0(t3)							/* t2 <- *(t3) */
	REG_S	t2, 0(t1)							/* *(t1) <- t2 */
	blt	t0, t1, _relocate_copy_to_upper_loop	/* link start < link end */
	jr	t4										/* link address of _relocate_done */
_wait_relocate_copy_done:
	la	t0, _start
	la	t1, _link_start
	REG_L	t1, 0(t1)
	beq	t0, t1, _wait_for_boot_hart				/* load address == link address */
	la	t2, _boot_status
	la	t3, _wait_for_boot_hart
	/* relocate load address of _wait_for_boot_hart to link address */
	sub	t3, t3, t0
	add	t3, t3, t1
1:
	/* waitting for relocate copy done (_boot_status == 1) */
	li	t4, BOOT_STATUS_RELOCATE_DONE
	REG_L	t5, 0(t2)		/* t2 <- _boot_status */
	/* Reduce the bus traffic so that boot hart may proceed faster */
	nop
	nop
	nop
	/* t5 <- *(_boot_status); t4 <- BOOT_STATUS_RELOCATE_DONE; */
	bgt     t4, t5, 1b
	jr	t3						/* wait_for_boot_hart */
_relocate_done:
	/*
	 * Mark relocate copy done
	 * Use _boot_status copy relative to the load address
	 */
	la	t0, _boot_status
	la	t1, _link_start
	REG_L	t1, 0(t1)
	la	t2, _load_start
	REG_L	t2, 0(t2)
	sub	t0, t0, t1
	add	t0, t0, t2
	li	t1, BOOT_STATUS_RELOCATE_DONE
	REG_S	t1, 0(t0)
	fence	rw, rw
...
```
```asm
...
_load_start:
	RISCV_PTR	_fw_start
_link_start:
	RISCV_PTR	_fw_start
_link_end:
	RISCV_PTR	_fw_reloc_end
...
```
{{<figure src="./image01.png" >}}

今回は、`_relocate_done`までにしておく。
これ以降は、link addressになる。
