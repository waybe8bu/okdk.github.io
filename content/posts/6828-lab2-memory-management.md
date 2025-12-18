+++
title = '[6.828] Lab 2: Memory Management'
date = 2024-01-13T00:00:00+00:00
+++

## 原理分析

虚拟地址(VA) -> 线性地址(LA) -> 物理地址(PA)

```
           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical
```

IA-32e 分页机制：

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-12.gif)

一般如 c 语言的指针 CS:IP（[寄存器大小](https://en.wikipedia.org/wiki/X86#32-bit)），实模式下通过 CS<<4+IP 得到 20-bit 的物理地址，而保护模式+开启分页下的寻址过程为：

> 1、段寄存器硬件设计实际有 48-bit，16-bit 的 visible part 和 32-bit 的 invisible part，16-bit 存 segment selector，通过 selector 去 GDT 找 segment descriptor，GDT 的位置由 GDTR 寄存器给出，找完之后 descriptor 会缓存到 invisible part。

selector：

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-6.gif)

descriptor：

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-3.gif)

> 2、descriptor.base + VA.offset 得到 LA。

LA：

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-8.gif)

> 3、CR3 寄存器指定 PD 的位置，通过 LA.DIR 去 PD 里找到指定的 PDE(Page Directory Entry)。

> 4、通过 PDE.31~12 找到 PT 的位置，通过 LA.PAGE 去 PT 里找到指定的 PTE(Page Table Entry)。

PTE：

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-10.gif)

> 5、最终通过 PTE.31~12 找到 page frame，再通过 LA.OFFSET 就能得到 PA。

## 代码分析

### 简化分段机制

`boot.S`，selector 都设置为 0：

```asm
# Set up the important data segment registers (DS, ES, SS).
xorw    %ax,%ax             # Segment number zero
movw    %ax,%ds             # -> Data Segment
movw    %ax,%es             # -> Extra Segment
movw    %ax,%ss             # -> Stack Segment
```

`lgdt gdtdesc` 给 GDTR 指 GDT 的位置： 

```asm
# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULL				# null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg

gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

GDT 第一项默认空段，`mmu.h` 找字节组合的宏：

```c
/*
 * Macros to build GDT entries in assembly.
 */
#define SEG_NULL						\
	.word 0, 0;						\
	.byte 0, 0, 0, 0
#define SEG(type,base,lim)					\
	.word (((lim) >> 12) & 0xffff), ((base) & 0xffff);	\
	.byte (((base) >> 16) & 0xff), (0x90 | (type)),		\
		(0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```

`granularity` 是 1 也就是单位是 page 4KB，`limit` 是 2^20，因此 code/data 段都是指向的 0~4GB。

### 临时页管理

`entry.S`：

```asm
# Load the physical address of entry_pgdir into cr3.  entry_pgdir
# is defined in entrypgdir.c.
movl	$(RELOC(entry_pgdir)), %eax
movl	%eax, %cr3
# Turn on paging.
movl	%cr0, %eax
orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
movl	%eax, %cr0
```

通过这段汇编可以知道 PD 的内容其实就是 `entrypgdir.c` 里 `entry_pgdir` 的位置：

```c
pte_t entry_pgtable[NPTENTRIES];

__attribute__((__aligned__(PGSIZE)))
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};

__attribute__((__aligned__(PGSIZE)))
pte_t entry_pgtable[NPTENTRIES] = {
	0x000000 | PTE_P | PTE_W,
  // 这里省略...
	0x3ff000 | PTE_P | PTE_W,
};
```

`kernel.ld` 已知：

```c
SECTIONS
{
	/* Link the kernel at this address: "." means the current address */
	. = 0xF0100000;
```

`memlayout.h` 已知：

```c
// All physical memory mapped at this address
#define	KERNBASE	0xF0000000
```

lab1 的内存低位分布如下：

```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

因此，很容易看出，通过 "手搓" `entry_pgtable` 的方式，直接将 PT 硬编码固定映射到 PA 0~4MB 的区间，由于 `KERNBASE` 的 VA 是 `0xF0000000`，为了避开 IO hole，link address 选在了 `0xF0100000`。

之所以有这个临时的页管理，是因为 `entry.S` 部分执行完分页就已经开启了，后续 c 语言一起 link 进来的代码也已经是 VA，但是分页真正的动态管理初始化代码还没有执行完，所以就需要一个 tiny 且固定的 PD/PT 暂时可用，这样在 `entry.S` 执行完到动态页管理完全生效之前的这部分 c 代码才能正常执行。

### 正式页管理

`entry.S` 第一个调用的 c 方法是：

```asm
# now to C code
call	i386_init
```

```c
void
i386_init(void)
{
	extern char edata[], end[];

	// Before doing anything else, complete the ELF loading process.
	// Clear the uninitialized global data (BSS) section of our program.
	// This ensures that all static/global variables start out zero.
	memset(edata, 0, end - edata);

	// Initialize the console.
	// Can't call cprintf until after we do this!
	cons_init();

	cprintf("6828 decimal is %o octal!\n", 6828);

	// Lab 2 memory management initialization functions
	mem_init();

	// Drop into the kernel monitor.
	while (1)
		monitor(NULL);
}
```

接着：

```c
// Set up a two-level page table:
//    kern_pgdir is its linear (virtual) address of the root
//
// This function only sets up the kernel part of the address space
// (ie. addresses >= UTOP).  The user part of the address space
// will be set up later.
//
// From UTOP to ULIM, the user is allowed to read but not write.
// Above ULIM the user cannot read or write.
void
mem_init(void)
{
	uint32_t cr0;
	size_t n;

	// Find out how much memory the machine has (npages & npages_basemem).
	i386_detect_memory();

	// Remove this line when you're ready to test this function.
	panic("mem_init: This function is not finished\n");

	//////////////////////////////////////////////////////////////////////
	// create initial page directory.
	kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
	memset(kern_pgdir, 0, PGSIZE);

	//////////////////////////////////////////////////////////////////////
	// Recursively insert PD in itself as a page table, to form
	// a virtual page table at virtual address UVPT.
	// (For now, you don't have understand the greater purpose of the
	// following line.)

	// Permissions: kernel R, user R
	kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;

	//////////////////////////////////////////////////////////////////////
	// Allocate an array of npages 'struct PageInfo's and store it in 'pages'.
	// The kernel uses this array to keep track of physical pages: for
	// each physical page, there is a corresponding struct PageInfo in this
	// array.  'npages' is the number of physical pages in memory.  Use memset
	// to initialize all fields of each struct PageInfo to 0.
	// Your code goes here:


	//////////////////////////////////////////////////////////////////////
	// Now that we've allocated the initial kernel data structures, we set
	// up the list of free physical pages. Once we've done so, all further
	// memory management will go through the page_* functions. In
	// particular, we can now map memory using boot_map_region
	// or page_insert
	page_init();

	check_page_free_list(1);
	check_page_alloc();
	check_page();

	//////////////////////////////////////////////////////////////////////
	// Now we set up virtual memory

	//////////////////////////////////////////////////////////////////////
	// Map 'pages' read-only by the user at linear address UPAGES
	// Permissions:
	//    - the new image at UPAGES -- kernel R, user R
	//      (ie. perm = PTE_U | PTE_P)
	//    - pages itself -- kernel RW, user NONE
	// Your code goes here:

	//////////////////////////////////////////////////////////////////////
	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	//     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
	//       the kernel overflows its stack, it will fault rather than
	//       overwrite memory.  Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	// Your code goes here:

	//////////////////////////////////////////////////////////////////////
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	// We might not have 2^32 - KERNBASE bytes of physical memory, but
	// we just set up the mapping anyway.
	// Permissions: kernel RW, user NONE
	// Your code goes here:

	// Check that the initial page directory has been set up correctly.
	check_kern_pgdir();

	// Switch from the minimal entry page directory to the full kern_pgdir
	// page table we just created.	Our instruction pointer should be
	// somewhere between KERNBASE and KERNBASE+4MB right now, which is
	// mapped the same way by both page tables.
	//
	// If the machine reboots at this point, you've probably set up your
	// kern_pgdir wrong.
	lcr3(PADDR(kern_pgdir));

	check_page_free_list(0);

	// entry.S set the really important flags in cr0 (including enabling
	// paging).  Here we configure the rest of the flags that we care about.
	cr0 = rcr0();
	cr0 |= CR0_PE|CR0_PG|CR0_AM|CR0_WP|CR0_NE|CR0_MP;
	cr0 &= ~(CR0_TS|CR0_EM);
	lcr0(cr0);

	// Some more checks, only possible after kern_pgdir is installed.
	check_page_installed_pgdir();
}
```

TODO
