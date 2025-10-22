+++
title = '[6.828] Lab 2: Memory Management'
date = 2024-01-13T00:00:00+00:00
+++

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

> 4、通过 PDE.31~12 找到 PT 的位置，通过 LA.PAGE 去 PT 里找到指定的 PTE（Page Table Entry）。

PTE：

![](https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-10.gif)

> 5、最终 PTE.31~12 + LA.OFFSET 就是 PA。
