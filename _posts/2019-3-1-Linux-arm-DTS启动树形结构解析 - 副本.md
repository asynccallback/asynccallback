---
layout: post
title: "Linux-arm-DTS启动树形结构解析"
subtitle: '从二进制文件中解析出DTS树形结构'
author: "404"
header-style: text
tags:
  - Linux
  - arm
  - DTS
---

>此篇文章为原创，转载请注明出处。

由[Linux-DTS解析过程](../../../../2019/02/19/Linux-DTS解析过程/)中我们知道了DTS的头结构体，由四部分组成：`fdt_header`、`memory reserve map`、`device-tree structure`、`device-tree strings`。如下：
![avatar](/img/in-post/Linux/2019219003.png)

简化版本结构如下：
![avatar](/img/in-post/Linux/201930101001.png)

#### 保留内存（memory reserve map）

这段保存的是一个保留内存映射列表，每个表由一对64位的物理地址和大小组成。其数据结构如下：
```c
struct fdt_reserve_entry {
    fdt64_t address;
    fdt64_t size;
};
```

#### device-tree structure&strings

由于我们以前就知道，`ftd_header`就是为`device-tree structure&strings`服务的，所以我们重点来解析`device-tree structure&strings`结构，其结构信息如下：
![avatar](/img/in-post/Linux/201930101002.png)

现在我们看看操作系统如何解析出这些结构，以`vexpress-v2p-ca9.dts`为例，此文件可以再Linux内核的dts文件中查找到。

在Linux中对DTS树（此时我们只判断对`Memory`节点的解析）调用过程如下：`setup_arch()`->`setup_machine_fdt()`->`early_init_dt_scan_nodes()`->`of_scan_flat_dt(early_init_dt_scan_memory, NULL)`,重点分析`of_scan_flat_dt()`、`of_scan_flat_dt(early_init_dt_scan_memory, NULL)`。`of_scan_flat_dt`函数如下：
```c
/**
 * of_scan_flat_dt - scan flattened tree blob and call callback on each.
 * @it: callback function
 * @data: context data pointer
 *
 * This function is used to scan the flattened device-tree, it is
 * used to extract the memory information at boot before we can
 * unflatten the tree
 **/
int __init of_scan_flat_dt(int (*it)(unsigned long node,
				     const char *uname, int depth,
				     void *data),
			   void *data)
{
	const void * blob = initial_boot_params;
	const char * pathp;
	int offset, rc = 0, depth = -1;
	if (!blob)
		return 0;
	for (offset = fdt_next_node(blob, -1, &depth);
	     offset >= 0 && depth >= 0 && !rc;
	     offset = fdt_next_node(blob, offset, &depth)) {
		pathp = fdt_get_name(blob, offset, NULL);
		if (*pathp == '/')
			pathp = kbasename(pathp);
		rc = it(offset, pathp, depth, data);
	}
	return rc;
}
```
此时的`const void * blob = initial_boot_params;`我们就把`initial_boot_params`用黑盒封装下，就当成DTS树形结构，因为具体`initial_boot_params`赋值过程我暂时没有找到，这也并不影响我们分析，接下来调用`fdt_next_node`：
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
				(* depth)++;
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
我们采用由地到上的具体调用过程来分析，接下来调用`fdt_next_tag()`:
```c
uint32_t fdt_next_tag(const void *fdt, int startoffset, int *nextoffset)
{
	const fdt32_t * tagp, * lenp;
	uint32_t tag;
	int offset = startoffset;
	const char * p;
	* nextoffset = -FDT_ERR_TRUNCATED;
	tagp = fdt_offset_ptr(fdt, offset, FDT_TAGSIZE);
	if (!tagp)
		return FDT_END; /* premature end * /
	tag = fdt32_to_cpu(* tagp);
	offset += FDT_TAGSIZE;
	 nextoffset = -FDT_ERR_BADSTRUCTURE;
	switch (tag) {
	case FDT_BEGIN_NODE:
		/* skip name * /
		do {
			p = fdt_offset_ptr(fdt, offset++, 1);
		} while (p && (*p != '\0'));
		if (!p)
			return FDT_END; /* premature end * /
		break;
	case FDT_PROP:
		lenp = fdt_offset_ptr(fdt, offset, sizeof(*lenp));
		if (!lenp)
			return FDT_END; /* premature end * /
		/ * skip-name offset, length and value * /
		offset += sizeof(struct fdt_property) - FDT_TAGSIZE
			+ fdt32_to_cpu(*lenp);
		if (fdt_version(fdt) < 0x10 && fdt32_to_cpu(*lenp) >= 8 &&
		    ((offset - fdt32_to_cpu(*lenp)) % 8) != 0)
			offset += 4;
		break;
	case FDT_END:
	case FDT_END_NODE:
	case FDT_NOP:
		break;
	default:
		return FDT_END;
	}
	if (!fdt_offset_ptr(fdt, startoffset, offset - startoffset))
		return FDT_END; /* premature end * /
	* nextoffset = FDT_TAGALIGN(offset);
	return tag;
}
```
接下来调用`fdt_offset_ptr`:
```c
const void *fdt_offset_ptr(const void *fdt, int offset, unsigned int len)
{
	unsigned absoffset = offset + fdt_off_dt_struct(fdt);
	if ((absoffset < offset)
	    || ((absoffset + len) < absoffset)
	    || (absoffset + len) > fdt_totalsize(fdt))
		return NULL;
	if (fdt_version(fdt) >= 0x11)
		if (((offset + len) < offset)
		    || ((offset + len) > fdt_size_dt_struct(fdt)))
			return NULL;
	return fdt_offset_ptr_(fdt, offset);
}
```
好了，就从这步来分析，我们忽略掉边界条件的分析，在此，我们再重复下ftd头文件的格式：
```c
struct fdt_header {
	fdt32_t magic;			       / * magic word FDT_MAGIC * /
	fdt32_t totalsize;		     / * total size of DT block * /
	fdt32_t off_dt_struct;		 / * offset to structure * /
	fdt32_t off_dt_strings;		 / * offset to strings * /
	fdt32_t off_mem_rsvmap;		 / * offset to memory reserve map * /
	fdt32_t version;		         / * format version * /
	fdt32_t last_comp_version;	 / * last compatible version * /

	/* version 2 fields below * /
	fdt32_t boot_cpuid_phys;	 / * Which physical CPU id we're booting on * /
	/* version 3 fields below * /
	fdt32_t size_dt_strings;	 / * size of the strings block * /

	/* version 17 fields below * /
	fdt32_t size_dt_struct;		 / * size of the structure block * /
};
```
`fdt_off_dt_struct（ftd）`是用来得到`ftd->off_dt_struct`的值，`fdt_offset_ptr_(fdt, offset)`中与基地址和offset相加地到相对应的dt_struct结构，arm linux中，为什么要用`fdt_off_dt_struct（ftd）`来得到所需地址，我在分析过程中觉得完全可以用`ftd->off_dt_struct`来得到同样的东西，为何要多此一举？我现在还没想清楚，想清后我会加上解释在下面，如果有能理解的请留言我。我们总结这一步就是得到具体的`dt_struct`在数据结构中的位置。

下面我们以具体的vexress设备树编译文件来分析：
![avatar](/img/in-post/Linux/201930101003.png)

其中红色部分为`fdt_header`，黄色部分为`memory reserve map`,此时`dt_struct`的开始位置刚好等于fdt(假设为0x00)+fdt_header.off_dt_struct(0x38)。接下来的部分便是操作系统来解析`dt_struct`。我们可以接下来的4个字节为`0x01`，Linux对相应字节的对应为：
![avatar](/img/in-post/Linux/201930101004.png)
同时`dt_struct`中关于`FDT_BEGIN_NODE`、`FDT_PROP`部分结构体如下：
```c
struct fdt_node_header {
	fdt32_t tag;
	char name[0];
};

struct fdt_property {
	fdt32_t tag;
	fdt32_t len;
	fdt32_t nameoff;
	char data[0];
};
```
结合代码，当检测到是`FDT_BEGIN_NODE`后，会调用`early_init_dt_scan_memory()`,在此节点的属性中寻找有没有`device_type=\'memory\'`，如果没有，则继续扫描下一个节点，期间根据`FDT_PROP`的结构组成，跳过`FDT_PROP`；如果有`device_type=\'memory\'`，则会把这些加入到`memblock`的子系统中，包含此物理节点的起始位置和大小。不同的`FDT_BEGIN_NODE`中含有`device_type=\'memory\'`部分，构成了`memblock`的数组。
