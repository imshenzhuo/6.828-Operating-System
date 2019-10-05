# Lab 2: Memory Management

本实验将会为操作系统写内存管理的代码. 内存管理分为两部分

1. 为内核分配物理内存
2. 虚拟内存空间

## 准备

``` bash
athena% cd ~/6.828/lab
athena% add git
athena% git pull
Already up-to-date.
athena% git checkout -b lab2 origin/lab2
Branch lab2 set up to track remote branch refs/remotes/origin/lab2.
Switched to a new branch "lab2"
athena% git merge lab1
Merge made by recursive.
 kern/kdebug.c  |   11 +++++++++--
 kern/monitor.c |   19 +++++++++++++++++++
 lib/printfmt.c |    7 +++----
 3 files changed, 31 insertions(+), 6 deletions(-)
athena% 
```



## Part 1: Physical Page Management

哪些页是空闲的? 哪些页是被使用的? 如何组织起来用.

### Exercise 1

按照给定的顺序, 编写代码

#### boot_alloc()

``` C
static void *
boot_alloc(uint32_t n)
{
	static char *nextfree;	// virtual address of next byte of free memory
	char *result;

	// Initialize nextfree if this is the first time.
	// 'end' is a magic symbol automatically generated by the linker,
	// which points to the end of the kernel's bss segment:
	// the first virtual address that the linker did *not* assign
	// to any kernel code or global variables.
	if (!nextfree) {
		extern char end[];
		nextfree = ROUNDUP((char *) end, PGSIZE);
	}

	// Allocate a chunk large enough to hold 'n' bytes, then update
	// nextfree.  Make sure nextfree is kept aligned
	// to a multiple of PGSIZE.
	//
	if (n > 0) 	{
		result = nextfree;
		nextfree += ROUNDUP(n, PGSIZE);
		return result;
	} else	{
		return nextfree;
	}
	// LAB 2: Your code here.
	return NULL;
}
```

#### mem_init

``` c
kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
```

联系下文得到, `pages`其实是数组+链表的组合, 十分巧妙

```c
pages = (struct PageInfo *)boot_alloc(sizeof(struct PageInfo) * npages);
memset(pages, 0, sizeof(struct PageInfo) * npages);
```

#### page_init

##### page&物理内存

在mem_init中分配在`kern_pgdir`后的`pages`包含了计算机系统所有的`npages`个页, 标记页的使用情况的数据结构是这样的

``` C
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;
};
```

只有两项: 下一个空闲页 & 正在被使用的次数

并没有存该页对应那个物理内存, 因为每一个页和他所对应的物理内存是线性关系的. 什么意思呢? 

首先,`pages`是连续存储的数组, 共有`npage`个数组项, 每一个都有自己的索引;
然后, 如果拿到索引是*i*的`struct PageInfo`, 那么他对应的物理内存就是第*i*个页, 也就是*i*左移12位

##### page初始化

该函数的工作就是根据物理内存初始化页表, 哪些是空闲的, 哪些是在使用的, 空闲的要用空闲链表穿起来, 再使用的要标记在忙. 

如下图, 当前的物理内存使用情况. 最开始是BIOS建立的中断描述符表, 不会超过一个页, 然后是lab1中提到的到1M处结束的hole, 之后就是1M处开始的内核以及刚刚分配的`pgdir`和`pages`了



![](http://images2015.cnblogs.com/blog/688140/201511/688140-20151112192634884-863839744.png)

所以任务就是将上图中不是白色的页设置为在使用, 白色的页用链表链接. 注释中也写的很清楚

``` C
void
page_init(void)
{
	size_t i;
	page_free_list = NULL;
	pages[0].pp_link = NULL;
	pages[0].pp_ref = 1;

	for (i = 1; i < npages; i++)
	{
		if (i < npages_basemem)	{
			pages[i].pp_ref = 0;
			pages[i].pp_link = page_free_list;
			page_free_list = &pages[i];
		// } else if( i <= (EXTPHYSMEM / PGSIZE) ||  ) {
		} else if( i < ( ((uint32_t)boot_alloc(0) - KERNBASE) >> PGSHIFT)) {
			pages[i].pp_link = NULL;
			pages[i].pp_ref++;
		} else {
			pages[i].pp_ref = 0;
			pages[i].pp_link = page_free_list;
			page_free_list = &pages[i];
		}
	}
}
```

#### page_alloc & page_free

链表的一个练习

``` C
struct PageInfo *
page_alloc(int alloc_flags)
{
	if (!page_free_list)	return NULL;

	struct PageInfo *pg = page_free_list;
	page_free_list = page_free_list->pp_link;
	pg->pp_link = NULL;
	if (alloc_flags & ALLOC_ZERO)
		memset(page2kva(pg), 0, PGSIZE);
	// Fill this function in
	return pg;
}
void
page_free(struct PageInfo *pp)
{
	assert(pp && pp->pp_ref == 0 && pp->pp_link == NULL);
	pp->pp_link = page_free_list;
	page_free_list = pp;

	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
}
```

## Part 2: Virtual Memory

### Exercise 4

#### pgdir_walk

``` C
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	uint32_t pd_index = PDX(va);
	uint32_t pte_index = PTX(va);
	pde_t pde = pgdir[pd_index];
	if (pde & PTE_P) {
		// 存在 直接返回pte的地址就好了
		// pde中的前20位(PTE_ADDR(pde))存储的是物理地址, 要转为虚拟地址KADDR(*) ptebase
		pte_t *ptebase = KADDR(PTE_ADDR(pde)); 
		return ptebase + pte_index;
	} else if (!create) {
		return NULL;
	}
	// pde没有对应的页表, 分配并且安装之
	struct PageInfo* pg = page_alloc(ALLOC_ZERO);
	if (!pg)	return NULL;
	pg->pp_ref++;
	pgdir[pd_index] = page2pa(pg) | PTE_P | PTE_U | PTE_W;
	// 这里创建了一部分, 该函数仅仅是返回pte
	// pte要不是有效的要不就是0, 因为都初始化为0了
	pte_t *ptebase = KADDR(PTE_ADDR(pgdir[pd_index]));
	return ptebase + pte_index;
	// Fill this function in
	return NULL;
}
```

#### boot_map_region

只用于一开始的映射

``` C
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	int i;
	for (i = 0; i < (size+PGSIZE-1)/PGSIZE; i++) {
		pte_t* pte = pgdir_walk(pgdir, (void* )va, 1); //create
		if (!pte)	panic("boot_map_region panic: out of memory!\n");
		*pte = pa | perm | PTE_P;
		va += PGSIZE;
		pa += PGSIZE;
	}
	cprintf("end: VA %08x mapped to PA %08x, size:08x\n", va, pa, size);
	// Fill this function in
}
```

#### page_lookup

``` C
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	pte_t *pte = pgdir_walk(pgdir, va, 0);
	if (!pte)	return NULL;
	*pte_store = pte;
	return pa2page(PTE_ADDR(*pte));
	// Fill this function in
	return NULL;
}
```

#### page_remove

``` C
void
page_remove(pde_t *pgdir, void *va)
{
	pte_t *pte_store;
	struct PageInfo *pg = page_lookup(pgdir, va, &pte_store);
	if (!pg)	return;
	page_decref(pg);
	*pte_store = 0;
	tlb_invalidate(pgdir, va);
	// Fill this function in
}
```

#### page_insert

``` C
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	pte_t *pte = pgdir_walk(pgdir, va, 0);
	if(pte)	{
		if (*pte & PTE_P)	// 找到一个页并且再被使用
			page_remove(pgdir, va);
		if (page_free_list == pp) 
			page_free_list = page_free_list->pp_link;
	} else {
		pte = pgdir_walk(pgdir, va, 1);
		if(!pte)	return -E_NO_MEM;
	}
	
	*pte = page2pa(pp) | perm | PTE_P;
	pp->pp_ref++;
	tlb_invalidate(pgdir, va);
	// Fill this function in
	return 0;
}
```

## Part 3: Kernel Address Space

对照图映射

``` bash
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.  JOS user programs map pages temporarily at UTEMP.
 */

```



``` C
boot_map_region(
	kern_pgdir,
	UPAGES,
	ROUNDUP((sizeof(struct PageInfo) * npages), PGSIZE),
	PADDR(pages),
	PTE_U | PTE_P
);
```

``` C
boot_map_region(
	kern_pgdir,
	KSTACKTOP - KSTKSIZE,
	KSTKSIZE,
	PADDR(bootstack),
	(PTE_W) | (PTE_P)
);
```

``` C
boot_map_region(
	kern_pgdir,
	KERNBASE,
	ROUNDUP((0xFFFFFFFF - KERNBASE), PGSIZE),
	0,
	(PTE_W) | (PTE_P)
);
```
