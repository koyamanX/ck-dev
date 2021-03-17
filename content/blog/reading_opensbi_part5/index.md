---
title: "Reading OpenSBI part5"
date: 2021-03-17T18:39:28+09:00
author: "@koyamanX"
categories: ["OpenSBI"]
tags: ["OpenSBI", "Linux", "RISC-V"]
draft: false
---

[前回]({{<ref "/blog/reading_opensbi_part5/index.md" >}})はプラットフォームの初期化まで読んだ。今回はFDTのリロケーションと`_start_warm`を詳しく読んでいく。
<!--more-->

### FDT relocation
前段のブートローダーより受け取ったFDTのPAをOpenSBIのメモリ空間(`FW_PAYLOAD_FDT_ADDR`により指定されていれば)にリロケーションする。
`FW_PAYLOAD_FDT_ADDR`の指定がない場合は、リロケーションはしない。
`firmware/fw_base.S`
```asm
...
	/*
	 * Relocate Flatened Device Tree (FDT)
	 * source FDT address = previous arg1
	 * destination FDT address = next arg1
	 *
	 * Note: We will preserve a0 and a1 passed by
	 * previous booting stage.
	 */
	beqz	a1, _fdt_reloc_done
	/* Mask values in a3 and a4 */
	li	a3, ~(__SIZEOF_POINTER__ - 1)
	li	a4, 0xff
	/* t1 = destination FDT start address */
	MOV_3R	s0, a0, s1, a1, s2, a2
	call	fw_next_arg1
	add	t1, a0, zero
	MOV_3R	a0, s0, a1, s1, a2, s2
	beqz	t1, _fdt_reloc_done
	beq	t1, a1, _fdt_reloc_done
	and	t1, t1, a3
	/* t0 = source FDT start address */
	add	t0, a1, zero
	and	t0, t0, a3
	/* t2 = source FDT size in big-endian */
#if __riscv_xlen == 64
	lwu	t2, 4(t0)
#else
	lw	t2, 4(t0)
#endif
	/* t3 = bit[15:8] of FDT size */
	add	t3, t2, zero
	srli	t3, t3, 16
	and	t3, t3, a4
	slli	t3, t3, 8
	/* t4 = bit[23:16] of FDT size */
	add	t4, t2, zero
	srli	t4, t4, 8
	and	t4, t4, a4
	slli	t4, t4, 16
	/* t5 = bit[31:24] of FDT size */
	add	t5, t2, zero
	and	t5, t5, a4
	slli	t5, t5, 24
	/* t2 = bit[7:0] of FDT size */
	srli	t2, t2, 24
	and	t2, t2, a4
	/* t2 = FDT size in little-endian */
	or	t2, t2, t3
	or	t2, t2, t4
	or	t2, t2, t5
	/* t2 = destination FDT end address */
	add	t2, t1, t2
	/* FDT copy loop */
	ble	t2, t1, _fdt_reloc_done
_fdt_reloc_again:
	REG_L	t3, 0(t0)
	REG_S	t3, 0(t1)
	add	t0, t0, __SIZEOF_POINTER__
	add	t1, t1, __SIZEOF_POINTER__
	blt	t1, t2, _fdt_reloc_again
_fdt_reloc_done:

	/* mark boot hart done */
	li	t0, BOOT_STATUS_BOOT_HART_DONE
	la	t1, _boot_status
	REG_S	t0, 0(t1)
	fence	rw, rw
	j	_start_warm
...
```
`a1`レジスタは前段のブートローダーより渡された(OpenSBIでないことに注意)、FDTのPAを持っているわけであるが、\
`beqz a1, _fdt_reloc_done`\
により`NULL`であれば、`_fdt_reloc_done`とする。

`a3`レジスタにマスクをつくる。(`__SIZEOF_POINTER__`からワードアドレスのマスクを作る)
例えば、32bitであれば、下位2ビットをマスクする。

レジスタを退避する。`(s0, s1, s2) <- (a0, a1, a2)`
`fw_next_arg1`関数を呼び、FDTのPA(OpenSBIにより渡されるFDT PA、もしあれば)をa0レジスタに取得する。(`firmware/fw_payload.S`)
`t1`レジスタにオリジナルの値(FDT PA src)を保存する。
レジスタの復帰`(a0, a1, a2) <- (s0, s1, s2)`。
読み出した、FDT PAの`NULL`チェックをする。
OpenSBIのFDTと前段のブートローダーより受け取ったFDTが同じもの(アドレス的に)の場合は、リロケーションはしない。(同じ場合はすでに、FDTはOpenSBIのメモリ空間に存在するはず。)

\
ここからの処理は`(a1 != NULL) && (t1 != NULL) && (a1 != t1)`であることが保証されている。
ただし、`a1`は前段のブートローダーからのFDT PAで、`t1`はOpenSBIにより配置されるFDTのPAである。
FDTのPAを先程作ったマスクで、ワードアドレスにする。(FDTはワード境界に配置されていることを前提にしているはず。後続のコードはワード単位でアクセス。）
前段のブートローダーより受け取ったFDT PAを`a1`レジスタから`t0`レジスタへ移す。マスクをして、ワードアドレスにする。
`t0`アドレスはソースアドレスである。

### FDTのサイズ取得
FDTの情報はFDTの先頭にある`fdt header`にて示される。
構造体は以下のようなものとなる。([Device Tree Specification v0.3参考](https://www.devicetree.org/specifications/))
データはBig-endianで表現される。
```c
struct fdt_header {
	uint32_t magic;
	uint32_t totalsize;
	uint32_t off_dt_struct;
	uint32_t off_dt_strings;
	uint32_t off_mem_rsvmap;
	uint32_t version;
	uint32_t last_comp_version;
	uint32_t boot_cpuid_phys;
	uint32_t size_dt_strings;
	uint32_t size_dt_struct;
};
```
ロード命令で構造体のオフセット(0x4)である、totalsizeを`t2`レジスタに取得する。
RV32の場合は`lw`命令でロードできるが、RV64の場合は64ビットに符号拡張されることを防ぐため(`lwu`)を使用する。
RISC-VはLittle-endianなので、totalsizeを`t2`レジスタへ変換をする。
上に上げたコード片からエンディアンの変換部分を抜粋。
```asm
...
	/* t3 = bit[15:8] of FDT size */
	add	t3, t2, zero
	srli	t3, t3, 16
	and	t3, t3, a4
	slli	t3, t3, 8
	/* t4 = bit[23:16] of FDT size */
	add	t4, t2, zero
	srli	t4, t4, 8
	and	t4, t4, a4
	slli	t4, t4, 16
	/* t5 = bit[31:24] of FDT size */
	add	t5, t2, zero
	and	t5, t5, a4
	slli	t5, t5, 24
	/* t2 = bit[7:0] of FDT size */
	srli	t2, t2, 24
	and	t2, t2, a4
	/* t2 = FDT size in little-endian */
	or	t2, t2, t3
	or	t2, t2, t4
	or	t2, t2, t5
...
```

### FDTをソースからディスティネーションへコピー
`t1`レジスタにはOpenSBIによりロードされるFDTのPAがある。(`FW_PAYLOAD_FDT_ADDR`が指定)
`t1`レジスタはディスティネーションを示す。
`t2`レジスタ(totalsize)に、`t1`レジスタを足すことで、ディスティネーションでのFDTの最終アドレス(高位アドレス)がわかる。
`ble	t2, t1, _fdt_reloc_done`にて、totalsizeが正当であるか(0以下でないか)チェックしている。

最初に上げたコード片よりコピー処理を抜粋。
```asm
	ble	t2, t1, _fdt_reloc_done
_fdt_reloc_again:
	REG_L	t3, 0(t0)
	REG_S	t3, 0(t1)
	add	t0, t0, __SIZEOF_POINTER__
	add	t1, t1, __SIZEOF_POINTER__
	blt	t1, t2, _fdt_reloc_again
_fdt_reloc_done:

```

### boot hartの処理完了
`_boot_status`に完了のステータスコードを保存し、fence命令で他のコアから見えることを保証する。
その後、hartのwarm boot(`_start_warm`)の処理になる。
```asm
	/* mark boot hart done */
	li	t0, BOOT_STATUS_BOOT_HART_DONE
	la	t1, _boot_status
	REG_S	t0, 0(t1)
	fence	rw, rw
	j	_start_warm
```

長くなってきたので、`_start_warm`は次回に見送る。
