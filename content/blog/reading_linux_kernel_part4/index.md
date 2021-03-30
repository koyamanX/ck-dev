---
title: "Reading linux kernel part4"
date: 2021-03-30T22:23:30+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: false
---

`setup_vm`の続きを読んでいく。
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

#ifndef __PAGETABLE_PMD_FOLDED
	/* Setup fixmap PMD */
	create_pmd_mapping(fixmap_pmd, FIXADDR_START,
			   (uintptr_t)fixmap_pte, PMD_SIZE, PAGE_TABLE);
	/* Setup trampoline PGD and PMD */
	create_pgd_mapping(trampoline_pg_dir, PAGE_OFFSET,
			   (uintptr_t)trampoline_pmd, PGDIR_SIZE, PAGE_TABLE);
	create_pmd_mapping(trampoline_pmd, PAGE_OFFSET,
			   load_pa, PMD_SIZE, PAGE_KERNEL_EXEC);
#else
	/* Setup trampoline PGD */
	create_pgd_mapping(trampoline_pg_dir, PAGE_OFFSET,
			   load_pa, PGDIR_SIZE, PAGE_KERNEL_EXEC);
#endif
	/*
	 * Setup early PGD covering entire kernel which will allows
	 * us to reach paging_init(). We map all memory banks later
	 * in setup_vm_final() below.
	 */
	end_va = PAGE_OFFSET + load_sz;
	for (va = PAGE_OFFSET; va < end_va; va += map_size)
		create_pgd_mapping(early_pg_dir, va,
				   load_pa + (va - PAGE_OFFSET),
				   map_size, PAGE_KERNEL_EXEC);

	/* Create fixed mapping for early FDT parsing */
	end_va = __fix_to_virt(FIX_FDT) + FIX_FDT_SIZE;
	for (va = __fix_to_virt(FIX_FDT); va < end_va; va += PAGE_SIZE)
		create_pte_mapping(fixmap_pte, va,
				   dtb_pa + (va - __fix_to_virt(FIX_FDT)),
				   PAGE_SIZE, PAGE_KERNEL);

	/* Save pointer to DTB for early FDT parsing */
	dtb_early_va = (void *)fix_to_virt(FIX_FDT) + (dtb_pa & ~PAGE_MASK);
	/* Save physical address for memblock reservation */
	dtb_early_pa = dtb_pa;
}
```

`arch/riscv/mm/init.c`
`setup_vm`続き
```c
	uintptr_t load_pa = (uintptr_t)(&_start);
	uintptr_t load_sz = (uintptr_t)(&_end) - load_pa;
	uintptr_t map_size = best_map_size(load_pa, MAX_EARLY_MAPPING_SIZE);
```
`arch/riscv/mm/init.c`
```c
static uintptr_t __init best_map_size(phys_addr_t base, phys_addr_t size)
{
	/* Upgrade to PMD_SIZE mappings whenever possible */
	if ((base & (PMD_SIZE - 1)) || (size & (PMD_SIZE - 1)))
		return PAGE_SIZE;

	return PMD_SIZE;
}
```

`load_pa`は0x80400000である。
`load_sz`はload addressの`_end`から`_start`を引くことで、カーネル自体のサイズを求める。
`map_size`は`best_map_size`で求めるが、`best_map_size`では、可能な限り`PMD_SIZE`(`PGDIR_SIZE`)へpromoteする。
不可能な場合は`PAGE_SIZE`にする。
今回のパターンでは`PGDIR_SIZE`へpromoteしており、4MBページングを行う。

`setup_vm`では、カーネルをリンクアドレスからロードアドレスへマッピングする。
また、fixmap領域にDTBをマッピングする。

以下に`setup_vm`で作成する`early_pg_dir`を図示する。
{{<figure src="./image00.png" >}}
以下に`setup_vm`で作成する`trampoline_pg_dir`を図示する。
{{<figure src="./image01.png" >}}
最後に、dtbの仮想アドレスと物理アドレスを保存する。
```c
	/* Save pointer to DTB for early FDT parsing */
	dtb_early_va = (void *)fix_to_virt(FIX_FDT) + (dtb_pa & ~PAGE_MASK);
	/* Save physical address for memblock reservation */
	dtb_early_pa = dtb_pa;
```
