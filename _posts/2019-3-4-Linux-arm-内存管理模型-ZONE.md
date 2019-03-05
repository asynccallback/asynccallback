---
layout: post
title: "Linux-arm-内存管理模型-ZONE"
subtitle: ZONEk数据结构及管理'
author: "404"
header-style: text
tags:
  - Linux
  - arm
  - Memory
  - ZONE
---

> 本篇文章为原创，转载请注明

在内存管理模型中，我们已经写过一篇关于Zone管理的概述，这篇详细介绍相关数据结构和函数部分，首先放一个整体结构图：
![avatar](/img/in-post/Linux/201930201002.png)

我们首先来看pg_data_t的数据结构：
```c
/*
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout. On UMA machines there is a single pglist_data which
 * describes the whole memory.
 *
 * Memory statistics and page replacement data structures are maintained on a
 * per-zone basis.
 */
struct bootmem_data;
typedef struct pglist_data {
	//一个结构数组，包含了结点中各内存域的数据结构zone
	struct zone node_zones[MAX_NR_ZONES];
	//指定了备用结点机器内存域的列表，以便在当前结点没有可用空间时，在备用结点分配内存
	struct zonelist node_zonelists[MAX_ZONELISTS];
	//内存域的个数        
	int nr_zones;
#ifdef CONFIG_FLAT_NODE_MEM_MAP	/* means !SPARSEMEM */
	//指向结点的第一个页框的页结构，该页结构位于全局mem_map中某个位置
	struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#ifndef CONFIG_NO_BOOTMEM
	//启动内存分配器
	struct bootmem_data *bdata;
#endif
#ifdef CONFIG_MEMORY_HOTPLUG
	/*
	 * Must be held any time you expect node_start_pfn, node_present_pages
	 * or node_spanned_pages stay constant.  Holding this will also
	 * guarantee that any pfn_valid() stays that way.
	 *
	 * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
	 * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG.
	 *
	 * Nests above zone->lock and zone->span_seqlock
	 */
	spinlock_t node_size_lock;
#endif
	//结点起始页框
	unsigned long node_start_pfn;
	//结点总页框数（不包含洞）
	unsigned long node_present_pages; /* total number of physical pages */
	//结点总页框数（包含洞）
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	//结点id
	int node_id;
	//交换守护进程的等待列表
	wait_queue_head_t kswapd_wait;
	//本结点交换守护进程
	wait_queue_head_t pfmemalloc_wait;
	struct task_struct *kswapd;	/* Protected by
					   mem_hotplug_begin/end() */
	int kswapd_order;
	enum zone_type kswapd_classzone_idx;

	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */

#ifdef CONFIG_COMPACTION
	int kcompactd_max_order;
	enum zone_type kcompactd_classzone_idx;
	wait_queue_head_t kcompactd_wait;
	struct task_struct *kcompactd;
#endif
#ifdef CONFIG_NUMA_BALANCING
	/* Lock serializing the migrate rate limiting window */
	spinlock_t numabalancing_migrate_lock;

	/* Rate limiting time interval */
	unsigned long numabalancing_migrate_next_window;

	/* Number of pages migrated during the rate limiting time interval */
	unsigned long numabalancing_migrate_nr_pages;
#endif
	/*
	 * This is a per-node reserve of pages that are not available
	 * to userspace allocations.
	 */
	unsigned long		totalreserve_pages;

#ifdef CONFIG_NUMA
	/*
	 * zone reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* Write-intensive fields used by page reclaim */
	ZONE_PADDING(_pad1_)
	spinlock_t		lru_lock;

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
	/*
	 * If memory initialisation on large machines is deferred then this
	 * is the first PFN that needs to be initialised.
	 */
	unsigned long first_deferred_pfn;
	unsigned long static_init_size;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	spinlock_t split_queue_lock;
	struct list_head split_queue;
	unsigned long split_queue_len;
#endif

	/* Fields commonly accessed by the page reclaim scanner */
	struct lruvec		lruvec;

	/*
	 * The target ratio of ACTIVE_ANON to INACTIVE_ANON pages on
	 * this node's LRU.  Maintained by the pageout code.
	 */
	unsigned int inactive_ratio;

	unsigned long		flags;

	ZONE_PADDING(_pad2_)

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
```

然后看下他管理的Zone：
```c
struct zone {
	/* Read-mostly fields */
	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long watermark[NR_WMARK];
	unsigned long nr_reserved_highatomic;
	long lowmem_reserve[MAX_NR_ZONES];
#ifdef CONFIG_NUMA
	int node;
#endif
	struct pglist_data	*zone_pgdat;
	struct per_cpu_pageset __percpu *pageset;
#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */
	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;
	unsigned long		managed_pages;
	unsigned long		spanned_pages;
	unsigned long		present_pages;
	const char		*name;
#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Number of isolated pageblock. It is used to solve incorrect
	 * freepage counting problem due to racy retrieving migratetype
	 * of pageblock. Protected by zone->lock.
	 */
	unsigned long		nr_isolate_pageblock;
#endif
#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif
	int initialized;
	/* Write-intensive fields used from the page allocator */
	ZONE_PADDING(_pad1_)
	/* free areas of different sizes */
	struct free_area	free_area[MAX_ORDER];
	/* zone flags, see below */
	unsigned long		flags;
	/* Primarily protects free_area */
	spinlock_t		lock;
	/* Write-intensive fields used by compaction and vmstats. */
	ZONE_PADDING(_pad2_)
	unsigned long percpu_drift_mark;
#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* pfn where compaction free scanner should start */
	unsigned long		compact_cached_free_pfn;
	/* pfn where async and sync compaction migration scanner should start */
	unsigned long		compact_cached_migrate_pfn[2];
#endif
#ifdef CONFIG_COMPACTION
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
	int			compact_order_failed;
#endif
#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* Set to true when the PG_migrate_skip bits should be cleared */
	bool			compact_blockskip_flush;
#endif
	bool			contiguous;
	ZONE_PADDING(_pad3_)
	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	atomic_long_t		vm_numa_stat[NR_VM_NUMA_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;

```

解析其中重要的字段：
- watermark：每个zone在系统启动时会计算出3个水为值，分别是WMARK_MIN，WMARK_LOW和WMARK_HIGH水位，这在页面分配器和kswapd页面回收中会用到。
- lowmem_reserve：zone中预留的内存。
- zone_pgdat：指向内存节点。
- pageset：用于维护Pre-CPU上的一系列页面，以减少自旋锁的争用。
- zone_start_pfn：zone中开始的页帧号。
- managed_pages：zone中被伙伴系统管理的页面数量。
- spanned_pages：zone包含的页面数量。
- present_pages：zone里实际管理的页面数量。对一些体系结构来说，其值和spanned_pages相等。
- free_area：管理空闲区域的数组，包含管理链表等。
- lock：并行访问时用于对zone保护的自旋锁。
- lru_lock：用于对zone中LRU链表进行访问时进行保护的自旋锁。
- lruvec：LRU链表集合。
- vm_stat：zone计数。

Zone其初始化方向是`bootmem_init`->`zone_list_init`->`free_area_init_node`,对zone进行初始化，其中用于伙伴管理系统的free_area没有初始化，代码不是很复杂，可以自行了解。其中我就有个问题，期间计算size部分用到`arch_zone_lowest_possible_pfn`，但是在arm32体系中，我并没有找到初始化他的地方，arm64下倒是有初始化部分，这个还得了解下。

接下来便是mem_map的初始化：
```c
static void __init_refok alloc_node_mem_map(struct pglist_data *pgdat)
{
	/* Skip empty nodes */
	if (!pgdat->node_spanned_pages)//如果内存结点没有没存页，直接返回
		return;

#ifdef CONFIG_FLAT_NODE_MEM_MAP
	/* ia64 gets its own node_mem_map, before this, without bootmem */
	if (!pgdat->node_mem_map) {//如果还没有为结点分配mem_map，则需要为结点分配mem_map
		unsigned long size, start, end;
		struct page *map;

		/*
		 * The zone's endpoints aren't required to be MAX_ORDER
		 * aligned but the node_mem_map endpoints must be in order
		 * for the buddy allocator to function correctly.
		 */
		start = pgdat->node_start_pfn & ~(MAX_ORDER_NR_PAGES - 1);//确定起点，以MAX_ORDER_NR_PAGES的大小对齐
		end = pgdat->node_start_pfn + pgdat->node_spanned_pages;//计算结束点
		end = ALIGN(end, MAX_ORDER_NR_PAGES);//以MAX_ORDER_NR_PAGES对齐，与上面的功能一致，将内存映射对齐到伙伴系统的最大分配阶
		size =  (end - start) * sizeof(struct page);//计算所需内存的大小
		map = alloc_remap(pgdat->node_id, size);//为内存映射分配内存
		if (!map)//如果分配不成功，则使用普通的自举内存分配器进行分配
			map = alloc_bootmem_node(pgdat, size);
		pgdat->node_mem_map = map + (pgdat->node_start_pfn - start);
	}
#ifndef CONFIG_NEED_MULTIPLE_NODES
	/*
	 * With no DISCONTIG, the global mem_map is just set as node 0's
	 */
	if (pgdat == NODE_DATA(0)) {
		mem_map = NODE_DATA(0)->node_mem_map;
#ifdef CONFIG_ARCH_POPULATES_NODE_MAP
		if (page_to_pfn(mem_map) != pgdat->node_start_pfn)
			mem_map -= (pgdat->node_start_pfn - ARCH_PFN_OFFSET);
#endif /* CONFIG_ARCH_POPULATES_NODE_MAP */
	}
#endif
#endif /* CONFIG_FLAT_NODE_MEM_MAP */
}
```
