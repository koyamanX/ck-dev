---
title: "Reading linux kernel part2"
date: 2021-03-23T13:48:04+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: false
---
`setup_vm`を読んでいく。
<!--more-->

## setup\_vm
`arch/riscv/mm/init.c`

`setup_vm`の段階では、load addressからlink addressへのリロケーションが終わっていないので`PC-relative`なコード生成が必須である。
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

### pgtable.h
わからないので、まずはヘッダーファイルから読んでいく。
ページング関連で主要なのヘッダーファイルは`arch/riscv/include/asm/pgtable.h`っぽい。
`arch/riscv/include/asm/pgtable.h`
```asm
...
#ifdef CONFIG_64BIT
#include <asm/pgtable-64.h>
#else
#include <asm/pgtable-32.h>
#endif /* CONFIG_64BIT */
...
```

### pgtable-32.h
RV32の場合は`arch/riscv/include/asm/pgtable-32.h`も関連してくる。
まずは、こちらから読んでいく。
`arch/riscv/include/asm/pgtable-32.h`
```asm
#include <asm-generic/pgtable-nopmd.h>
#include <linux/const.h>

/* Size of region mapped by a page global directory */
#define PGDIR_SHIFT     22
#define PGDIR_SIZE      (_AC(1, UL) << PGDIR_SHIFT)
#define PGDIR_MASK      (~(PGDIR_SIZE - 1))
```

`PGDIR_SIZE`は`0x400000`となる。
`PGDIR_MASK`は`0xffc00000`となる。

### Linux 5-Level Paging
Linuxでは、仮想アドレスを5つに分けて物理アドレスにマッピングする。
一部のx86\_64プロセッサではサポートしているようである。
5-Level Pagingのイメージ図を示す。
{{<figure src="./image00.png" >}}
それぞれ、Page Global Directory(PGD)、Page 4-Level Directory(P4D)、Page Upper Directory(PUD)、PMD(Page Middle Directory)、PTE(Page Table Entry)である。
また、Physical PageはPage Frame Number(PFN)で一意に求めることができる。
`PAGE_OFFSET`が0x1000の場合、
例えば、0x00000000番地ではPFNは0(page 0)となり、0x00001000番地はPFNは1(page 1)となる。

### RISC-Vページング
RISC-VにおけるLinuxのページングのイメージ図を示す。
PUDはRISC-Vでは、使用しない。
`arch/riscv/include/asm/pgtable.h`
```asm
#ifndef __ASSEMBLY__
/* Page Upper Directory not used in RISC-V */
#include <asm-generic/pgtable-nopud.h>
```
また、P4DについてRISC-Vでは使用しない。

`include/asm-generic/pgtable-nopud.h`
```asm
#ifndef __ASSEMBLY__

#include <asm-generic/pgtable-nop4d.h>

#define __PAGETABLE_PUD_FOLDED 1
```
`include/asm-generic/pgtable-nop4d.h`
```asm
#define __PAGETABLE_P4D_FOLDED 1
```
{{<figure src="./image01.png" >}}
なお、SV32ページングでは、最大2段ページングが可能である。
よって、SV32では、PGD -> Page Tableの順で索引する。
なお、PGD内にLeaf PTE(V&&((R|W|X)&&!(!R|W)))がある場合は、4Mページングとなる。
RISC-Vでは、`satp`レジスタがPGDの物理アドレス22ビットを保持する(ページウォーク時には左に2回シフト)。
4Mページングの場合はPGDを一回索引するだけである。(offsetは22bitとなる)
{{<figure src="./image02.png" >}}

### SV32ページング
`arch/riscv/include/asm/pgtable-32.h`
```asm
/* Size of region mapped by a page global directory */
#define PGDIR_SHIFT     22
#define PGDIR_SIZE      (_AC(1, UL) << PGDIR_SHIFT)
#define PGDIR_MASK      (~(PGDIR_SIZE - 1))
```

`arch/riscv/include/asm/page.h`
```asm
#define PAGE_SHIFT	(12)
#define PAGE_SIZE	(_AC(1, UL) << PAGE_SHIFT)
#define PAGE_MASK	(~(PAGE_SIZE - 1))
...
#define HPAGE_SHIFT		PMD_SHIFT
#define HPAGE_SIZE		(_AC(1, UL) << HPAGE_SHIFT)
#define HPAGE_MASK              (~(HPAGE_SIZE - 1))
#define HUGETLB_PAGE_ORDER      (HPAGE_SHIFT - PAGE_SHIFT)
...
/* Page Global Directory entry */
typedef struct {
	unsigned long pgd;
} pgd_t;

/* Page Table entry */
typedef struct {
	unsigned long pte;
} pte_t;

typedef struct {
	unsigned long pgprot;
} pgprot_t;

typedef struct page *pgtable_t;
```
`struct page`は`include/linux/mm_types.h`で定義されている、ページ記述子である。

`SHIFT`マクロはあるレベルにおいて、マップされる長さを示す。
例えば、Page Tableであれば、`PAGE_SHIFT`は12なので、`2**12 = 4KB`がマッピングの`SIZE`となる。
`MASK`マクロは`SHIFT`より上位のビットすべてをマスク可能な定数である。例えば、`SHIFT`が12の場合、`0xfffff000`となる。


`PAGE_`マクロは、4KBページング用のマクロであり、`HPAGE`は4MBページング(Huge Page)である。

PTEのフラグは`pgprot_t`構造体に格納する。

`pgd_t`、`pte_t`、`pgprot_t`の値を作るマクロとして、それぞれ`pgd_val(val)`、`pte_val(val)`、`pgprot_val(val)`が用意されている。
また、キャスト用のマクロとして`__pgd(var)`、`__pte(var)`、`__pgprot(var)`が用意されている。

PTEのフラグ用のマクロは`arch/riscv/include/asm/pgtable-bits.h`で定義されている。
`arch/riscv/include/asm/pgtable-bits.h`
```asm
#define _PAGE_PRESENT   (1 << 0)
#define __PAGE_READ      (1 << 1)    /* Readable */
#define __PAGE_WRITE     (1 << 2)    /* Writable */
#define __PAGE_EXEC      (1 << 3)    /* Executable */
#define _PAGE_USER      (1 << 4)    /* User */
#define _PAGE_GLOBAL    (1 << 5)    /* Global */
#define _PAGE_ACCESSED  (1 << 6)    /* Set by hardware on any access */
#define _PAGE_DIRTY     (1 << 7)    /* Set by hardware on any write */
#define _PAGE_SOFT      (1 << 8)    /* Reserved for software */
```

`pgd_offset`は`mm_struct`とアドレスを取り、与えられたアドレスをカバーする`pgd`を返す。
`pte_offset`は`pmd`とアドレスを取り、与えられたアドレスをカバーする`pte`を返す。
`pte_none`、`pgd_none`マクロは、与えられたエントリーが存在しない場合は1を返す。
`pte_present`、`pgd_present`マクロは、与えられたエントリーが存在する場合は1を返す。
`pte_clear`、`pgd_clear`マクロは、与えられたエントリーをクリアする。

`arch/riscv/include/asm/pgtable.h`
```asm
static inline int pmd_present(pmd_t pmd)
{
	return (pmd_val(pmd) & (_PAGE_PRESENT | _PAGE_PROT_NONE));
}

static inline int pmd_none(pmd_t pmd)
{
	return (pmd_val(pmd) == 0);
}

static inline int pmd_bad(pmd_t pmd)
{
	return !pmd_present(pmd);
}

#define pmd_leaf	pmd_leaf
static inline int pmd_leaf(pmd_t pmd)
{
	return pmd_present(pmd) &&
	       (pmd_val(pmd) & (_PAGE_READ | _PAGE_WRITE | _PAGE_EXEC));
}

static inline void set_pmd(pmd_t *pmdp, pmd_t pmd)
{
	*pmdp = pmd;
}

static inline void pmd_clear(pmd_t *pmdp)
{
	set_pmd(pmdp, __pmd(0));
}

static inline pgd_t pfn_pgd(unsigned long pfn, pgprot_t prot)
{
	return __pgd((pfn << _PAGE_PFN_SHIFT) | pgprot_val(prot));
}

static inline unsigned long _pgd_pfn(pgd_t pgd)
{
	return pgd_val(pgd) >> _PAGE_PFN_SHIFT;
}

static inline struct page *pmd_page(pmd_t pmd)
{
	return pfn_to_page(pmd_val(pmd) >> _PAGE_PFN_SHIFT);
}

static inline unsigned long pmd_page_vaddr(pmd_t pmd)
{
	return (unsigned long)pfn_to_virt(pmd_val(pmd) >> _PAGE_PFN_SHIFT);
}

/* Yields the page frame number (PFN) of a page table entry */
static inline unsigned long pte_pfn(pte_t pte)
{
	return (pte_val(pte) >> _PAGE_PFN_SHIFT);
}

#define pte_page(x)     pfn_to_page(pte_pfn(x))

/* Constructs a page table entry */
static inline pte_t pfn_pte(unsigned long pfn, pgprot_t prot)
{
	return __pte((pfn << _PAGE_PFN_SHIFT) | pgprot_val(prot));
}

#define mk_pte(page, prot)       pfn_pte(page_to_pfn(page), prot)

static inline int pte_present(pte_t pte)
{
	return (pte_val(pte) & (_PAGE_PRESENT | _PAGE_PROT_NONE));
}

static inline int pte_none(pte_t pte)
{
	return (pte_val(pte) == 0);
}

static inline int pte_write(pte_t pte)
{
	return pte_val(pte) & _PAGE_WRITE;
}

static inline int pte_exec(pte_t pte)
{
	return pte_val(pte) & _PAGE_EXEC;
}

static inline int pte_huge(pte_t pte)
{
	return pte_present(pte)
		&& (pte_val(pte) & (_PAGE_READ | _PAGE_WRITE | _PAGE_EXEC));
}

static inline int pte_dirty(pte_t pte)
{
	return pte_val(pte) & _PAGE_DIRTY;
}

static inline int pte_young(pte_t pte)
{
	return pte_val(pte) & _PAGE_ACCESSED;
}

static inline int pte_special(pte_t pte)
{
	return pte_val(pte) & _PAGE_SPECIAL;
}

/* static inline pte_t pte_rdprotect(pte_t pte) */

static inline pte_t pte_wrprotect(pte_t pte)
{
	return __pte(pte_val(pte) & ~(_PAGE_WRITE));
}

/* static inline pte_t pte_mkread(pte_t pte) */

static inline pte_t pte_mkwrite(pte_t pte)
{
	return __pte(pte_val(pte) | _PAGE_WRITE);
}

/* static inline pte_t pte_mkexec(pte_t pte) */

static inline pte_t pte_mkdirty(pte_t pte)
{ return __pte(pte_val(pte) | _PAGE_DIRTY); } 
static inline pte_t pte_mkclean(pte_t pte)
{
	return __pte(pte_val(pte) & ~(_PAGE_DIRTY));
}

static inline pte_t pte_mkyoung(pte_t pte)
{
	return __pte(pte_val(pte) | _PAGE_ACCESSED);
}

static inline pte_t pte_mkold(pte_t pte)
{
	return __pte(pte_val(pte) & ~(_PAGE_ACCESSED));
}

static inline pte_t pte_mkspecial(pte_t pte)
{
	return __pte(pte_val(pte) | _PAGE_SPECIAL);
}

static inline pte_t pte_mkhuge(pte_t pte)
{
	return pte;
}

/* Modify page protection bits */
static inline pte_t pte_modify(pte_t pte, pgprot_t newprot)
{
	return __pte((pte_val(pte) & _PAGE_CHG_MASK) | pgprot_val(newprot));
}
#define pgd_ERROR(e) \
	pr_err("%s:%d: bad pgd " PTE_FMT ".\n", __FILE__, __LINE__, pgd_val(e))


/* Commit new configuration to MMU hardware */
static inline void update_mmu_cache(struct vm_area_struct *vma,
	unsigned long address, pte_t *ptep)
{
	/*
	 * The kernel assumes that TLBs don't cache invalid entries, but
	 * in RISC-V, SFENCE.VMA specifies an ordering constraint, not a
	 * cache flush; it is necessary even after writing invalid entries.
	 * Relying on flush_tlb_fix_spurious_fault would suffice, but
	 * the extra traps reduce performance.  So, eagerly SFENCE.VMA.
	 */
	local_flush_tlb_page(address);
}

#define __HAVE_ARCH_PTE_SAME
static inline int pte_same(pte_t pte_a, pte_t pte_b)
{
	return pte_val(pte_a) == pte_val(pte_b);
}

/*
 * Certain architectures need to do special things when PTEs within
 * a page table are directly modified.  Thus, the following hook is
 * made available.
 */
static inline void set_pte(pte_t *ptep, pte_t pteval)
{
	*ptep = pteval;
}

void flush_icache_pte(pte_t pte);

static inline void set_pte_at(struct mm_struct *mm,
	unsigned long addr, pte_t *ptep, pte_t pteval)
{
	if (pte_present(pteval) && pte_exec(pteval))
		flush_icache_pte(pteval);

	set_pte(ptep, pteval);
}

static inline void pte_clear(struct mm_struct *mm,
	unsigned long addr, pte_t *ptep)
{
	set_pte_at(mm, addr, ptep, __pte(0));
}

#define __HAVE_ARCH_PTEP_SET_ACCESS_FLAGS
static inline int ptep_set_access_flags(struct vm_area_struct *vma,
					unsigned long address, pte_t *ptep,
					pte_t entry, int dirty)
{
	if (!pte_same(*ptep, entry))
		set_pte_at(vma->vm_mm, address, ptep, entry);
	/*
	 * update_mmu_cache will unconditionally execute, handling both
	 * the case that the PTE changed and the spurious fault case.
	 */
	return true;
}

#define __HAVE_ARCH_PTEP_GET_AND_CLEAR
static inline pte_t ptep_get_and_clear(struct mm_struct *mm,
				       unsigned long address, pte_t *ptep)
{
	return __pte(atomic_long_xchg((atomic_long_t *)ptep, 0));
}

#define __HAVE_ARCH_PTEP_TEST_AND_CLEAR_YOUNG
static inline int ptep_test_and_clear_young(struct vm_area_struct *vma,
					    unsigned long address,
					    pte_t *ptep)
{
	if (!pte_young(*ptep))
		return 0;
	return test_and_clear_bit(_PAGE_ACCESSED_OFFSET, &pte_val(*ptep));
}

#define __HAVE_ARCH_PTEP_SET_WRPROTECT
static inline void ptep_set_wrprotect(struct mm_struct *mm,
				      unsigned long address, pte_t *ptep)
{
	atomic_long_and(~(unsigned long)_PAGE_WRITE, (atomic_long_t *)ptep);
}

#define __HAVE_ARCH_PTEP_CLEAR_YOUNG_FLUSH
static inline int ptep_clear_flush_young(struct vm_area_struct *vma,
					 unsigned long address, pte_t *ptep)
{
	/*
	 * This comment is borrowed from x86, but applies equally to RISC-V:
	 *
	 * Clearing the accessed bit without a TLB flush
	 * doesn't cause data corruption. [ It could cause incorrect
	 * page aging and the (mistaken) reclaim of hot pages, but the
	 * chance of that should be relatively low. ]
	 *
	 * So as a performance optimization don't flush the TLB when
	 * clearing the accessed bit, it will eventually be flushed by
	 * a context switch or a VM operation anyway. [ In the rare
	 * event of it not getting flushed for a long time the delay
	 * shouldn't really matter because there's no real memory
	 * pressure for swapout to react to. ]
	 */
	return ptep_test_and_clear_young(vma, address, ptep);
}
```

`mk_pte`マクロは`struct page`とプロテクションビットを取り、`pte_t`を返す。
`pte_page`はpteに対応するpageを返す。
`pmd_page`は`pmd`に対応するpageを返す。
`set_pte`はpage tableと`pte_t`を取り、pteをpage tableに挿入する。



## References
- [ARM32 Page Tables](https://people.kernel.org/linusw/arm32-page-tables)
- [Chapter 3  Page Table Management](https://www.kernel.org/doc/gorman/html/understand/understand006.html)
- [第62章 ページ記述子（ struct page )](https://mkguytone.github.io/allocator-navigatable/ch62.html)
- [linux kernel - how to get physical address (memory management)?](https://stackoverflow.com/questions/41090469/linux-kernel-how-to-get-physical-address-memory-management)
- [24.4. 5-level paging](https://www.kernel.org/doc/html/latest/x86/x86_64/5level-paging.html)
- [Linux メモリ管理 徹底入門(カーネル編)](https://www.kimullaa.com/entry/2020/02/16/155746)
- [hiboma/Linuxカーネル解読室/Linuxカーネル解読室-1.md](https://github.com/hiboma/hiboma/blob/master/Linux%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB%E8%A7%A3%E8%AA%AD%E5%AE%A4/Linux%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB%E8%A7%A3%E8%AA%AD%E5%AE%A4-1.md)
- [【Linux】mm_struct は何してるか](https://ryuichi1208.hateblo.jp/entry/2019/01/29/221155)
- [メモリ管理、アドレス空間、ページテーブル](http://www.coins.tsukuba.ac.jp/~yas/coins/os2-2010/2011-01-25/)



今回はこのくらいにしておく。次回から`arch/riscv/mm/init.c`を読んでいく。
