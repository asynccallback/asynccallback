---
layout: post
title: "Linux-DTS解析过程"
subtitle: 'DTS解析过程'
author: "404"
header-style: text
tags:
  - Linux
  - DTS
---
>本篇文章转自[smcdef](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html)

在分析DTS解析前，我们先假设系统中的DTS设备树已经生成，先不关心系统如何生成DTS设备树。

#### Device Tree头信息
DTS的结构是方便我们理解，为了让系统操作，需要DTC来将DTS编译成dtb，它是一种二进制格式。  
Linux环境下通过fdtddump工具，可以看到dtb文件的输出，以s5pv21_smc.dtb为例来说明，执行`fdtdump -sd s5pv21_smc.dtb`后输出如下：
```
// magic:  0xd00dfeed
// totalsize:  0xce4 (3300)
// off_dt_struct: 0x38
// off_dt_strings: 0xc34
// off_mem_rsvmap: 0x28
// version: 17
// last_comp_version: 16
// boot_cpuid_phys: 0x0
// size_dt_strings: 0xb0
// size_dt_struct: 0xbfc
```
以上就是Device Tree文件头信息，存储在dtb文件的开头部分。在Linux内核中使用`struct fdt_header`结构体描述，`struct fdt_header`结构体定义在'scripts\\dtc\\libfdt\\fdt.h'文件中。
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
fdtdump工具的输出信息即是以上结构中每一个成员的值，`struct fdt_header`结构体包含了Device Tree的私有信息。例如: `fdt_header.magic`是fdt的魔数,固定值为`0xd00dfeed`，`fdt_header.totalsize`是fdt文件的大小。使用二进制工具打开s5pv21_smc.dtb验证。s5pv21_smc.dtb二进制文件头信息如下图所示。从下图中可以得到Device Tree的文件是以大端模式储存。并且，头部信息和fdtdump的输出信息一致。
![avatar](/img/in-post/Linux/2019219001.png)
Device Tree中的节点信息举例如下图所示：
![avatar](/img/in-post/Linux/2019219002.png)

Device Tree源文件的结构分为header、fill_area、dt_struct及dt_string四个区域。header为头信息，fill_area为填充区域，填充数字0，dt_struct存储节点数值及名称相关信息，dt_string存储属性名。例如：a-string-property就存储在dt_string区，\"A string\"及node1就存储在dt_struct区域。  

我们可以给一个设备节点添加lable，之后可以通过&lable的形式访问这个lable，这种引用是通过phandle（pointer handle）进行的。例如，下图中的node1就是一个lable，node@0的子节点child-node@0通过&node1引用node@1节点。像是这种phandle的节点，在经过DTC工具编译之后，&node1会变成一个特殊的整型数字n，假设n值为1，那么在node@1节点下自动生成两个属性，属性如下：
linux,phandle = <0x00000001>;
phandle = <0x00000001>;

node@0的子节点child-node@0中的a-reference-to-something = <&node1>会变成a-reference-to-something = < 0x00000001>。此处0x00000001就是一个phandle得值，每一个phandle都有一个独一无二的整型值，在后续kernel中通过这个特殊的数字间接找到引用的节点。通过查看fdtdump输出信息以及dtb二进制文件信息，得到struct fdt_header和文件结构之间的关系信息如所示。
![avatar](/img/in-post/Linux/2019219003.png)  
通过以上分析，可以得到Device Tree文件结构如下图所示。dtb的头部首先存放的是fdt_header的结构体信息，接着是填充区域，填充大小为off_dt_struct – sizeof(struct fdt_header)，填充的值为0。接着就是struct fdt_property结构体的相关信息。最后是dt_string部分。  
![avatar](/img/in-post/Linux/2019219004.png)  

Device Tree源文件的结构分为header、fill_area、dt_struct及dt_string四个区域。fill_area区域填充数值0。节点（node）信息使用`struct fdt_node_header`结构体描述。属性信息使用`struct fdt_property`结构体描述。各个结构体信息如下:
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
`struct fdt_node_header`描述节点信息，tag是标识node的起始结束等信息的标志位，name指向node名称的首地址。tag的取值如下：
```c
#define FDT_BEGIN_NODE	0x1		/ * Start node: full name * /
#define FDT_END_NODE	0x2		/ * End node * /
#define FDT_PROP	      0x3		/ * Property: name off, size, content * /
#define FDT_NOP		0x4		/ * nop * /
#define FDT_END		0x9
```
`FDT_BEGIN_NODE`和`FDT_END_NODE`标识node节点的起始和结束，`FDT_PROP`标识node节点下面的属性起始符，`FDT_END`标识Device Tree的结束标识符。因此，对于每个node节点的tag标识符一般为`FDT_BEGIN_NODE`，对于每个node节点下面的属性的tag标识符一般是FDT_PROP。  

描述属性采用`struct fdt_property`描述，tag标识是属性，取值为`FDT_PROP`；len为属性值的长度（包括‘\0’，单位：字节）；`nameoff`为属性名称存储位置相对于off_dt_strings的偏移地址。

例如：`compatible = \"samsung,goni\", \"samsung,s5pv210\"`;`compatible`是属性名称，\"samsung,goni\", \"samsung,s5pv210\"是属性值。`compatible`属性名称字符串存放的区域是dt_string。\"samsung,goni\", \"samsung,s5pv210\"存放的位置是`fdt_property.data`后面。因此`fdt_property.data`指向该属性值。`fdt_property.tag`的值为属性标识，`len`为属性值的长度（包括‘\0’，单位：字节）,此处`len = 29`。`nameoff`为`compatible`字符串的位置相对于`off_dt_strings`的偏移地址，即`&compatible = nameoff + off_dt_strings`。 dt_struct在Device Tree中的结构如下图所示。节点的嵌套也带来tag标识符的嵌套。
![avatar](/img/in-post/Linux/2019219005.png)  

#### kernel解析Device Tree
Device Tree文件结构描述就以上`struct fdt_header`、`struct fdt_node_header`及`struct fdt_property`三个结构体描述。kernel会根据Device Tree的结构解析出kernel能够使用的struct property结构体。kernel根据Device Tree中所有的属性解析出数据填充struct property结构体。struct property结构体描述如下：
```c
struct property {
	char * name;                          / * property full name * /
	int length;                          /* property value length * /
	void * value;                         / * property value * /
	struct property * next;             / * next property under the same node * /
	unsigned long flags;
	unsigned int unique_id;
	struct bin_attribute attr;        / * 属性文件，与sysfs文件系统挂接 * /
};
```
总的来说，kernel根据Device Tree的文件结构信息转换成struct property结构体，并将同一个node节点下面的所有属性通过property.next指针进行链接，形成一个单链表。  

kernel中究竟是如何解析Device Tree的呢？下面分析函数解析过程。函数调用过程如下图所示。kernel的C语言阶段的入口函数是`init/main.c/stsrt_kernel()`函数，在`early_init_dt_scan_nodes()`中会做以下三件事：
- 扫描`/chosen或者/chose@0`节点下面的bootargs属性值到boot_command_line，此外，还处理initrd相关的property，并保存在`initrd_start`和`initrd_end`这两个全局变量中；
- 扫描根节点下面，获取`{size,address}-cells`信息，并保存在`dt_root_size_cells`和`dt_root_addr_cells`全局变量中；
- 扫描具有`device_type = \"memory\"`属性的`/memory`或者`/memory@0`节点下面的`reg`属性值，并把相关信息保存在`meminfo`中，全局变量`meminfo`保存了系统内存相关的信息。
![avatar](/img/in-post/Linux/2019219006.png)  

Device Tree中的每一个node节点经过kernel处理都会生成一个`struct device_node`的结构体，`struct device_node`最终一般会被挂接到具体的`struct device`结构体。`struct device_node`结构体描述如下：
```c
struct device_node {
	const char *name;              /* node的名称，取最后一次\"/\"和\"@\"之间子串 */
	const char *type;              /* device_type的属性名称，没有为<NULL> */
	phandle phandle;               /* phandle属性值 */
	const char *full_name;        /* 指向该结构体结束的位置，存放node的路径全名，例如：/chosen */
	struct fwnode_handle fwnode;

	struct	property *properties;  /* 指向该节点下的第一个属性，其他属性与该属性链表相接 */
	struct	property *deadprops;   /* removed properties */
	struct	device_node *parent;   /* 父节点 */
	struct	device_node *child;    /* 子节点 */
	struct	device_node *sibling;  /* 姊妹节点，与自己同等级的node */
	struct	kobject kobj;            /* sysfs文件系统目录体现 */
	unsigned long _flags;          /* 当前node状态标志位，见/include/linux/of.h line124-127 */
	void	*data;
};

/* flag descriptions (need to be visible even when !CONFIG_OF) */
#define OF_DYNAMIC        1 /* node and properties were allocated via kmalloc */
#define OF_DETACHED       2 /* node has been detached from the device tree*/
#define OF_POPULATED      3 /* device already created for the node */
#define OF_POPULATED_BUS 4 /* of_platform_populate recursed to children of this node */
```
`struct device_node`结构体中的每个成员作用已经备注了注释信息，下面分析以上信息是如何得来的。Device Tree的解析首先从`unflatten_device_tree()`开始，代码列出如下：
```c
/**
 * unflatten_device_tree - create tree of device_nodes from flat blob
 *
 * unflattens the device-tree passed by the firmware, creating the
 * tree of struct device_node. It also fills the \"name\" and \"type\"
 * pointers of the nodes so the normal device-tree walking functions
 * can be used.
 */
void __init unflatten_device_tree(void)
{
	__unflatten_device_tree(initial_boot_params, &of_root,
				early_init_dt_alloc_memory_arch);

	/* Get pointer to \"/chosen\" and \"/aliases\" nodes for use everywhere */
	of_alias_scan(early_init_dt_alloc_memory_arch);
}

/**
 * __unflatten_device_tree - create tree of device_nodes from flat blob
 *
 * unflattens a device-tree, creating the
 * tree of struct device_node. It also fills the \"name\" and \"type\"
 * pointers of the nodes so the normal device-tree walking functions
 * can be used.
 * @blob: The blob to expand
 * @mynodes: The device_node tree created by the call
 * @dt_alloc: An allocator that provides a virtual address to memory
 * for the resulting tree
 */
static void __unflatten_device_tree(const void *blob,
			     struct device_node **mynodes,
			     void * (*dt_alloc)(u64 size, u64 align))
{
	unsigned long size;
	int start;
	void *mem;

    /* 省略部分不重要部分 */
	/* First pass, scan for size */
	start = 0;
	size = (unsigned long)unflatten_dt_node(blob, NULL, &start, NULL, NULL, 0, true);
	size = ALIGN(size, 4);

	/* Allocate memory for the expanded device tree */
	mem = dt_alloc(size + 4, __alignof__(struct device_node));
	memset(mem, 0, size);

	/* Second pass, do actual unflattening */
	start = 0;
	unflatten_dt_node(blob, mem, &start, NULL, mynodes, 0, false);
}
```
分析以上代码，在`unflatten_device_tree()`中，调用函数`__unflatten_device_tree()`，参数`initial_boot_params`指向Device Tree在内存中的首地址，`of_root`在经过该函数处理之后，会指向根节点，`early_init_dt_alloc_memory_arch`是一个函数指针，为`struct device_node`和`struct property`结构体分配内存的回调函数（callback）。在`__unflatten_device_tree()`函数中，两次调用`unflatten_dt_node()`函数，第一次是为了得到Device Tree转换成`struct device_node`和`struct property`结构体需要分配的内存大小，第二次调用才是具体填充每一个`struct device_node`和`struct property`结构体。`unflatten_dt_node()`代码列出如下：
```c
/**
 * unflatten_dt_node - Alloc and populate a device_node from the flat tree
 * @blob: The parent device tree blob
 * @mem: Memory chunk to use for allocating device nodes and properties
 * @poffset: pointer to node in flat tree
 * @dad: Parent struct device_node
 * @nodepp: The device_node tree created by the call
 * @fpsize: Size of the node path up at the current depth.
 * @dryrun: If true, do not allocate device nodes but still calculate needed
 * memory size
 */
static void * unflatten_dt_node(const void *blob,
				void *mem,
				int *poffset,
				struct device_node *dad,
				struct device_node **nodepp,
				unsigned long fpsize,
				bool dryrun)
{
	const __be32 *p;
	struct device_node *np;
	struct property *pp, **prev_pp = NULL;
	const char *pathp;
	unsigned int l, allocl;
	static int depth;
	int old_depth;
	int offset;
	int has_name = 0;
	int new_format = 0;

	/* 获取node节点的name指针到pathp中 */
	pathp = fdt_get_name(blob, *poffset, &l);
	if (!pathp)
		return mem;

	allocl = ++l;

	/* version 0x10 has a more compact unit name here instead of the full
	 * path. we accumulate the full path size using \"fpsize\", we'll rebuild
	 * it later. We detect this because the first character of the name is
	 * not '/'.
	 */
	if ((*pathp) != '/') {
		new_format = 1;
		if (fpsize == 0) {
			/* root node: special case. fpsize accounts for path
			 * plus terminating zero. root node only has '/', so
			 * fpsize should be 2, but we want to avoid the first
			 * level nodes to have two '/' so we use fpsize 1 here
			 */
			fpsize = 1;
			allocl = 2;
			l = 1;
			pathp = \"\";
		} else {
			/* account for '/' and path size minus terminal 0
			 * already in 'l'
			 */
			fpsize += l;
			allocl = fpsize;
		}
	}

	/* 分配struct device_node内存，包括路径全称大小 */
	np = unflatten_dt_alloc(&mem, sizeof(struct device_node) + allocl,
				__alignof__(struct device_node));
	if (!dryrun) {
		char *fn;
		of_node_init(np);

		/* 填充full_name，full_name指向该node节点的全路径名称字符串 */
		np->full_name = fn = ((char *)np) + sizeof(*np);
		if (new_format) {
			/* rebuild full path for new format */
			if (dad && dad->parent) {
				strcpy(fn, dad->full_name);
				fn += strlen(fn);
			}
			*(fn++) = '/';
		}
		memcpy(fn, pathp, l);

		/* 节点挂接到相应的父节点、子节点和姊妹节点 */
		prev_pp = &np->properties;
		if (dad != NULL) {
			np->parent = dad;
			np->sibling = dad->child;
			dad->child = np;
		}
	}
	/* 处理该node节点下面所有的property */
	for (offset = fdt_first_property_offset(blob, *poffset);
	     (offset >= 0);
	     (offset = fdt_next_property_offset(blob, offset))) {
		const char *pname;
		u32 sz;

		if (!(p = fdt_getprop_by_offset(blob, offset, &pname, &sz))) {
			offset = -FDT_ERR_INTERNAL;
			break;
		}

		if (pname == NULL) {
			pr_info(\"Can't find property name in list !\n\");
			break;
		}
		if (strcmp(pname, \"name\") == 0)
			has_name = 1;
		pp = unflatten_dt_alloc(&mem, sizeof(struct property),
					__alignof__(struct property));
		if (!dryrun) {
			/* We accept flattened tree phandles either in
			 * ePAPR-style \"phandle\" properties, or the
			 * legacy \"linux,phandle\" properties.  If both
			 * appear and have different values, things
			 * will get weird.  Don't do that. */

			/* 处理phandle，得到phandle值 */
			if ((strcmp(pname, \"phandle\") == 0) ||
			    (strcmp(pname, \"linux,phandle\") == 0)) {
				if (np->phandle == 0)
					np->phandle = be32_to_cpup(p);
			}
			/* And we process the \"ibm,phandle\" property
			 * used in pSeries dynamic device tree
			 * stuff */
			if (strcmp(pname, \"ibm,phandle\") == 0)
				np->phandle = be32_to_cpup(p);
			pp->name = (char *)pname;
			pp->length = sz;
			pp->value = (__be32 *)p;
			*prev_pp = pp;
			prev_pp = &pp->next;
		}
	}
	/* with version 0x10 we may not have the name property, recreate
	 * it here from the unit name if absent
	 */
	/* 为每个node节点添加一个name的属性 */
	if (!has_name) {
		const char *p1 = pathp, *ps = pathp, *pa = NULL;
		int sz;

		/* 属性name的value值为node节点的名称，取\"/\"和\"@\"之间的子串 */
		while (*p1) {
			if ((*p1) == '@')
				pa = p1;
			if ((*p1) == '/')
				ps = p1 + 1;
			p1++;
		}
		if (pa < ps)
			pa = p1;
		sz = (pa - ps) + 1;
		pp = unflatten_dt_alloc(&mem, sizeof(struct property) + sz,
					__alignof__(struct property));
		if (!dryrun) {
			pp->name = \"name\";
			pp->length = sz;
			pp->value = pp + 1;
			*prev_pp = pp;
			prev_pp = &pp->next;
			memcpy(pp->value, ps, sz - 1);
			((char *)pp->value)[sz - 1] = 0;
		}
	}
	/* 填充device_node结构体中的name和type成员 */
	if (!dryrun) {
		*prev_pp = NULL;
		np->name = of_get_property(np, \"name\", NULL);
		np->type = of_get_property(np, \"device_type\", NULL);

		if (!np->name)
			np->name = \"<NULL>\";
		if (!np->type)
			np->type = \"<NULL>\";
	}

	old_depth = depth;
	*poffset = fdt_next_node(blob, *poffset, &depth);
	if (depth < 0)
		depth = 0;
	/* 递归调用node节点下面的子节点 */
	while (*poffset > 0 && depth > old_depth)
		mem = unflatten_dt_node(blob, mem, poffset, np, NULL,
					fpsize, dryrun);

	if (*poffset < 0 && *poffset != -FDT_ERR_NOTFOUND)
		pr_err(\"unflatten: error %d processing FDT\n\", *poffset);

	/*
	 * Reverse the child list. Some drivers assumes node order matches .dts
	 * node order
	 */
	if (!dryrun && np->child) {
		struct device_node *child = np->child;
		np->child = NULL;
		while (child) {
			struct device_node *next = child->sibling;
			child->sibling = np->child;
			np->child = child;
			child = next;
		}
	}

	if (nodepp)
		*nodepp = np;

	return mem;
}

```
通过以上函数处理就得到了所有的`struct device_node`结构体，为每一个node都会自动添加一个名称为\"name\"的property，`property.length`的值为当前node的名称取最后一个\"/\"和\"@\"之间的子串（包括‘\0’）。例如：/serial@e2900800，则length = 7，property.value = device_node.name = \"serial\"。

#### platform_device和device_node绑定
经过以上解析，Device Tree的数据已经全部解析出具体的`struct device_node`和`struct property`结构体，下面需要和具体的device进行绑定。首先讲解`platform_device`和`device_node`的绑定过程。在`arch/arm/kernel/setup.c`文件中，`customize_machine()``函数负责填充`struct platform_device`结构体。函数调用过程如下图所示。
![avatar](/img/in-post/Linux/2019219007.png)  
```c
const struct of_device_id of_default_bus_match_table[] = {
	{ .compatible = \"simple-bus\", },
	{ .compatible = \"simple-mfd\", },
#ifdef CONFIG_ARM_AMBA
	{ .compatible = \"arm,amba-bus\", },
#endif /* CONFIG_ARM_AMBA */
	{} /* Empty terminated list */
};

int of_platform_populate(struct device_node *root,
			const struct of_device_id *matches,
			const struct of_dev_auxdata *lookup,
			struct device *parent)
{
	struct device_node *child;
	int rc = 0;

	/* 获取根节点 */
	root = root ? of_node_get(root) : of_find_node_by_path(\"/\");
	if (!root)
		return -EINVAL;

	/* 为根节点下面的每一个节点创建platform_device结构体 */
	for_each_child_of_node(root, child) {
		rc = of_platform_bus_create(child, matches, lookup, parent, true);
		if (rc) {
			of_node_put(child);
			break;
		}
	}
	/* 更新device_node flag标志位 */
	of_node_set_flag(root, OF_POPULATED_BUS);

	of_node_put(root);
	return rc;
}

static int of_platform_bus_create(struct device_node *bus,
				  const struct of_device_id *matches,
				  const struct of_dev_auxdata *lookup,
				  struct device *parent, bool strict)
{
	const struct of_dev_auxdata *auxdata;
	struct device_node *child;
	struct platform_device *dev;
	const char *bus_id = NULL;
	void *platform_data = NULL;
	int rc = 0;

	/* 只有包含\"compatible\"属性的node节点才会生成相应的platform_device结构体 */
	/* Make sure it has a compatible property */
	if (strict && (!of_get_property(bus, \"compatible\", NULL))) {
		return 0;
	}
	/* 省略部分代码 */
	/*
	 * 针对节点下面得到status = \"ok\" 或者status = \"okay\"或者不存在status属性的
	 * 节点分配内存并填充platform_device结构体
	 */
	dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
	if (!dev || !of_match_node(matches, bus))
		return 0;

	/* 递归调用节点解析函数，为子节点继续生成platform_device结构体，前提是父节点
	 * 的\"compatible\" = \"simple-bus\"，也就是匹配of_default_bus_match_table结构体中的数据
	 */
	for_each_child_of_node(bus, child) {
		rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
		if (rc) {
			of_node_put(child);
			break;
		}
	}
	of_node_set_flag(bus, OF_POPULATED_BUS);
	return rc;
}
```

总的来说，当`of_platform_populate()`函数执行完毕，kernel就为DTB中所有包含`compatible`属性名的第一级node创建`platform_device`结构体，并向平台设备总线注册设备信息。如果第一级node的`compatible`属性值等于\"simple-bus\"、\"simple-mfd\"或者\"arm,amba-bus\"的话，kernel会继续为当前node的第二级包含`compatible`属性的node创建`platform_device`结构体，并注册设备。Linux系统下的设备大多都是挂载在平台总线下的，因此在平台总线被注册后，会根据of_root节点的树结构，去寻找该总线的子节点，所有的子节点将被作为设备注册到该总线上。

#### i2c_client和device_node绑定
经过`customize_machine()`函数的初始化，DTB已经转换成`platform_device`结构体，这其中就包含`i2c adapter`设备，不同的SoC需要通过平台设备总线的方式自己实现`i2c adapter`设备的驱动。例如：`i2c_adapter`驱动的`probe`函数中会调用`i2c_add_numbered_adapter()`注册`adapter`驱动，函数流执行如下图所示。
![avatar](/img/in-post/Linux/2019219008.png)  

在`of_i2c_register_devices()`函数内部便利i2c节点下面的每一个子节点，并为子节点（status = \"isable\"的除外）创建`i2c_client`结构体，并与子节点的`device_node`挂接。其中`i2c_client`的填充是在`i2c_new_device()`中进行的，最后`device_register()`。在构建`i2c_client`的时候，会对node下面的`compatible`属性名称的厂商名字去除作为`i2c_client`的name。例如：`compatible = “maxim,ds1338\"`,则`i2c_client->name = \"s1338\"`。

#### Device_Tree与sysfs
kernel启动流程为`start_kernel()→rest_init()→kernel_thread():kernel_init()→do_basic_setup()→driver_init()→of_core_init()`，在`of_core_init()`函数中在`sys/firmware/devicetree/base`目录下面为设备树展开成sysfs的目录和二进制属性文件，所有的`node`节点就是一个目录，所有的`property`属性就是一个二进制属性文件。
