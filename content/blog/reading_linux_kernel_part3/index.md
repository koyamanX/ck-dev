---
title: "Reading linux kernel part3"
date: 2021-03-24T13:41:26+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: false
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
つまり、実体は`alloc_pte`となる。
`pgdp`に作成したページテーブルを挿入する。
なお、PAGE\_TABLEというprotは、PRESET(V bit)のみが立っている状態で、non-leaf PTEを指す。

`arch/riscv/mm/init.c`
```c
#ifndef __PAGETABLE_PMD_FOLDED
...
#define pgd_next_t		pte_t
#define alloc_pgd_next(__va)	alloc_pte(__va)
#define get_pgd_next_virt(__pa)	get_pte_virt(__pa)
#define create_pgd_next_mapping(__nextp, __va, __pa, __sz, __prot)	\
	create_pte_mapping(__nextp, __va, __pa, __sz, __prot)
#define fixmap_pgd_next		fixmap_pte
#endif
...
static phys_addr_t __init alloc_pte(uintptr_t va)
{
	/*
	 * We only create PMD or PGD early mappings so we
	 * should never reach here with MMU disabled.
	 */
	BUG_ON(!mmu_enabled);

	return memblock_phys_alloc(PAGE_SIZE, PAGE_SIZE);
}
```

## memblock
`memblock`はブート時にカーネルが使用するメモリ管理の手法である。
`memblock`にはAPIが用意されており、このAPIを通して、メモリの追加、取得等を行う。
まずは、構造体を見てみる。

`include/linux/memblock.h`
```c
/**
 * struct memblock - memblock allocator metadata
 * @bottom_up: is bottom up direction?
 * @current_limit: physical address of the current allocation limit
 * @memory: usable memory regions
 * @reserved: reserved memory regions
 */
struct memblock {
	bool bottom_up;  /* is bottom up direction? */
	phys_addr_t current_limit;
	struct memblock_type memory;
	struct memblock_type reserved;
};

/**
 * struct memblock_type - collection of memory regions of certain type
 * @cnt: number of regions
 * @max: size of the allocated array
 * @total_size: size of all regions
 * @regions: array of regions
 * @name: the memory type symbolic name
 */
struct memblock_type {
	unsigned long cnt;
	unsigned long max;
	phys_addr_t total_size;
	struct memblock_region *regions;
	char *name;
};

/**
 * struct memblock_region - represents a memory region
 * @base: base address of the region
 * @size: size of the region
 * @flags: memory region attributes
 * @nid: NUMA node id
 */
struct memblock_region {
	phys_addr_t base;
	phys_addr_t size;
	enum memblock_flags flags;
#ifdef CONFIG_NEED_MULTIPLE_NODES
	int nid;
#endif
};

/**
 * enum memblock_flags - definition of memory region attributes
 * @MEMBLOCK_NONE: no special request
 * @MEMBLOCK_HOTPLUG: hotpluggable region
 * @MEMBLOCK_MIRROR: mirrored region
 * @MEMBLOCK_NOMAP: don't add to kernel direct mapping
 */
enum memblock_flags {
	MEMBLOCK_NONE		= 0x0,	/* No special request */
	MEMBLOCK_HOTPLUG	= 0x1,	/* hotpluggable region */
	MEMBLOCK_MIRROR		= 0x2,	/* mirrored region */
	MEMBLOCK_NOMAP		= 0x4,	/* don't add to kernel direct mapping */
};
```
### `struct memblock`について。
`memblock.bottom_up`はbottom upでメモリ確保をするかを示すフラグである。
`memblock.current_limit`は物理メモリの上限サイズである。
`memblock.memory`は使用可能メモリの領域で、`memblock.reserved`は予約領域である。

### `struct memblock_type`について。
順番は違うが、まず、`memblock_type.regions`はメモリの領域(`struct memblock_region`の配列)である。
`memblock_type.cnt`は`memblock_type.regions`の数で、
`max`は`memblock_regions`のトータルサイズ、
`memblock_type.total_size`は、確保したモリ領域のトータルサイズである。
`memblock_type.name`はメモリブロックの名前である。

### `memblock_region`である。
`memblock_region`はメモリ領域を表す構造体であり、`memblock_region.base`は領域のベースアドレス、`memblock_region.size`は領域のサイズ、`memblock_region.flags`は領域に対するフラグである。
フラグは`enum memblock_flags`で表される。

### memblockの初期化

`mm/memblock.c`
```c
struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,	/* empty dummy entry */
	.memory.max		= INIT_MEMBLOCK_REGIONS,
	.memory.name		= "memory",

	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,	/* empty dummy entry */
	.reserved.max		= INIT_MEMBLOCK_RESERVED_REGIONS,
	.reserved.name		= "reserved",

	.bottom_up		= false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
...
#define INIT_MEMBLOCK_REGIONS			128
#define INIT_PHYSMEM_REGIONS			4
#ifndef INIT_MEMBLOCK_RESERVED_REGIONS
# define INIT_MEMBLOCK_RESERVED_REGIONS		INIT_MEMBLOCK_REGIONS
```

`include/linux/memblock.h`
```c
#define MEMBLOCK_ALLOC_ANYWHERE	(~(phys_addr_t)0)
```

つまり、`memblock.current_limit`は`0xffffffff`である。

### alloc\_pte
`arch/riscv/mm/init.c`
```c
static phys_addr_t __init alloc_pte(uintptr_t va)
{
	/*
	 * We only create PMD or PGD early mappings so we
	 * should never reach here with MMU disabled.
	 */
	BUG_ON(!mmu_enabled);

	return memblock_phys_alloc(PAGE_SIZE, PAGE_SIZE);
}
```
`memblock_phys_alloc`は`memblock.memory.regions`より、`PAGE_SIZE`境界配置された、`PAGE_SIZE`分の領域を確保する。

`include/linux/memblock.h`
```c
static inline phys_addr_t memblock_phys_alloc(phys_addr_t size,
					      phys_addr_t align)
{
	return memblock_phys_alloc_range(size, align, 0,
					 MEMBLOCK_ALLOC_ACCESSIBLE);
}
```
`start:0x00000000`、`end:MEMBLOCK_ALLOC_ACCESSIBLE(0x0)`、`size:PAGE_SIZE`、`align:PAGE_SIZE`を確保する。
`end`が`MEMBLOCK_ALLOC_ACCESSIBLE`の場合は、`memblock_phys_alloc_range`では、`start`~`memblock.current_limit`の領域からメモリ領域を割り当てが制限される。

`mm/memblock.c`
```c
/**
 * memblock_phys_alloc_range - allocate a memory block inside specified range
 * @size: size of memory block to be allocated in bytes
 * @align: alignment of the region and block's size
 * @start: the lower bound of the memory region to allocate (physical address)
 * @end: the upper bound of the memory region to allocate (physical address)
 *
 * Allocate @size bytes in the between @start and @end.
 *
 * Return: physical address of the allocated memory block on success,
 * %0 on failure.
 */
phys_addr_t __init memblock_phys_alloc_range(phys_addr_t size,
					     phys_addr_t align,
					     phys_addr_t start,
					     phys_addr_t end)
{
	return memblock_alloc_range_nid(size, align, start, end, NUMA_NO_NODE,
					false);
}
```

`mm/memblock.c`
```c
/**
 * memblock_alloc_range_nid - allocate boot memory block
 * @size: size of memory block to be allocated in bytes
 * @align: alignment of the region and block's size
 * @start: the lower bound of the memory region to allocate (phys address)
 * @end: the upper bound of the memory region to allocate (phys address)
 * @nid: nid of the free area to find, %NUMA_NO_NODE for any node
 * @exact_nid: control the allocation fall back to other nodes
 *
 * The allocation is performed from memory region limited by
 * memblock.current_limit if @end == %MEMBLOCK_ALLOC_ACCESSIBLE.
 *
 * If the specified node can not hold the requested memory and @exact_nid
 * is false, the allocation falls back to any node in the system.
 *
 * For systems with memory mirroring, the allocation is attempted first
 * from the regions with mirroring enabled and then retried from any
 * memory region.
 *
 * In addition, function sets the min_count to 0 using kmemleak_alloc_phys for
 * allocated boot memory block, so that it is never reported as leaks.
 *
 * Return:
 * Physical address of allocated memory block on success, %0 on failure.
 */
phys_addr_t __init memblock_alloc_range_nid(phys_addr_t size,
					phys_addr_t align, phys_addr_t start,
					phys_addr_t end, int nid,
					bool exact_nid)
{
	enum memblock_flags flags = choose_memblock_flags();
	phys_addr_t found;

	if (WARN_ONCE(nid == MAX_NUMNODES, "Usage of MAX_NUMNODES is deprecated. Use NUMA_NO_NODE instead\n"))
		nid = NUMA_NO_NODE;

	if (!align) {
		/* Can't use WARNs this early in boot on powerpc */
		dump_stack();
		align = SMP_CACHE_BYTES;
	}

again:
	found = memblock_find_in_range_node(size, align, start, end, nid,
					    flags);
	if (found && !memblock_reserve(found, size))
		goto done;

	if (nid != NUMA_NO_NODE && !exact_nid) {
		found = memblock_find_in_range_node(size, align, start,
						    end, NUMA_NO_NODE,
						    flags);
		if (found && !memblock_reserve(found, size))
			goto done;
	}

	if (flags & MEMBLOCK_MIRROR) {
		flags &= ~MEMBLOCK_MIRROR;
		pr_warn("Could not allocate %pap bytes of mirrored memory\n",
			&size);
		goto again;
	}

	return 0;

done:
	/* Skip kmemleak for kasan_init() due to high volume. */
	if (end != MEMBLOCK_ALLOC_KASAN)
		/*
		 * The min_count is set to 0 so that memblock allocated
		 * blocks are never reported as leaks. This is because many
		 * of these blocks are only referred via the physical
		 * address which is not looked up by kmemleak.
		 */
		kmemleak_alloc_phys(found, size, 0, 0);

	return found;
}
...
/**
 * memblock_find_in_range_node - find free area in given range and node
 * @size: size of free area to find
 * @align: alignment of free area to find
 * @start: start of candidate range
 * @end: end of candidate range, can be %MEMBLOCK_ALLOC_ANYWHERE or
 *       %MEMBLOCK_ALLOC_ACCESSIBLE
 * @nid: nid of the free area to find, %NUMA_NO_NODE for any node
 * @flags: pick from blocks based on memory attributes
 *
 * Find @size free area aligned to @align in the specified range and node.
 *
 * When allocation direction is bottom-up, the @start should be greater
 * than the end of the kernel image. Otherwise, it will be trimmed. The
 * reason is that we want the bottom-up allocation just near the kernel
 * image so it is highly likely that the allocated memory and the kernel
 * will reside in the same node.
 *
 * If bottom-up allocation failed, will try to allocate memory top-down.
 *
 * Return:
 * Found address on success, 0 on failure.
 */
static phys_addr_t __init_memblock memblock_find_in_range_node(phys_addr_t size,
					phys_addr_t align, phys_addr_t start,
					phys_addr_t end, int nid,
					enum memblock_flags flags)
{
	phys_addr_t kernel_end, ret;

	/* pump up @end */
	if (end == MEMBLOCK_ALLOC_ACCESSIBLE ||
	    end == MEMBLOCK_ALLOC_KASAN)
		end = memblock.current_limit;

	/* avoid allocating the first page */
	start = max_t(phys_addr_t, start, PAGE_SIZE);
	end = max(start, end);
	kernel_end = __pa_symbol(_end);

	/*
	 * try bottom-up allocation only when bottom-up mode
	 * is set and @end is above the kernel image.
	 */
	if (memblock_bottom_up() && end > kernel_end) {
		phys_addr_t bottom_up_start;

		/* make sure we will allocate above the kernel */
		bottom_up_start = max(start, kernel_end);

		/* ok, try bottom-up allocation first */
		ret = __memblock_find_range_bottom_up(bottom_up_start, end,
						      size, align, nid, flags);
		if (ret)
			return ret;

		/*
		 * we always limit bottom-up allocation above the kernel,
		 * but top-down allocation doesn't have the limit, so
		 * retrying top-down allocation may succeed when bottom-up
		 * allocation failed.
		 *
		 * bottom-up allocation is expected to be fail very rarely,
		 * so we use WARN_ONCE() here to see the stack trace if
		 * fail happens.
		 */
		WARN_ONCE(IS_ENABLED(CONFIG_MEMORY_HOTREMOVE),
			  "memblock: bottom-up allocation failed, memory hotremove may be affected\n");
	}

	return __memblock_find_range_top_down(start, end, size, align, nid,
					      flags);
}
...
/**
 * __memblock_find_range_top_down - find free area utility, in top-down
 * @start: start of candidate range
 * @end: end of candidate range, can be %MEMBLOCK_ALLOC_ANYWHERE or
 *       %MEMBLOCK_ALLOC_ACCESSIBLE
 * @size: size of free area to find
 * @align: alignment of free area to find
 * @nid: nid of the free area to find, %NUMA_NO_NODE for any node
 * @flags: pick from blocks based on memory attributes
 *
 * Utility called from memblock_find_in_range_node(), find free area top-down.
 *
 * Return:
 * Found address on success, 0 on failure.
 */
static phys_addr_t __init_memblock
__memblock_find_range_top_down(phys_addr_t start, phys_addr_t end,
			       phys_addr_t size, phys_addr_t align, int nid,
			       enum memblock_flags flags)
{
	phys_addr_t this_start, this_end, cand;
	u64 i;

	for_each_free_mem_range_reverse(i, nid, flags, &this_start, &this_end,
					NULL) {
		this_start = clamp(this_start, start, end);
		this_end = clamp(this_end, start, end);

		if (this_end < size)
			continue;

		cand = round_down(this_end - size, align);
		if (cand >= this_start)
			return cand;
	}

	return 0;
}
```
つまり、`alloc_pte`にて、`PAGE_SIZE`に境界配置された`PAGE_SIZE`サイズの領域を`memblock`より確保する。

## setup\_vmの続き

### create\_pgd\_mapping
`arch/riscv/mm/init.c`
```
	if (pgd_val(pgdp[pgd_idx]) == 0) {
		next_phys = alloc_pgd_next(va);
		pgdp[pgd_idx] = pfn_pgd(PFN_DOWN(next_phys), PAGE_TABLE);
		nextp = get_pgd_next_virt(next_phys);
		memset(nextp, 0, PAGE_SIZE);
...
```
`pfn_pgd`で、`next_phys`(PAGE_TABLE)に対する`PTE`を作り、`pgdp[pgd_idx]`に挿入する。
`arch/riscv/mm/init.c`
```c
#ifndef __PAGETABLE_PMD_FOLDED
...
#define pgd_next_t		pte_t
#define alloc_pgd_next(__va)	alloc_pte(__va)
#define get_pgd_next_virt(__pa)	get_pte_virt(__pa)
#define create_pgd_next_mapping(__nextp, __va, __pa, __sz, __prot)	\
	create_pte_mapping(__nextp, __va, __pa, __sz, __prot)
#define fixmap_pgd_next		fixmap_pte
#endif
```

`get_pgd_next_virt`は`get_pte_virt`である。

`arch/riscv/mm/init.c`
```c
static pte_t *__init get_pte_virt(phys_addr_t pa)
{
	if (mmu_enabled) {
		clear_fixmap(FIX_PTE);
		return (pte_t *)set_fixmap_offset(FIX_PTE, pa);
	} else {
		return (pte_t *)((uintptr_t)pa);
	}
}
```

### fixmap
fixmapはコンパイルタイムで確定する仮想アドレスである。
それぞれのアドレスは1つのページフレームに対応する。
カーネルはブート時にこれらをポインターとしてこの変わらないアドレスを使用する。
fixmapの定義は`arch/riscv/include/asm/fixmap.h`に存在する。
`arch/riscv/include/asm/fixmap.h`
```c
/*
 * Here we define all the compile-time 'special' virtual addresses.
 * The point is to have a constant address at compile time, but to
 * set the physical address only in the boot process.
 *
 * These 'compile-time allocated' memory buffers are page-sized. Use
 * set_fixmap(idx,phys) to associate physical memory with fixmap indices.
 */
enum fixed_addresses {
	FIX_HOLE,
#define FIX_FDT_SIZE	SZ_1M
	FIX_FDT_END,
	FIX_FDT = FIX_FDT_END + FIX_FDT_SIZE / PAGE_SIZE - 1,
	FIX_PTE,
	FIX_PMD,
	FIX_TEXT_POKE1,
	FIX_TEXT_POKE0,
	FIX_EARLYCON_MEM_BASE,
	__end_of_fixed_addresses
};
```

`arch/riscv/include/asm/pgtable.h`
```c
#ifdef CONFIG_MMU

#define VMALLOC_SIZE     (KERN_VIRT_SIZE >> 1)
#define VMALLOC_END      (PAGE_OFFSET - 1)
#define VMALLOC_START    (PAGE_OFFSET - VMALLOC_SIZE)

#define BPF_JIT_REGION_SIZE	(SZ_128M)
#define BPF_JIT_REGION_START	(PAGE_OFFSET - BPF_JIT_REGION_SIZE)
#define BPF_JIT_REGION_END	(VMALLOC_END)

/*
 * Roughly size the vmemmap space to be large enough to fit enough
 * struct pages to map half the virtual address space. Then
 * position vmemmap directly below the VMALLOC region.
 */
#define VMEMMAP_SHIFT \
	(CONFIG_VA_BITS - PAGE_SHIFT - 1 + STRUCT_PAGE_MAX_SHIFT)
#define VMEMMAP_SIZE	BIT(VMEMMAP_SHIFT)
#define VMEMMAP_END	(VMALLOC_START - 1)
#define VMEMMAP_START	(VMALLOC_START - VMEMMAP_SIZE)

/*
 * Define vmemmap for pfn_to_page & page_to_pfn calls. Needed if kernel
 * is configured with CONFIG_SPARSEMEM_VMEMMAP enabled.
 */
#define vmemmap		((struct page *)VMEMMAP_START)

#define PCI_IO_SIZE      SZ_16M
#define PCI_IO_END       VMEMMAP_START
#define PCI_IO_START     (PCI_IO_END - PCI_IO_SIZE)

#define FIXADDR_TOP      PCI_IO_START
#ifdef CONFIG_64BIT
#define FIXADDR_SIZE     PMD_SIZE
#else
#define FIXADDR_SIZE     PGDIR_SIZE
#endif
#define FIXADDR_START    (FIXADDR_TOP - FIXADDR_SIZE)
```

つまり、FIXMAP領域(`FIXADDR_TOP`)は`PCI_IO_START`(`0x9e000000`)より始まる。
以下にメモリマップを示す。
{{<figure src="./image00.png" >}}
fixmapはindexを用いて、indexに対応するfixmap内のページを割り当てる。
次にindexから仮想アドレスへ割り当てる方法を見ていく。

#### \_\_set\_fixmap
`__set_fixmap`では、`enum fixed_addresses`に対応するindexを物理アドレスへの変換をページテーブルに挿入する。
`fixmap_pte`はfixmap用のページテーブルであり、fixmap領域4096KBをカバーする。
`__fix_to_virt`にて、indexから仮想アドレスを計算する。
次に、仮想アドレスと物理アドレスの変換するPTEを用意し、`fixmap_pte`テーブルを上書きする。

`arch/riscv/mm/init.c`
```c
pgd_t swapper_pg_dir[PTRS_PER_PGD] __page_aligned_bss;
pgd_t trampoline_pg_dir[PTRS_PER_PGD] __page_aligned_bss;
pte_t fixmap_pte[PTRS_PER_PTE] __page_aligned_bss;
static bool mmu_enabled;

#define MAX_EARLY_MAPPING_SIZE	SZ_128M

pgd_t early_pg_dir[PTRS_PER_PGD] __initdata __aligned(PAGE_SIZE);

void __set_fixmap(enum fixed_addresses idx, phys_addr_t phys, pgprot_t prot)
{
	unsigned long addr = __fix_to_virt(idx);
	pte_t *ptep;

	BUG_ON(idx <= FIX_HOLE || idx >= __end_of_fixed_addresses);

	ptep = &fixmap_pte[pte_index(addr)];

	if (pgprot_val(prot))
		set_pte(ptep, pfn_pte(phys >> PAGE_SHIFT, prot));
	else
		pte_clear(&init_mm, addr, ptep);
	local_flush_tlb_page(addr);
}
```
#### \_\_fix\_to\_virtおよび\_\_virt\_to\_fix
これらのマクロで、fixmapのindexから仮想アドレスへの相互変換を行う。
`include/asm-generic/fixmap.h`
```c
#define __fix_to_virt(x)	(FIXADDR_TOP - ((x) << PAGE_SHIFT))
#define __virt_to_fix(x)	((FIXADDR_TOP - ((x)&PAGE_MASK)) >> PAGE_SHIFT)

#ifndef __ASSEMBLY__
/*
 * 'index to address' translation. If anyone tries to use the idx
 * directly without translation, we catch the bug with a NULL-deference
 * kernel oops. Illegal ranges of incoming indices are caught too.
 */
static __always_inline unsigned long fix_to_virt(const unsigned int idx)
{
	BUILD_BUG_ON(idx >= __end_of_fixed_addresses);
	return __fix_to_virt(idx);
}

static inline unsigned long virt_to_fix(const unsigned long vaddr)
{
	BUG_ON(vaddr >= FIXADDR_TOP || vaddr < FIXADDR_START);
	return __virt_to_fix(vaddr);
}
```
`__fix_to_virt`では、`FIXADDR_TOP`から`index<<PAGE_SHIFT`を引くことで求める。
つまり、indexに対応する仮想アドレスのベースアドレスを返す。
逆に、`__virt_to_fix`では、仮想アドレスより、対応するindexを求める。
つまり、ページフレーム番号を求める。

#### clear\_fixmapとset\_fixmap

`include/asm-generic/fixmap.h`
```c
/*
 * Provide some reasonable defaults for page flags.
 * Not all architectures use all of these different types and some
 * architectures use different names.
 */
#ifndef FIXMAP_PAGE_NORMAL
#define FIXMAP_PAGE_NORMAL PAGE_KERNEL
#endif
#if !defined(FIXMAP_PAGE_RO) && defined(PAGE_KERNEL_RO)
#define FIXMAP_PAGE_RO PAGE_KERNEL_RO
#endif
#ifndef FIXMAP_PAGE_NOCACHE
#define FIXMAP_PAGE_NOCACHE PAGE_KERNEL_NOCACHE
#endif
#ifndef FIXMAP_PAGE_IO
#define FIXMAP_PAGE_IO PAGE_KERNEL_IO
#endif
#ifndef FIXMAP_PAGE_CLEAR
#define FIXMAP_PAGE_CLEAR __pgprot(0)
#endif

#ifndef set_fixmap
#define set_fixmap(idx, phys)				\
	__set_fixmap(idx, phys, FIXMAP_PAGE_NORMAL)
#endif

#ifndef clear_fixmap
#define clear_fixmap(idx)			\
	__set_fixmap(idx, 0, FIXMAP_PAGE_CLEAR)
#endif

```

`arch/riscv/include/asm/pgtable.h`
```c
#define _PAGE_KERNEL		(_PAGE_READ \
				| _PAGE_WRITE \
				| _PAGE_PRESENT \
				| _PAGE_ACCESSED \
				| _PAGE_DIRTY)

#define PAGE_KERNEL		__pgprot(_PAGE_KERNEL)
#define PAGE_KERNEL_EXEC	__pgprot(_PAGE_KERNEL | _PAGE_EXEC)

```

`setup_fixmap`と`clear_fixmap`は`__set_fixmap`のラッパーである。
PTEのprotを適切にセットする。

#### set\_fixmap\_offset
`include/asm-generic/fixmap.h`
```c
/* Return a pointer with offset calculated */
#define __set_fixmap_offset(idx, phys, flags)				\
({									\
	unsigned long ________addr;					\
	__set_fixmap(idx, phys, flags);					\
	________addr = fix_to_virt(idx) + ((phys) & (PAGE_SIZE - 1));	\
	________addr;							\
})

#define set_fixmap_offset(idx, phys) \
	__set_fixmap_offset(idx, phys, FIXMAP_PAGE_NORMAL)
```
`set_fixmap_offset`は`__set_fixmap`によりfixmapのセットを行い、
`fix_to_virt`によりindexから対応する仮想アドレスのベースアドレスを求め、それにオフセット(物理アドレス)を足しそれを返す。


### get\_pte\_virt
`arch/riscv/mm/init.c`
```c
static pte_t *__init get_pte_virt(phys_addr_t pa)
{
	if (mmu_enabled) {
		clear_fixmap(FIX_PTE);
		return (pte_t *)set_fixmap_offset(FIX_PTE, pa);
	} else {
		return (pte_t *)((uintptr_t)pa);
	}
}
```
`get_pte_virt`はfixmap領域の`FIX_PTE`を物理アドレス(`pa`)に割り当てる。

## setup\_vm続き
### create\_pgd\_mapping
`arch/riscv/mm/init.c`
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
`memset`にて、pteを0で初期化する。
次に、`pgdp`にエントリーが存在しない場合を見ていく。
`PFN_PHYS`はPFNから物理アドレスを計算する。
`get_pgd_next_virt`にて、`FIX_PTE`を`next_phys`に変換するようにする。
つまり、`create_pgd_mapping`はszが4MBページングの範囲ならPGDのエントリーを作成する。
szが4KBの範囲ならPGDのエントリーをページテーブルを指すようにする。ページテーブルのエントリーが実際の変換を行うようにする(`create_pgd_next_mapping`)。

### create\_pgd\_next\_mapping

`arch/riscv/mm/init.c`
```c
#define create_pgd_next_mapping(__nextp, __va, __pa, __sz, __prot)	\
	create_pte_mapping(__nextp, __va, __pa, __sz, __prot)
...
static void __init create_pte_mapping(pte_t *ptep,
				      uintptr_t va, phys_addr_t pa,
				      phys_addr_t sz, pgprot_t prot)
{
	uintptr_t pte_idx = pte_index(va);

	BUG_ON(sz != PAGE_SIZE);

	if (pte_none(ptep[pte_idx]))
		ptep[pte_idx] = pfn_pte(PFN_DOWN(pa), prot);
}
```
`include/linux/pgtable.h`
```c
static inline unsigned long pte_index(unsigned long address)
{
	return (address >> PAGE_SHIFT) & (PTRS_PER_PTE - 1);
}
```
`pte_index`はアドレスに対応するページテーブル配列のindexを求める。

`arch/riscv/include/asm/pgtable.h`
```c
static inline int pte_none(pte_t pte)
{
	return (pte_val(pte) == 0);
}
```
`pte_none`はPTEがNULLであることを確かめる。
次に、ページテーブルにエントリーを挿入する。

とりあえず、ここまで。
