---
title: "Reading linux kernel part3"
date: 2021-03-24T13:41:26+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: true
---
`setup_vm`の実装を読んでいく。
<!--more-->

## setup\_vm
`arch/riscv/mm/init.c`
```c
asmlinkage void __init setup_vm(uintptr_t dtb_pa)
{
	uintptr_t va, end_va;
	uintptr_t load_pa = (uintptr_t)(&_start);
	uintptr_t load_sz = (uintptr_t)(&_end) - load_pa;
	uintptr_t map_size = best_map_size(load_pa, MAX_EARLY_MAPPING_SIZE);

	va_pa_offset = PAGE_OFFSET - load_pa;
	pfn_base = PFN_DOWN(load_pa);

	/*
	 * Enforce boot alignment requirements of RV32 and
	 * RV64 by only allowing PMD or PGD mappings.
	 */
	BUG_ON(map_size == PAGE_SIZE);

	/* Sanity check alignment and size */
	BUG_ON((PAGE_OFFSET % PGDIR_SIZE) != 0);
	BUG_ON((load_pa % map_size) != 0);
	BUG_ON(load_sz > MAX_EARLY_MAPPING_SIZE);

	/* Setup early PGD for fixmap */
	create_pgd_mapping(early_pg_dir, FIXADDR_START,
			   (uintptr_t)fixmap_pgd_next, PGDIR_SIZE, PAGE_TABLE);
```
`BUG_ON`は、コンディションがtrueの場合、panicするものである。

`include/asm-generic/bug.h`
```asm
/*
 * Don't use BUG() or BUG_ON() unless there's really no way out; one
 * example might be detecting data structure corruption in the middle
 * of an operation that can't be backed out of.  If the (sub)system
 * can somehow continue operating, perhaps with reduced functionality,
 * it's probably not BUG-worthy.
 *
 * If you're tempted to BUG(), think again:  is completely giving up
 * really the *only* solution?  There are usually better options, where
 * users don't need to reboot ASAP and can mostly shut down cleanly.
 */
#ifndef HAVE_ARCH_BUG
#define BUG() do { \
	printk("BUG: failure at %s:%d/%s()!\n", __FILE__, __LINE__, __func__); \
	barrier_before_unreachable(); \
	panic("BUG!"); \
} while (0)
#endif

#ifndef HAVE_ARCH_BUG_ON
#define BUG_ON(condition) do { if (unlikely(condition)) BUG(); } while (0)
#endif
```

なお、`unlikely`、`likely`は条件分岐において、ヒントとしてコンパイラへ与え、通常時の処理を最適化を行う。
`include/linux/compiler.h`
```asm
# define likely(x)	__builtin_expect(!!(x), 1)
# define unlikely(x)	__builtin_expect(!!(x), 0)
```

`setup_vm`で最初に行う処理は
```c
	create_pgd_mapping(early_pg_dir, FIXADDR_START,
			   (uintptr_t)fixmap_pgd_next, PGDIR_SIZE, PAGE_TABLE);
```
である。
`PGD`において、マッピングの作成を行う関数のよう。
引数をよく見てみる。

`arch/linux/riscv/mm/init.c`
```
static void __init create_pgd_mapping(pgd_t *pgdp,
				      uintptr_t va, phys_addr_t pa,
				      phys_addr_t sz, pgprot_t prot)
{
	pgd_next_t *nextp;
	phys_addr_t next_phys;
	uintptr_t pgd_idx = pgd_index(va);

	if (sz == PGDIR_SIZE) {
		if (pgd_val(pgdp[pgd_idx]) == 0)
			pgdp[pgd_idx] = pfn_pgd(PFN_DOWN(pa), prot);
		return;
	}

	if (pgd_val(pgdp[pgd_idx]) == 0) {
		next_phys = alloc_pgd_next(va);
		pgdp[pgd_idx] = pfn_pgd(PFN_DOWN(next_phys), PAGE_TABLE);
		nextp = get_pgd_next_virt(next_phys);
		memset(nextp, 0, PAGE_SIZE);
	} else {
		next_phys = PFN_PHYS(_pgd_pfn(pgdp[pgd_idx]));
		nextp = get_pgd_next_virt(next_phys);
	}

	create_pgd_next_mapping(nextp, va, pa, sz, prot);
}
```
`pgdp`はPGDへのポインタである。
`va`は仮想アドレスで、`pa`にページをマッピングする。
`sz`はマッピングのサイズであり、`prot`はPTEのプロテクションビットである。

`uintptr_t pgd_idx = pgd_index(va);`で、アドレスに対するPGDのindexを求めている。

次に、`PGDIR_SIZE`が作成するマップサイズか確かめている。`PGDIR_SIZE`はSV32の場合、4MBになる。
つまり、4MBのマッピングを作る場合は、一つのPGDエントリーで十分である。
```c
	if (sz == PGDIR_SIZE) {
		if (pgd_val(pgdp[pgd_idx]) == 0)
			pgdp[pgd_idx] = pfn_pgd(PFN_DOWN(pa), prot);
		return;
	}
```
先程求めたindexを使い、PGDにエントリーすでに存在しない場合は、`pfn_pgd`にてエントリーを作成する。
`PFN_DOWN`はアドレスから`PFN`を求める。

`arch/riscv/include/asm/pgtable.h`
```c
static inline pgd_t pfn_pgd(unsigned long pfn, pgprot_t prot)
{
	return __pgd((pfn << _PAGE_PFN_SHIFT) | pgprot_val(prot));
}
```

`include/linux/pfn.h`
```c
#define PFN_UP(x)	(((x) + PAGE_SIZE-1) >> PAGE_SHIFT)
#define PFN_DOWN(x)	((x) >> PAGE_SHIFT)
```

`PGDIR_SIZE`でないマッピングを`PGD`に作る場合を考える。

`arch/riscv/mm/init.c setup_vm続き`
```c
	if (pgd_val(pgdp[pgd_idx]) == 0) {
		next_phys = alloc_pgd_next(va);
		pgdp[pgd_idx] = pfn_pgd(PFN_DOWN(next_phys), PAGE_TABLE);
		nextp = get_pgd_next_virt(next_phys);
		memset(nextp, 0, PAGE_SIZE);
	} else {
		next_phys = PFN_PHYS(_pgd_pfn(pgdp[pgd_idx]));
		nextp = get_pgd_next_virt(next_phys);
	}

	create_pgd_next_mapping(nextp, va, pa, sz, prot);
```
エントリーがPGDに存在するか確かめる。まずは、存在しない(true)の場合を見ていく。
`alloc_pgd_next`はPGDの次のレベルのテーブルを作る。SV32の場合はPUD、PMDはFOLDEDされているため、次のレベルはページテーブルである。
`pgdp`に作成したページテーブルを挿入する。
なお、PAGE\_TABLEというprotは、PRESET(V bit)のみが立っている状態で、non-leaf PTEを指す。

