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

```
SECTIONS
{
	/* Link the kernel at this address: "." means the current address */
	. = 0xF0100000;
```

`memlayout.h` 已知：

```
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

TODO
