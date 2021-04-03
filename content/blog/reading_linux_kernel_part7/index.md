---
title: "Reading linux kernel part7"
date: 2021-04-03T21:15:43+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["Linux", "RISC-V"]
draft: true
---

前回で`handle_exception`が読み終わった。今回は、`_start_kernel`から`start_kernel`までを読む。
<!--more-->

`arch/riscv/kernel/head.S`
```asm
	call setup_trap_vector
	/* Restore C environment */
	la tp, init_task
	sw zero, TASK_TI_CPU(tp)
	la sp, init_thread_union + THREAD_SIZE

#ifdef CONFIG_KASAN
	call kasan_early_init
#endif
	/* Start the kernel */
	call soc_early_init
	call parse_dtb
	tail start_kernel
```

次に、`tp`に`init_task`のアドレスをセットする。
前に見たように、
カーネルモードでのトラップ(`sscratch == 0`)は`tp`が`task_struct`へのポインタを保持していることを期待している。
なお、`task_struct`の中に`thread_info`がメンバーとして存在し、その中にカーネルのスタックポインタがメンバーとして存在する。
この`task_struct.thread_info.kernel_sp`を用いてカーネルのスタックへアクセスする。
`init_task`は`.data section`に配置されている。(`arch/riscv/kernel/vmlinux.lds.S`を参照)

次に、`task_struct.thread_info.cpu`を0(おそらく`hartid`、これを実行するのはブートhartのはず。)をセットする。

`init/init_task.c`
```c
/*
 * Set up the first task table, touch at your own risk!. Base=0,
 * limit=0x1fffff (=2MB)
 */
struct task_struct init_task
#ifdef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	__init_task_data
#endif
	__aligned(L1_CACHE_BYTES)
= {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	.thread_info	= INIT_THREAD_INFO(init_task),
	.stack_refcount	= REFCOUNT_INIT(1),
#endif
	.state		= 0,
	.stack		= init_stack,
	.usage		= REFCOUNT_INIT(2),
	.flags		= PF_KTHREAD,
	.prio		= MAX_PRIO - 20,
	.static_prio	= MAX_PRIO - 20,
	.normal_prio	= MAX_PRIO - 20,
	.policy		= SCHED_NORMAL,
	.cpus_ptr	= &init_task.cpus_mask,
	.cpus_mask	= CPU_MASK_ALL,
	.nr_cpus_allowed= NR_CPUS,
	.mm		= NULL,
	.active_mm	= &init_mm,
	.restart_block	= {
		.fn = do_no_restart_syscall,
	},
...	
```
長いので省略する。
これは最初のプロセス(init)のタスク構造体となる。

もう一度`init_task_union`を確認する。
以下のような共用体になっている。
なお、`sp`はこの共用体を`stack`と見たときの最後の要素のアドレスが設定されている。

`include/linux/sched.h`
```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

### \_start\_kernelの続き。
#### soc\_early\_init
`arch/riscv/kernel/head.S`
```asm
	call soc_early_init
	call parse_dtb
	tail start_kernel
```

次に、`soc_early_init`を見ていく。
これは、`parse_dtb`前にSoCのメモリなどを初期化するために使用するものである。とりあえずは使用していない。

`arch/riscv/kernel/soc.c`
```c
/*
 * This is called extremly early, before parse_dtb(), to allow initializing
 * SoC hardware before memory or any device driver initialization.
 */
void __init soc_early_init(void)
{
	void (*early_fn)(const void *fdt);
	const struct of_device_id *s;
	const void *fdt = dtb_early_va;

	for (s = (void *)&__soc_early_init_table_start;
	     (void *)s < (void *)&__soc_early_init_table_end; s++) {
		if (!fdt_node_check_compatible(fdt, 0, s->compatible)) {
			early_fn = s->data;
			early_fn(fdt);
			return;
		}
	}
}
```
`FDT`を読んで、オフセット0のノードの`compatible`プロパティが、`__soc_early_init_table_start`に登録されている`compatible`とマッチするかを調べる。
マッチする場合は、そのSoCに対応する初期化処理を行う。
なお、`FDT`の検査には`libfdt`を使っている？様子。


`scripts/dtc/libfdt/libfdt.h`
```c
/**
 * fdt_node_check_compatible: check a node's compatible property
 * @fdt: pointer to the device tree blob
 * @nodeoffset: offset of a tree node
 * @compatible: string to match against
 *
 *
 * fdt_node_check_compatible() returns 0 if the given node contains a
 * 'compatible' property with the given string as one of its elements,
 * it returns non-zero otherwise, or on error.
 *
 * returns:
 *	0, if the node has a 'compatible' property listing the given string
 *	1, if the node has a 'compatible' property, but it does not list
 *		the given string
 *	-FDT_ERR_NOTFOUND, if the given node has no 'compatible' property
 *	-FDT_ERR_BADOFFSET, if nodeoffset does not refer to a BEGIN_NODE tag
 *	-FDT_ERR_BADMAGIC,
 *	-FDT_ERR_BADVERSION,
 *	-FDT_ERR_BADSTATE,
 *	-FDT_ERR_BADSTRUCTURE, standard meanings
 */
int fdt_node_check_compatible(const void *fdt, int nodeoffset,
			      const char *compatible);
```
`SOC_EARLY_INIT_DECLARE`を使用して`__soc_init_table`へ追加をするのかな？
`arch/riscv/include/asm/soc.h`
```c
#define SOC_EARLY_INIT_DECLARE(name, compat, fn)			\
	static const struct of_device_id __soc_early_init__##name	\
		__used __section(__soc_early_init_table)		\
		 = { .compatible = compat, .data = fn  }
```
これで、Linker Scriptがいい感じに`start`と`end`の間に配置してくれるようである。
`arch/riscv/kernel/vmlinux.lds.S`
```asm
SECTIONS
{
	/* Beginning of code and text segment */
	. = LOAD_OFFSET;
	_start = .;
	HEAD_TEXT_SECTION
	. = ALIGN(PAGE_SIZE);

	__init_begin = .;
	INIT_TEXT_SECTION(PAGE_SIZE)
	. = ALIGN(8);
	__soc_early_init_table : {
		__soc_early_init_table_start = .;
		KEEP(*(__soc_early_init_table))
		__soc_early_init_table_end = .;
	}
```

#### parse\_dtb
次に`parse_dtb`を読んでいく。
`parse_dtb`の目的はブートコマンドラインのオプションのセットとメモリブロック(memblock)の追加である。

`arch/riscv/kernel/setup.c`
```c
void __init parse_dtb(void)
{
	if (early_init_dt_scan(dtb_early_va))
		return;

	pr_err("No DTB passed to the kernel\n");
#ifdef CONFIG_CMDLINE_FORCE
	strlcpy(boot_command_line, CONFIG_CMDLINE, COMMAND_LINE_SIZE);
	pr_info("Forcing kernel command line to: %s\n", boot_command_line);
#endif
}
```

`early_init_dt_scan`にて`dtb_early_va`を用いてDTBを確認し、正しいDTBの構造であればパースし処理を行う。戻り値は`true`となる。
それ以外は`false`を返す。

#### early\_init\_dt\_scan

`driver/of/fdt.c`
```c
bool __init early_init_dt_scan(void *params)
{
	bool status;

	status = early_init_dt_verify(params);
	if (!status)
		return false;

	early_init_dt_scan_nodes();
	return true;
}
```
まずは、`DTB`が正しいか`early_init_dt_verify`にて調べる。

```
bool __init early_init_dt_verify(void *params)
{
	if (!params)
		return false;

	/* check device tree validity */
	if (fdt_check_header(params))
		return false;

	/* Setup flat device-tree pointer */
	initial_boot_params = params;
	of_fdt_crc32 = crc32_be(~0, initial_boot_params,
				fdt_totalsize(initial_boot_params));
	return true;
}
```
これは、DTBのヘッダーを見て正しいかを調べている。
あまり深くは調べないでおく。

次に、`early_init_dt_scan_nodes`を見ていく。
```c
void __init early_init_dt_scan_nodes(void)
{
	int rc = 0;

	/* Retrieve various information from the /chosen node */
	rc = of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
	if (!rc)
		pr_warn("No chosen node found, continuing without\n");

	/* Initialize {size,address}-cells info */
	of_scan_flat_dt(early_init_dt_scan_root, NULL);

	/* Setup memory, calling early_init_dt_add_memory_arch */
	of_scan_flat_dt(early_init_dt_scan_memory, NULL);
}
```
最初に、`of_scan_flat_dt`を用いてルートのchosen nodeからブートコマンドラインパラメタを取り出す。

`of_scan_flat_dt`は以下のとおりである。

```c
/**
 * of_scan_flat_dt - scan flattened tree blob and call callback on each.
 * @it: callback function
 * @data: context data pointer
 *
 * This function is used to scan the flattened device-tree, it is
 * used to extract the memory information at boot before we can
 * unflatten the tree
 */
int __init of_scan_flat_dt(int (*it)(unsigned long node,
				     const char *uname, int depth,
				     void *data),
			   void *data)
{
	const void *blob = initial_boot_params;
	const char *pathp;
	int offset, rc = 0, depth = -1;

	if (!blob)
		return 0;

	for (offset = fdt_next_node(blob, -1, &depth);
	     offset >= 0 && depth >= 0 && !rc;
	     offset = fdt_next_node(blob, offset, &depth)) {

		pathp = fdt_get_name(blob, offset, NULL);
		rc = it(offset, pathp, depth, data);
	}
	return rc;
}
```
この関数では、ノードを順に見ていき(`fdt_next_node`にてノードのオフセットを取得)、コールバック(`it`)を実行する。
なお、取り出したデータは`data`に格納される。
```c
int fdt_next_node(const void *fdt, int offset, int *depth)
{
	int nextoffset = 0;
	uint32_t tag;

	if (offset >= 0)
		if ((nextoffset = fdt_check_node_offset_(fdt, offset)) < 0)
			return nextoffset;

	do {
		offset = nextoffset;
		tag = fdt_next_tag(fdt, offset, &nextoffset);

		switch (tag) {
		case FDT_PROP:
		case FDT_NOP:
			break;

		case FDT_BEGIN_NODE:
			if (depth)
				(*depth)++;
			break;

		case FDT_END_NODE:
			if (depth && ((--(*depth)) < 0))
				return nextoffset;
			break;

		case FDT_END:
			if ((nextoffset >= 0)
			    || ((nextoffset == -FDT_ERR_TRUNCATED) && !depth))
				return -FDT_ERR_NOTFOUND;
			else
				return nextoffset;
		}
	} while (tag != FDT_BEGIN_NODE);

	return offset;
}
```

`early_init_dt_scan_nodes`再掲。
```c
void __init early_init_dt_scan_nodes(void)
{
	int rc = 0;

	/* Retrieve various information from the /chosen node */
	rc = of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
	if (!rc)
		pr_warn("No chosen node found, continuing without\n");

	/* Initialize {size,address}-cells info */
	of_scan_flat_dt(early_init_dt_scan_root, NULL);

	/* Setup memory, calling early_init_dt_add_memory_arch */
	of_scan_flat_dt(early_init_dt_scan_memory, NULL);
}
```

まずは、`early_init_dt_scan_chosen`関数

`drivers/of/fdt.c`
```c
int __init early_init_dt_scan_chosen(unsigned long node, const char *uname,
				     int depth, void *data)
{
	int l;
	const char *p;
	const void *rng_seed;

	pr_debug("search \"chosen\", depth: %d, uname: %s\n", depth, uname);

	if (depth != 1 || !data ||
	    (strcmp(uname, "chosen") != 0 && strcmp(uname, "chosen@0") != 0))
		return 0;

	early_init_dt_check_for_initrd(node);

	/* Retrieve command line */
	p = of_get_flat_dt_prop(node, "bootargs", &l);
	if (p != NULL && l > 0)
		strlcpy(data, p, min(l, COMMAND_LINE_SIZE));

	/*
	 * CONFIG_CMDLINE is meant to be a default in case nothing else
	 * managed to set the command line, unless CONFIG_CMDLINE_FORCE
	 * is set in which case we override whatever was found earlier.
	 */
#ifdef CONFIG_CMDLINE
#if defined(CONFIG_CMDLINE_EXTEND)
	strlcat(data, " ", COMMAND_LINE_SIZE);
	strlcat(data, CONFIG_CMDLINE, COMMAND_LINE_SIZE);
#elif defined(CONFIG_CMDLINE_FORCE)
	strlcpy(data, CONFIG_CMDLINE, COMMAND_LINE_SIZE);
#else
	/* No arguments from boot loader, use kernel's  cmdl*/
	if (!((char *)data)[0])
		strlcpy(data, CONFIG_CMDLINE, COMMAND_LINE_SIZE);
#endif
#endif /* CONFIG_CMDLINE */

	pr_debug("Command line is: %s\n", (char *)data);

	rng_seed = of_get_flat_dt_prop(node, "rng-seed", &l);
	if (rng_seed && l > 0) {
		add_bootloader_randomness(rng_seed, l);

		/* try to clear seed so it won't be found. */
		fdt_nop_property(initial_boot_params, node, "rng-seed");

		/* update CRC check value */
		of_fdt_crc32 = crc32_be(~0, initial_boot_params,
				fdt_totalsize(initial_boot_params));
	}

	/* break now */
	return 1;
}
```

`early_init_dt_scan_root`関数
`drivers/of/fdt.c`
```c
/**
 * early_init_dt_scan_root - fetch the top level address and size cells
 */
int __init early_init_dt_scan_root(unsigned long node, const char *uname,
				   int depth, void *data)
{
	const __be32 *prop;

	if (depth != 0)
		return 0;

	dt_root_size_cells = OF_ROOT_NODE_SIZE_CELLS_DEFAULT;
	dt_root_addr_cells = OF_ROOT_NODE_ADDR_CELLS_DEFAULT;

	prop = of_get_flat_dt_prop(node, "#size-cells", NULL);
	if (prop)
		dt_root_size_cells = be32_to_cpup(prop);
	pr_debug("dt_root_size_cells = %x\n", dt_root_size_cells);

	prop = of_get_flat_dt_prop(node, "#address-cells", NULL);
	if (prop)
		dt_root_addr_cells = be32_to_cpup(prop);
	pr_debug("dt_root_addr_cells = %x\n", dt_root_addr_cells);

	/* break now */
	return 1;
}
```
この関数で、`root`ノードの`size_cells`と`addr_cells`を読み出す。

`early_init_dt_scan_memory`関数
`drivers/of/fdt.c`
```c
/**
 * early_init_dt_scan_memory - Look for and parse memory nodes
 */
int __init early_init_dt_scan_memory(unsigned long node, const char *uname,
				     int depth, void *data)
{
	const char *type = of_get_flat_dt_prop(node, "device_type", NULL);
	const __be32 *reg, *endp;
	int l;
	bool hotpluggable;

	/* We are scanning "memory" nodes only */
	if (type == NULL || strcmp(type, "memory") != 0)
		return 0;

	reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);
	if (reg == NULL)
		reg = of_get_flat_dt_prop(node, "reg", &l);
	if (reg == NULL)
		return 0;

	endp = reg + (l / sizeof(__be32));
	hotpluggable = of_get_flat_dt_prop(node, "hotpluggable", NULL);

	pr_debug("memory scan node %s, reg size %d,\n", uname, l);

	while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
		u64 base, size;

		base = dt_mem_next_cell(dt_root_addr_cells, &reg);
		size = dt_mem_next_cell(dt_root_size_cells, &reg);

		if (size == 0)
			continue;
		pr_debug(" - %llx ,  %llx\n", (unsigned long long)base,
		    (unsigned long long)size);

		early_init_dt_add_memory_arch(base, size);

		if (!hotpluggable)
			continue;

		if (early_init_dt_mark_hotplug_memory_arch(base, size))
			pr_warn("failed to mark hotplug range 0x%llx - 0x%llx\n",
				base, base + size);
	}

	return 0;
}
```
この関数では、`memory`プロパティを読み、`memblock`へメモリ領域を追加する。

`drivers/of/fdt.c`
```c
void __init __weak early_init_dt_add_memory_arch(u64 base, u64 size)
{
	const u64 phys_offset = MIN_MEMBLOCK_ADDR;

	if (size < PAGE_SIZE - (base & ~PAGE_MASK)) {
		pr_warn("Ignoring memory block 0x%llx - 0x%llx\n",
			base, base + size);
		return;
	}

	if (!PAGE_ALIGNED(base)) {
		size -= PAGE_SIZE - (base & ~PAGE_MASK);
		base = PAGE_ALIGN(base);
	}
	size &= PAGE_MASK;

	if (base > MAX_MEMBLOCK_ADDR) {
		pr_warn("Ignoring memory block 0x%llx - 0x%llx\n",
			base, base + size);
		return;
	}

	if (base + size - 1 > MAX_MEMBLOCK_ADDR) {
		pr_warn("Ignoring memory range 0x%llx - 0x%llx\n",
			((u64)MAX_MEMBLOCK_ADDR) + 1, base + size);
		size = MAX_MEMBLOCK_ADDR - base + 1;
	}

	if (base + size < phys_offset) {
		pr_warn("Ignoring memory block 0x%llx - 0x%llx\n",
			base, base + size);
		return;
	}
	if (base < phys_offset) {
		pr_warn("Ignoring memory range 0x%llx - 0x%llx\n",
			base, phys_offset);
		size -= phys_offset - base;
		base = phys_offset;
	}
	memblock_add(base, size);
}
```
`memblock_add`関数で`memblock`へ追加する。
これで、Linuxからメモリを使用することができるようになる。
なお、`soc_early_init`にてメモリコントローラは初期化済みであることが必須である。

最後に`start_kernel`を実行する。
次からは`start_kernel`本体を読んでいく。
なお、`FDT`のパースについては詳しくは読んでいないので、必要に応じて読んでいく。
