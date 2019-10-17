# Lab 3: User Environments

lab3实现让一个用户态进程运行所需要的内核基础功能

- 建立跟踪用户环境的数据结构
- 创建用户进程
- 加载程序镜像
- 运行用户进程
- 处理系统调用和程序异常

> environment & process 是可以互换的, 此处的意义在于, environment强调JOS, Process强调UNIX

### 准备

合并`lab2`代码

``` bash
athena% cd ~/6.828/lab
athena% add git
athena% git commit -am 'changes to lab2 after handin'
Created commit 734fab7: changes to lab2 after handin
 4 files changed, 42 insertions(+), 9 deletions(-)
athena% git pull
Already up-to-date.
athena% git checkout -b lab3 origin/lab3
Branch lab3 set up to track remote branch refs/remotes/origin/lab3.
Switched to a new branch "lab3"
athena% git merge lab2
Merge made by recursive.
 kern/pmap.c |   42 +++++++++++++++++++
 1 files changed, 42 insertions(+), 0 deletions(-)
athena% 
```

#### 内联汇编

[参考资料](https://link.zhihu.com/?target=http%3A//csapp.cs.cmu.edu/3e/waside/waside-embedded-asm.pdf)

## Part A: User Environments and Exception Handling

新的头文件`inc/env.h`包含了`JOS`用户环境的基本定义. 内核用`Env`数据结构跟踪每一个用户进程. 在本实验中只会创建一个进程, 但要设计支持多进程的内核. 

在文件`kern/env.c`中, 内核维护三个关于环境的全局变量

``` C
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```

- envs 指向`Env`结构数组, 代表OS全部的进程
- curenv  当前环境, 执行envs数组的某一项
- env_free_list 维护所有的未使用的环境

### Environment State

`inc/env.h`定义了`Env`结构如下, 更多的域会在lab4中定义

``` C
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
```



像Unix进程一样. `JOS`环境将“线程”和“地址空间”的概念结合在一起.  线程主要是由保存的寄存器定义, 地址空间被页表目录定义, 进程通过这两个概念和CPU联系. 

`xv6`中每个进程都有kernal stack, `JOS`只有一个 kernal stack.

### Allocating the Environments Array

#### Exercise 1

实验二类似, 分配和映射`envs`数组, 在`kern/pmap.c`文件中

``` C
//////////////////////////////////////////////////////////////////////
// Make 'envs' point to an array of size 'NENV' of 'struct Env'.
// LAB 3: Your code here.
extern struct Env* envs;
envs = (struct Env *)boot_alloc(sizeof(struct Env)* NENV);
```

``` C
boot_map_region(
	kern_pgdir,
	UENVS,
	ROUNDUP((sizeof(struct Env) * NENV), PGSIZE),
	PADDR(envs),
	PTE_U | PTE_P
);
```

分配并且添加映射, 将跟踪记录进程(环境)的`envs`映射到`UENVS`

因为先是`boot_alloc`之后才是`page_init`, 所以对page管理没有影响

### Creating and Running Environments

要依次完成`env.c`的以下函数

- env_init
- env_setup_vm
- region_alloc
- load_icode
- env_create
- env_run

进程的创建是需要可执行文件(Windows中的exe, Linux中的elf)的, 因为还没有文件系统, 所以就加载嵌入在内核里的静态二进制镜像文件, 作为ELF可执行镜像.

#### Exercise 2

> env_init
>

首先是初始化`envs`, 类似于`pages`, 按照注释很容易写出代码, 注意要求第一次调用返回的是`envs[0]`

``` C
void
env_init(void)
{
	// Set up envs array
	// LAB 3: Your code here.
	for(int i = NENV - 1; i >= 0; i--)
	{
		envs[i].env_status = ENV_FREE;
		envs[i].env_id = 0;
		envs[i].env_link = env_free_list;
		env_free_list = &envs[i];
	}
	// Per-CPU part of the initialization
	env_init_percpu();
}
```

> env_setup_vm(struct Env *e)
>

因为操作用户进程的是内核, 所以**env_pgdir**存放的是该进程页表目录的内核虚拟地址, 当切换进程的时候, 内核根据自己的页表模板转为物理地址加载到`cr3`中.

``` C
static int
env_setup_vm(struct Env *e)
{
	int i;
	struct PageInfo *p = NULL;
	// Allocate a page for the page directory
	if (!(p = page_alloc(ALLOC_ZERO)))
		return -E_NO_MEM;

	// LAB 3: Your code here.
	e->env_pgdir = page2kva(p);
	memcpy(e->env_pgdir, kern_pgdir, PGSIZE);
	p->pp_ref++;
	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

	return 0;
}
```

> region_alloc(struct Env *e, void *va, size_t len)
>

这是进程最常用的函数, 为环境`e`分配长度为`len`字节映射到地址空间`va`的内存, 也就是为用户进程分配新的空间. 

- 确保[va, va+len]在分配范围内而且要让[va,  va+len]都要按照页对齐, 就要两边扩展, 所以va使用ROUNDDOWN, va+len使用ROUNDUP
- 用户页的权限全部开放
- 当进程需要分配内存的时候, 实际上就调用的该代码, 特定是一页一页的分配

``` C
static void
region_alloc(struct Env *e, void *va, size_t len)
{
	// LAB 3: Your code here.
	// (But only if you need it for load_icode.)
	//
	// Hint: It is easier to use region_alloc if the caller can pass
	//   'va' and 'len' values that are not page-aligned.
	//   You should round va down, and round (va + len) up.
	//   (Watch out for corner-cases!)
	// Corner case: e equals to NULL
	if(e == 0)
		panic("The struct Env could not be NULL");
	// corner case: len equals to 0
	if(len == 0)
		return;
	uint32_t va_start = ROUNDDOWN((uint32_t)va, PGSIZE);
	uint32_t va_end = ROUNDUP((uint32_t)va + len, PGSIZE);
	// corner case: overflow of va_end
	if(va_start > va_end)
	{
		panic("The va_end is too large, and exceeds 32 bit RAM Limit");
	}
	// For each PGSIZE of va, allocate a physical page and do the mapping
	for(uint32_t va_current = va_start; va_current < va_end; va_current += PGSIZE)
	{
		// First, allocate a physical page
		struct PageInfo* p = page_alloc(0);
		if(p == 0)
			panic("No enough memory when allocating physical page");
		// Then, map the va to this physical page
		if(page_insert(e->env_pgdir, p, (void*)va_current, PTE_W | PTE_U | PTE_P) < 0)
			panic("Page insertaion failed.");
	}
}
```

> load_icode(struct Env *e, uint8_t *binary)

根据读取的可执行文件ELF, 为一个用户进程建立最初栈和处理器flags, 栈的初始大小是一个page

该函数反复使用region_alloc为进程分配了程序段内存和一页大小的stack内存

``` C
static void
load_icode(struct Env *e, uint8_t *binary)
{
	// LAB 3: Your code here.
	struct Elf* elf = (struct Elf*)binary;
	// check the ELF magic header
	if (elf->e_magic != ELF_MAGIC)
		panic("This is not a valid ELF Binary Image!");
	// Because env_pgdir and kern_pgdir are almost the same, so mappings for bianry also
	// exists in env_pgdir, so we switch to env_pgdir first, so we can use memset functions
	// Do not forget to switch back!
	lcr3(PADDR(e->env_pgdir));
	// Load each segment from binary image to the corresponding va
	struct Proghdr* ph = (struct Proghdr*)((uint32_t)binary + elf->e_phoff);
	struct Proghdr* eph = ph + ((struct Elf*)binary)->e_phnum;
	uint32_t va;
	for(; ph < eph; ph++)
	{
		if(ph->p_type != ELF_PROG_LOAD)
			continue;
		va = (uint32_t)binary + ph->p_offset;
		// Allocate ph->p_memsz bytes using the function we finished before
		region_alloc(e, (void*)ph->p_va, ph->p_memsz);
		// first clear ph->p_memsz bytes
		memset((void*)ph->p_va, 0, ph->p_memsz);
		// Then do the copy
		memcpy((void*)ph->p_va, (void*)va, ph->p_filesz);
	}
	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.
	region_alloc(e, (void*)(USTACKTOP - PGSIZE), PGSIZE);
	// switch back to kern_pgdir
	lcr3(PADDR(kern_pgdir));
	// Finally, set the entry point
	e->env_tf.tf_eip = elf->e_entry;
	// LAB 3: Your code here.
}
```


> env_create


创建环境

``` C
void
env_create(uint8_t *binary, enum EnvType type)
{
	// LAB 3: Your code here.
	struct Env* e = NULL;
	if(env_alloc(&e, 0) < 0)
		panic("Env Allocation Failed");
	load_icode(e, binary);
	e->env_type = type;
}
```

> env_run

运行环境

``` C
void
env_run(struct Env *e)
{
	// Step 1: If this is a context switch (a new environment is running):
	//	   1. Set the current environment (if any) back to
	//	      ENV_RUNNABLE if it is ENV_RUNNING (think about
	//	      what other states it can be in),
	//	   2. Set 'curenv' to the new environment,
	//	   3. Set its status to ENV_RUNNING,
	//	   4. Update its 'env_runs' counter,
	//	   5. Use lcr3() to switch to its address space.
	// Step 2: Use env_pop_tf() to restore the environment's
	//	   registers and drop into user mode in the
	//	   environment.

	// Hint: This function loads the new environment's state from
	//	e->env_tf.  Go back through the code you wrote above
	//	and make sure you have set the relevant parts of
	//	e->env_tf to sensible values.

	// LAB 3: Your code here.
	if(curenv != NULL)
	{
		if(curenv->env_status == ENV_RUNNING)
			curenv->env_status = ENV_RUNNABLE;
		// Question: Do we need to save curenv's registers?
	}
	curenv = e;
	curenv->env_status = ENV_RUNNING;
	curenv->env_runs++;
	lcr3(PADDR(curenv->env_pgdir));
	env_pop_tf(&curenv->env_tf);
	//panic("env_run not yet implemented");
}
```

代码调用关系

- start (kern/entry.S)
- i386_init (kern/init.c)
  - cons_init
  - mem_init
  - env_init
  - trap_init
  - env_create
  - env_run
    - env_pop_tf

到目前为止, 并不能正常运行一个环境, 因为`JOS`还不能完成用户态到内核态的切换, 而且还可能出现无限重启的现象. 

### Handling Interrupts and Exceptions

目前为止, 用户空间中的第一个int $ 0x30系统调用指令是死胡同, 处理器进入用户态就回不到内核态了, 需要完成基本的异常和系统调用处理. 

### Basics of Protected Control Transfer

异常和中断都是受保护的控制转换, 让处理器从用户态切换到内核态, 保护是指在转换的过程中, 不给用户态代码干扰内核或者其他进程的机会. 

在Intel术语中, **中断**通常是处理器外部的异步时间造成的, 比如外部IO设备; 相反**异常**是当前的运行代码同步造成的, 比如除零异常.

在x86中, 设计了两个保护机制提供了保护, 只有在特定条件下才能进入内核

1. 中断描述符表, 内核指定进入哪个地方  如下图
   1. IDT可以在内核私有内存的任意位置, 由寄存器IDTR保存IDT的位置; 如果是256个中断向量, 每个8字节, 265个也就是占半个页大小
   2. IDT的每一项是一个描述符, 共有三种形式, 一本只用两种 trap 和 exception, 最主要的两个域是base & limit, 就是eip以及cs寄存器
2. `TSS Task State Segment` 
   1. 系统需要有一个区域用来保存用户态进程状态, 比如IP CS寄存器值, 但是，为了防止其他用户态代码做恶, 必须让该区域不能被其他用户态代码影响. 
   2. 出于此考虑, 当x86处理器发生一个中断或者异常造成用户态切换为内核态的同时, 也切换stack, **TSS指定了该stack的位置**, 处理器将旧的状态入栈, `SS`, `ESP`, `EFLAGS`, `CS`, `EIP`,以及可选的错误码, 然后才从IDT中加载cs和ip寄存器, 截止设置es和ss指向新的stack
   3. 尽管TSS很大，并且可能有多种用途, `JOS`只用它定义处理器用户态向内核态切换的内核栈, `JOS`只用TSS的`ESP0`和`SS0`域

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig9-1.gif)



### Types of Exceptions and Interrupts

中断包括

- 可屏蔽中断 由CPU外部设备向CPU的INTR引脚发送信号
- 不可屏蔽中断  由CPU外部设备向CPU的NMI引脚发送信号

异常包括

- 处理器检测到的, 可以细分为fault trap和abort
- 主动编程的  比如`INT 3` 等

编号

- 0~31号是处理器内部生成的同步异常向量号, 比如页错误就是向量14, 这些都是内部自己产生的
- 大于31号的是通过int指令生成的软件中断, 或者是外部设备发出的异步硬件中断

在本节中完成`JOS`处理0~31号的异常, 下一节中完善`JOS`处理系统调用中断(软中断), lab 4中完成外部中断的一个例子. 

### An Example

在TSS上

```bash
 					 +--------------------+ KSTACKTOP             
                     | 0x00000 | old SS   |     " - 4
                     |      old ESP       |     " - 8
                     |     old EFLAGS     |     " - 12
                     | 0x00000 | old CS   |     " - 16
                     |      old EIP       |     " - 20 <---- ESP 
                     +--------------------+             
```

### Nested Exceptions and Interrupts

异常和中断可以嵌套, 用户态到内核态要切换stack, 如果本来就在内核态, 那么就不切换stack, 直接在原来的stack上push就可以

### Setting Up the IDT

现在先建立和处理0~31号中断向量

头文件`inc/trap.h`和`kern/trap.h`中关于中断和异常的重要定义

- `kern/trap.h`包含对内核严格私有的定义
- `inc/trap.h`包含对用户级程序和库也可能有用的定义

并不是全部32个异常都会产生, 在处理器中有的异常压根不会存在. 

下面代码中以此构建中断向量表`IDT`, 表中的每一项有base 和 limit 以及优先级和是否trap

- base是中断向量对应的程序入口的段寄存器值 => CS
- limit是中断向量对应的程序入口的偏移寄存器值 => EIP
- 优先级表示描述符的优先级
- istrap 1代表trap 0 代表interrupt, 在interrupt中是中断寄存器IF是清零的

``` C
void
trap_init(void)
{
	extern struct Segdesc gdt[];

	// LAB 3: Your code here.
	void DIVIDE();
    SETGATE(idt[T_DIVIDE], 0, GD_KT, DIVIDE, 0);
	void DEBUG();
	SETGATE(idt[T_DEBUG], 0, GD_KT, DEBUG, 0);
	void NMI();
	SETGATE(idt[T_NMI], 0, GD_KT, NMI, 0);
	void BRKPT();
	SETGATE(idt[T_BRKPT], 0, GD_KT, BRKPT, 3);
	void OFLOW();
	SETGATE(idt[T_OFLOW], 0, GD_KT, OFLOW, 0);
	void BOUND();
	SETGATE(idt[T_BOUND], 0, GD_KT, BOUND, 0);
	void ILLOP();
	SETGATE(idt[T_ILLOP], 0, GD_KT, ILLOP, 0);
	void DEVICE();
	SETGATE(idt[T_DEVICE], 0, GD_KT, DEVICE, 0);
	void DBLFLT();
	SETGATE(idt[T_DBLFLT], 0, GD_KT, DBLFLT, 0);
	void TSS();
	SETGATE(idt[T_TSS], 0, GD_KT, TSS, 0);
	void SEGNP();
	SETGATE(idt[T_SEGNP], 0, GD_KT, SEGNP, 0);
	void STACK();
	SETGATE(idt[T_STACK], 0, GD_KT, STACK, 0);
	void GPFLT();
	SETGATE(idt[T_GPFLT], 0, GD_KT, GPFLT, 0);
	void PGFLT();
	SETGATE(idt[T_PGFLT], 0, GD_KT, PGFLT, 0);
	void FPERR();
	SETGATE(idt[T_FPERR], 0, GD_KT, FPERR, 0);
	void ALIGN();
	SETGATE(idt[T_ALIGN], 0, GD_KT, ALIGN, 0);
	void MCHK();
	SETGATE(idt[T_MCHK], 0, GD_KT, MCHK, 0);
	void SIMDERR();
	SETGATE(idt[T_SIMDERR], 0, GD_KT, SIMDERR, 0);
	void SYSCALL();
	SETGATE(idt[T_SYSCALL], 1, GD_KT, SYSCALL, 3);
	void DEFAULT();
	SETGATE(idt[T_DEFAULT], 0, GD_KT, DEFAULT, 0);
	// Per-CPU setup 
	trap_init_percpu();
}
```

上面代码中的函数都只有声明没有定义, 该函数的定义是汇编完成的, 如下代码

``` C
/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */
TRAPHANDLER_NOEC(DIVIDE, T_DIVIDE)
TRAPHANDLER_NOEC(DEBUG, T_DEBUG)
TRAPHANDLER_NOEC(NMI, T_NMI)
TRAPHANDLER_NOEC(BRKPT, T_BRKPT)
TRAPHANDLER_NOEC(OFLOW, T_OFLOW)
TRAPHANDLER_NOEC(BOUND, T_BOUND)
TRAPHANDLER_NOEC(ILLOP, T_ILLOP)
TRAPHANDLER_NOEC(DEVICE, T_DEVICE)
TRAPHANDLER(DBLFLT, T_DBLFLT)
TRAPHANDLER(TSS, T_TSS)
TRAPHANDLER(SEGNP, T_SEGNP)
TRAPHANDLER(STACK, T_STACK)
TRAPHANDLER(GPFLT, T_GPFLT)
TRAPHANDLER(PGFLT, T_PGFLT)
TRAPHANDLER_NOEC(FPERR, T_FPERR)
TRAPHANDLER(ALIGN, T_ALIGN)
TRAPHANDLER_NOEC(MCHK, T_MCHK)
TRAPHANDLER_NOEC(SIMDERR, T_SIMDERR)
TRAPHANDLER_NOEC(SYSCALL, T_SYSCALL)
TRAPHANDLER_NOEC(DEFAULT, T_DEFAULT)
```

以上一个个硬编码全都是为了记录异常的编号, 之后全部跳转到alltraps处, 构建后tf参数后, 在跳转到C语言trap函数

trap函数的参数的tf, 所以参考tf的结构定义, 需要软件手动添加的也就是这些

``` C
	struct PushRegs tf_regs;
	uint16_t tf_es;
	uint16_t tf_padding1;
	uint16_t tf_ds;
	uint16_t tf_padding2;
	uint32_t tf_trapno;
```

逆序压栈即可, 之前在TRAPHANDLER已经将trapno压栈, 接着完成其他的部分压栈后跳转

``` C
/*
 * Lab 3: Your code here for _alltraps
 */
.global _alltraps
_alltraps:
	# according to struct Trapframe, we need to push %ds and %es
	pushl %ds
	pushl %es
	pushal
	movw $GD_KD, %ax
	movw %ax, %ds
	movw %ax, %es
	pushl %esp 
	call trap
```



## Part B: Page Faults, Breakpoints Exceptions, and System Calls

所有的exception处理都汇聚到了trap函数, 函数参数tf包含了处理异常所需要的信息, 函数主体包括调用处理的子函数以及返回程序继续执行, 处理函数如下

``` C
static void
trap_dispatch(struct Trapframe *tf)
{
	// Handle processor exceptions.
	// LAB 3: Your code here.
	// Dispatch System Calls
	if(tf->tf_trapno == T_SYSCALL)
	{
		int32_t ret_val;
		struct PushRegs* regs = &(tf->tf_regs);
		ret_val = syscall(regs->reg_eax, regs->reg_edx, regs->reg_ecx, 
		regs->reg_ebx, regs->reg_edi, regs->reg_esi);
		// eax is unsigned, and store the return value of syscall
		regs->reg_eax = (uint32_t)ret_val;
		return;
	}
	if(tf->tf_trapno == T_PGFLT)
	{
		page_fault_handler(tf);
		return;
	}
	if(tf->tf_trapno == T_BRKPT || tf->tf_trapno == T_DEBUG)
	{
		monitor(tf);
		return;
	}
	// Unexpected trap: The user process or the kernel has a bug.
	print_trapframe(tf);
	if (tf->tf_cs == GD_KT)
		panic("unhandled trap in kernel");
	else {
		env_destroy(curenv);
		return;
	}
}
```

### Handling Page Faults

中断向量14 -- 页错误异常是尤其重要的一个联系. 当处理器发生页异常, 要访问的虚拟地址就被存在`CR2`寄存器中

``` C
void
page_fault_handler(struct Trapframe *tf)
{
	uint32_t fault_va;

	// Read processor's CR2 register to find the faulting address
	fault_va = rcr2();

	// Handle kernel-mode page faults.
	// LAB 3: Your code here.
	if((tf->tf_cs & 3) == 0)
		panic("Page Fault in Kernel-Mode at %08x.\n", fault_va);

	// We've already handled kernel-mode exceptions, so if we get here,
	// the page fault happened in user mode.

	// Destroy the environment that caused the fault.
	cprintf("[%08x] user fault va %08x ip %08x\n",
		curenv->env_id, fault_va, tf->tf_eip);
	print_trapframe(tf);
	env_destroy(curenv);
}
```

当在内核态的时候, 直接panic, 如果是用户态, 就结束进程. 

### The Breakpoint Exception

### System Call

``` C
// Dispatches to the correct kernel function, passing the arguments.
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	// Call the function corresponding to the 'syscallno' parameter.
	// Return any appropriate return value.
	// LAB 3: Your code here.
	
	switch (syscallno) {
		case SYS_cputs:
			sys_cputs((const char*)a1, (size_t)a2);
			// sys_cputs does not return a value
			return 0;
		case SYS_cgetc:
			return sys_cgetc();
		case SYS_getenvid:
			return sys_getenvid();
		case SYS_env_destroy:
			return sys_env_destroy((envid_t)a1);
		case NSYSCALLS:
			return 0;
		default:
			return -E_INVAL;
	}
	// panic("syscall not implemented");
}
```

系统调用编号在tf中的ax寄存器中, 按照取出来的编号执行

``` C
static void
sys_cputs(const char *s, size_t len)
{
	// Check that the user has permission to read memory [s, s+len).
	// Destroy the environment if not.

	// LAB 3: Your code here.
	user_mem_assert(curenv, (void*)s, len, PTE_U);
	pde_t* pgdir = curenv->env_pgdir;
	uint32_t addr = (uint32_t)s;
    pte_t* pte_addr = NULL;
	for(; addr < (uint32_t)(s + len); addr += PGSIZE)
	{
		// If the address is not valid, or user has no permission on it
		if(page_lookup(pgdir, (void*)addr, &pte_addr) == NULL || !(*pte_addr & PTE_U))
		{
			cprintf("Invalid Memory Access\n");
			env_destroy(curenv);
			return;
		}
	}
	// Print the string supplied by the user.
	cprintf("%.*s", len, s);
}
```

`user_mem_assert`在之后的实验完成

### User-mode startup

``` C
void
libmain(int argc, char **argv)
{
	// set thisenv to point at our Env structure in envs[].
	// LAB 3: Your code here.
	envid_t envid = sys_getenvid();
	thisenv = envs + ENVX(envid);

	// save the name of the program so that panic() can use it
	if (argc > 0)
		binaryname = argv[0];

	// call user main routine
	umain(argc, argv);

	// exit gracefully
	exit();
}
```



### Page faults and memory protection

大多数系统调用接口都允许用户程序将指针传递给内核。 这些指针指向要读取或写入的用户缓冲区。 然后，内核在执行系统调用时取消引用这些指针。 这有两个问题：

1. 发生在内核态的页错误可能比发生在用户态的页错误严重的多. 如果内核发生页错误(内核bug), 应该让整个内核停止. 但是当内核在解引用用户程序给的指针时, 需要有一种方法来记录实际上引起错误的是用户程序

2. 用户通过系统定义传给内核的指针可能去访问一些用户不能访问而内核可以访问的地方, 内核必须小心不要被欺骗去引用这样的指针，因为这可能会泄露私有信息或破坏内核的完整性。

现在，将通过一种机制来仔细检查这两个问题，该机制将仔细检查从用户空间传递到内核的所有指针。 当程序通过内核指针时，内核将检查地址是否在地址空间的用户部分中，以及页表是否允许进行内存操作。

第一个问题通过将发生在内核态的页错误直接退出解决. 第二个通过如下的添加检错代码实现. 

``` C
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
	// LAB 3: Your code here.
	// flag shows whether The check fails
	int flag = 0;
	// pte_addr points the page table entry
	pte_t* pte_addr = NULL;
	// addr is not a loop variable because we need it later to set user_mem_check_addr
	uintptr_t addr;
	// perm may not contain PTE_U 
	perm = perm | PTE_U | PTE_P;

	// To avoid overflow, write the condition like this
	// If there really exists overflow, then ULIM check will fail first
	for(addr = (uintptr_t)va; addr - (uintptr_t)va < len; addr += PGSIZE)
	{
		// First do the ULIM Check
		if(addr >= ULIM)
		{
			flag = 1;
			break;
		}
		// Then Check whether the page table entry exists
		if(page_lookup(env->env_pgdir, (void*)addr, &pte_addr) == NULL)
		{
			flag = 1;
			break;
		}
		// Finally do the perm check
		if((*pte_addr & perm) == 0)
		{
			flag = 1;
			break;
		}
	}
	// The first address that makes error must is aligned to page size
	if(flag)
	{
		user_mem_check_addr = (addr == (uintptr_t)va) ? addr : ROUNDDOWN(addr, PGSIZE);	
		return -E_FAULT;
	}
	return 0;
}
```

### 参考

https://pdos.csail.mit.edu/6.828/2018/labs/lab3

https://zhuanlan.zhihu.com/p/48862160