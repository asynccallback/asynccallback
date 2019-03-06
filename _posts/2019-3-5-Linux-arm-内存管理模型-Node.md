---
layout: post
title: "Linux-arm-内存管理模型-Node"
subtitle: 详细分析Node管理模型中的代码，从pg_data_t->zone->page
author: "404"
header-style: text
tags:
  - Linux
  - arm
  - Memory
  - pg_data_t
---

> 本篇文章为原创，转载请注明

前面我们分析了系统如何形成`memblock`，系统中生成Memblock管理体系后，内存可以被使用，但此时内存以大块形式存在，不便于精细化使用，故我们采取把内存分为node，node对于相应cpu，node再分为zone，zone再细化为page，我们最终以page形式来使用内存，现在接受如何实现这些，本章中大部分都是代码，会比较枯燥，但认真分析后还是挺有意思的。

系统从`paging_init`->`bootmem_init`，开始进行下一步的内存规划管理：
```c
void __init bootmem_init(void)
{
	unsigned long min, max_low, max_high;

	memblock_allow_resize();
	max_low = max_high = 0;

  /** min相当于Memblock中内存开始处
  **  maxlow是内存低址最高处，即892M处，
  **  max_high即内存地址最高处
  **  这里说的内存地址指的是操作系统的地址，即分配给操作系统使用的地址，
  **  可能是1G，可能是2G，具体看配置怎么分配
  **/
	find_limits(&min, &max_low, &max_high);

	early_memtest((phys_addr_t)min << PAGE_SHIFT,
		      (phys_addr_t)max_low << PAGE_SHIFT);

	/*
	 * Sparsemem tries to allocate bootmem in memory_present(),
	 * so must be done after the fixed reservations
	 */
	arm_memory_present();

	/*
	 * sparse_init() needs the bootmem allocator up and running.
	 */
	sparse_init();

	/*
	 * Now free the memory - free_area_init_node needs
	 * the sparse mem_map arrays initialized by sparse_init()
	 * for memmap_init_zone(), otherwise all PFNs are invalid.
   * 这里进去找地址空间的一些空洞，空洞主要是分配不同功能内存后，防止越界操作
   * 用空洞来进行预防
	 */
	zone_sizes_init(min, max_low, max_high);

	/*
	 * This doesn't seem to be used by the Linux memory manager any
	 * more, but is used by ll_rw_block.  If we can get rid of it, we
	 * also get rid of some of the stuff above as well.
	 */
	min_low_pfn = min;
	max_low_pfn = max_low;
	max_pfn = max_high;
}
```

上面分析中主要注重`find_limits`和`zone_sizes_init`，具体见上面分析，接下来进入`zone_sizes_init`,这里面代码分析没什么很重要的部分，直接跳到最后`free_area_init_node`。再`free_area_init_node`中，我们主要关注`alloc_node_mem_map`和`free_area_init_core`。`alloc_node_mem_map`是填充`pglist_data_t.node_mem_map`：
```c
static void __ref alloc_node_mem_map(struct pglist_data *pgdat)
{
	unsigned long __maybe_unused start = 0;
	unsigned long __maybe_unused offset = 0;

	/* Skip empty nodes */
	if (!pgdat->node_spanned_pages)
		return;

	start = pgdat->node_start_pfn & ~(MAX_ORDER_NR_PAGES - 1);
	offset = pgdat->node_start_pfn - start;
	/* ia64 gets its own node_mem_map, before this, without bootmem */
	if (!pgdat->node_mem_map) {
		unsigned long size, end;
		struct page *map;

		/*
		 * The zone's endpoints aren't required to be MAX_ORDER
		 * aligned but the node_mem_map endpoints must be in order
		 * for the buddy allocator to function correctly.
		 */
		end = pgdat_end_pfn(pgdat);
		end = ALIGN(end, MAX_ORDER_NR_PAGES);
		size =  (end - start) * sizeof(struct page);
		map = memblock_alloc_node_nopanic(size, pgdat->node_id);
		pgdat->node_mem_map = map + offset;
	}
	pr_debug("%s: node %d, pgdat %08lx, node_mem_map %08lx\n",
				__func__, pgdat->node_id, (unsigned long)pgdat,
				(unsigned long)pgdat->node_mem_map);
#ifndef CONFIG_NEED_MULTIPLE_NODES
	/*
	 * With no DISCONTIG, the global mem_map is just set as node 0's
	 */
	if (pgdat == NODE_DATA(0)) {
		mem_map = NODE_DATA(0)->node_mem_map;
#if defined(CONFIG_HAVE_MEMBLOCK_NODE_MAP) || defined(CONFIG_FLATMEM)
		if (page_to_pfn(mem_map) != pgdat->node_start_pfn)
			mem_map -= offset;
#endif /* CONFIG_HAVE_MEMBLOCK_NODE_MAP */
	}
```
我们知道，这个bootmem内存管理系统也只是暂时使用，最后还是有Buddy伙伴系统来交接，所以现在`start = pgdat->node_start_pfn & ~(MAX_ORDER_NR_PAGES - 1);`，用来跟伙伴系统最大内存对齐，方便以后使用。接着调用`memblock_alloc_node_nopanic`再memblock中分配内存，此时还是memblock内存管理系统起作用，其后调用`memblock_alloc_try_nid_nopanic(size, SMP_CACHE_BYTES, MEMBLOCK_LOW_LIMIT,MEMBLOCK_ALLOC_ACCESSIBLE, nid)`:
```c
/**
 * memblock_alloc_try_nid_nopanic - allocate boot memory block
 * @size: size of memory block to be allocated in bytes
 * @align: alignment of the region and block's size
 * @min_addr: the lower bound of the memory region from where the allocation
 *	  is preferred (phys address)
 * @max_addr: the upper bound of the memory region from where the allocation
 *	      is preferred (phys address), or %MEMBLOCK_ALLOC_ACCESSIBLE to
 *	      allocate only from memory limited by memblock.current_limit value
 * @nid: nid of the free area to find, %NUMA_NO_NODE for any node
 *
 * Public function, provides additional debug information (including caller
 * info), if enabled. This function zeroes the allocated memory.
 *
 * Return:
 * Virtual address of allocated memory block on success, NULL on failure.
 */
void * __init memblock_alloc_try_nid_nopanic(
				phys_addr_t size, phys_addr_t align,
				phys_addr_t min_addr, phys_addr_t max_addr,
				int nid)
{
	void *ptr;

	memblock_dbg("%s: %llu bytes align=0x%llx nid=%d from=%pa max_addr=%pa %pF\n",
		     __func__, (u64)size, (u64)align, nid, &min_addr,
		     &max_addr, (void *)_RET_IP_);

	ptr = memblock_alloc_internal(size, align,
					   min_addr, max_addr, nid);
	if (ptr)
		memset(ptr, 0, size);
	return ptr;
}


/**
 * memblock_alloc_internal - allocate boot memory block
 * @size: size of memory block to be allocated in bytes
 * @align: alignment of the region and block's size
 * @min_addr: the lower bound of the memory region to allocate (phys address)
 * @max_addr: the upper bound of the memory region to allocate (phys address)
 * @nid: nid of the free area to find, %NUMA_NO_NODE for any node
 *
 * The @min_addr limit is dropped if it can not be satisfied and the allocation
 * will fall back to memory below @min_addr. Also, allocation may fall back
 * to any node in the system if the specified node can not
 * hold the requested memory.
 *
 * The allocation is performed from memory region limited by
 * memblock.current_limit if @max_addr == %MEMBLOCK_ALLOC_ACCESSIBLE.
 *
 * The phys address of allocated boot memory block is converted to virtual and
 * allocated memory is reset to 0.
 *
 * In addition, function sets the min_count to 0 using kmemleak_alloc for
 * allocated boot memory block, so that it is never reported as leaks.
 *
 * Return:
 * Virtual address of allocated memory block on success, NULL on failure.
 */
static void * __init memblock_alloc_internal(
				phys_addr_t size, phys_addr_t align,
				phys_addr_t min_addr, phys_addr_t max_addr,
				int nid)
{
	phys_addr_t alloc;
	void *ptr;
	enum memblock_flags flags = choose_memblock_flags();

	if (WARN_ONCE(nid == MAX_NUMNODES, "Usage of MAX_NUMNODES is deprecated. Use NUMA_NO_NODE instead\n"))
		nid = NUMA_NO_NODE;

	/*
	 * Detect any accidental use of these APIs after slab is ready, as at
	 * this moment memblock may be deinitialized already and its
	 * internal data may be destroyed (after execution of memblock_free_all)
	 */
	if (WARN_ON_ONCE(slab_is_available()))
		return kzalloc_node(size, GFP_NOWAIT, nid);

	if (!align) {
		dump_stack();
		align = SMP_CACHE_BYTES;
	}

	if (max_addr > memblock.current_limit)
		max_addr = memblock.current_limit;
again:
	alloc = memblock_find_in_range_node(size, align, min_addr, max_addr,
					    nid, flags);
	if (alloc && !memblock_reserve(alloc, size))
		goto done;

	if (nid != NUMA_NO_NODE) {
		alloc = memblock_find_in_range_node(size, align, min_addr,
						    max_addr, NUMA_NO_NODE,
						    flags);
		if (alloc && !memblock_reserve(alloc, size))
			goto done;
	}

	if (min_addr) {
		min_addr = 0;
		goto again;
	}

	if (flags & MEMBLOCK_MIRROR) {
		flags &= ~MEMBLOCK_MIRROR;
		pr_warn("Could not allocate %pap bytes of mirrored memory\n",
			&size);
		goto again;
	}

	return NULL;
done:
	ptr = phys_to_virt(alloc);

	/* Skip kmemleak for kasan_init() due to high volume. */
	if (max_addr != MEMBLOCK_ALLOC_KASAN)
		/*
		 * The min_count is set to 0 so that bootmem allocated
		 * blocks are never reported as leaks. This is because many
		 * of these blocks are only referred via the physical
		 * address which is not looked up by kmemleak.
		 */
		kmemleak_alloc(ptr, size, 0, 0);

	return ptr;
}
```

![avatar](/img/in-post/Linux/201930201002.png)
