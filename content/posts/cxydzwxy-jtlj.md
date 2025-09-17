+++
title = '程序员的自我修养 - 静态链接'
date = 2024-01-11T00:00:00+00:00
+++

原文示例代码，`a.c`：

```c
extern int shared;

void swap(int* a, int* b);

int main()
{
    int a = 100;
    swap(&a, &shared);
}
```

`b.c`：

```c
int shared = 1;

void swap(int* a, int* b)
{
    *a ^= *b ^= *a ^= *b;
}
```

编译 + 链接：

```
/opt/homebrew/bin/i686-elf-gcc -c a.c b.c
/opt/homebrew/bin/i686-elf-ld a.o b.o -e main -o ab
```

段合并采用相似段合并的策略，查看合并前和合并后的 sections：

```
$ /opt/homebrew/bin/i686-elf-objdump -h a.o

a.o:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000039  00000000  00000000  00000034  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  00000000  00000000  0000006d  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  00000000  00000000  0000006d  2**0
                  ALLOC
  3 .comment      00000013  00000000  00000000  0000006d  2**0
                  CONTENTS, READONLY



$ /opt/homebrew/bin/i686-elf-objdump -h b.o

b.o:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000039  00000000  00000000  00000034  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  00000000  00000000  00000070  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  00000000  00000000  00000074  2**0
                  ALLOC
  3 .comment      00000013  00000000  00000000  00000074  2**0
                  CONTENTS, READONLY



$ /opt/homebrew/bin/i686-elf-objdump -h ab

ab:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000072  08048074  08048074  00000074  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  080490e8  080490e8  000000e8  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .comment      00000012  00000000  00000000  000000ec  2**0
                  CONTENTS, READONLY
```

反编译对比：

```
$ /opt/homebrew/bin/i686-elf-objdump -d a.o

a.o:     file format elf32-i386


Disassembly of section .text:

00000000 <main>:
   0:	8d 4c 24 04          	lea    0x4(%esp),%ecx
   4:	83 e4 f0             	and    $0xfffffff0,%esp
   7:	ff 71 fc             	push   -0x4(%ecx)
   a:	55                   	push   %ebp
   b:	89 e5                	mov    %esp,%ebp
   d:	51                   	push   %ecx
   e:	83 ec 14             	sub    $0x14,%esp
  11:	c7 45 f4 64 00 00 00 	movl   $0x64,-0xc(%ebp)
  18:	83 ec 08             	sub    $0x8,%esp
  1b:	68 00 00 00 00       	push   $0x0
  20:	8d 45 f4             	lea    -0xc(%ebp),%eax
  23:	50                   	push   %eax
  24:	e8 fc ff ff ff       	call   25 <main+0x25>
  29:	83 c4 10             	add    $0x10,%esp
  2c:	b8 00 00 00 00       	mov    $0x0,%eax
  31:	8b 4d fc             	mov    -0x4(%ebp),%ecx
  34:	c9                   	leave
  35:	8d 61 fc             	lea    -0x4(%ecx),%esp
  38:	c3                   	ret



$ /opt/homebrew/bin/i686-elf-objdump -d b.o

b.o:     file format elf32-i386


Disassembly of section .text:

00000000 <swap>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	8b 45 08             	mov    0x8(%ebp),%eax
   6:	8b 10                	mov    (%eax),%edx
   8:	8b 45 0c             	mov    0xc(%ebp),%eax
   b:	8b 00                	mov    (%eax),%eax
   d:	31 c2                	xor    %eax,%edx
   f:	8b 45 08             	mov    0x8(%ebp),%eax
  12:	89 10                	mov    %edx,(%eax)
  14:	8b 45 08             	mov    0x8(%ebp),%eax
  17:	8b 10                	mov    (%eax),%edx
  19:	8b 45 0c             	mov    0xc(%ebp),%eax
  1c:	8b 00                	mov    (%eax),%eax
  1e:	31 c2                	xor    %eax,%edx
  20:	8b 45 0c             	mov    0xc(%ebp),%eax
  23:	89 10                	mov    %edx,(%eax)
  25:	8b 45 0c             	mov    0xc(%ebp),%eax
  28:	8b 10                	mov    (%eax),%edx
  2a:	8b 45 08             	mov    0x8(%ebp),%eax
  2d:	8b 00                	mov    (%eax),%eax
  2f:	31 c2                	xor    %eax,%edx
  31:	8b 45 08             	mov    0x8(%ebp),%eax
  34:	89 10                	mov    %edx,(%eax)
  36:	90                   	nop
  37:	5d                   	pop    %ebp
  38:	c3                   	ret



$ /opt/homebrew/bin/i686-elf-objdump -d ab

ab:     file format elf32-i386


Disassembly of section .text:

08048074 <main>:
 8048074:	8d 4c 24 04          	lea    0x4(%esp),%ecx
 8048078:	83 e4 f0             	and    $0xfffffff0,%esp
 804807b:	ff 71 fc             	push   -0x4(%ecx)
 804807e:	55                   	push   %ebp
 804807f:	89 e5                	mov    %esp,%ebp
 8048081:	51                   	push   %ecx
 8048082:	83 ec 14             	sub    $0x14,%esp
 8048085:	c7 45 f4 64 00 00 00 	movl   $0x64,-0xc(%ebp)
 804808c:	83 ec 08             	sub    $0x8,%esp
 804808f:	68 e8 90 04 08       	push   $0x80490e8
 8048094:	8d 45 f4             	lea    -0xc(%ebp),%eax
 8048097:	50                   	push   %eax
 8048098:	e8 10 00 00 00       	call   80480ad <swap>
 804809d:	83 c4 10             	add    $0x10,%esp
 80480a0:	b8 00 00 00 00       	mov    $0x0,%eax
 80480a5:	8b 4d fc             	mov    -0x4(%ebp),%ecx
 80480a8:	c9                   	leave
 80480a9:	8d 61 fc             	lea    -0x4(%ecx),%esp
 80480ac:	c3                   	ret

080480ad <swap>:
 80480ad:	55                   	push   %ebp
 80480ae:	89 e5                	mov    %esp,%ebp
 80480b0:	8b 45 08             	mov    0x8(%ebp),%eax
 80480b3:	8b 10                	mov    (%eax),%edx
 80480b5:	8b 45 0c             	mov    0xc(%ebp),%eax
 80480b8:	8b 00                	mov    (%eax),%eax
 80480ba:	31 c2                	xor    %eax,%edx
 80480bc:	8b 45 08             	mov    0x8(%ebp),%eax
 80480bf:	89 10                	mov    %edx,(%eax)
 80480c1:	8b 45 08             	mov    0x8(%ebp),%eax
 80480c4:	8b 10                	mov    (%eax),%edx
 80480c6:	8b 45 0c             	mov    0xc(%ebp),%eax
 80480c9:	8b 00                	mov    (%eax),%eax
 80480cb:	31 c2                	xor    %eax,%edx
 80480cd:	8b 45 0c             	mov    0xc(%ebp),%eax
 80480d0:	89 10                	mov    %edx,(%eax)
 80480d2:	8b 45 0c             	mov    0xc(%ebp),%eax
 80480d5:	8b 10                	mov    (%eax),%edx
 80480d7:	8b 45 08             	mov    0x8(%ebp),%eax
 80480da:	8b 00                	mov    (%eax),%eax
 80480dc:	31 c2                	xor    %eax,%edx
 80480de:	8b 45 08             	mov    0x8(%ebp),%eax
 80480e1:	89 10                	mov    %edx,(%eax)
 80480e3:	90                   	nop
 80480e4:	5d                   	pop    %ebp
 80480e5:	c3                   	ret
```

可以看到 `a.o` 中调用 `swap()` 的相关汇编为：

```
24:	e8 fc ff ff ff       	call   25 <main+0x25>
```

去 [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) 找下手册 `Intel® 64 and IA-32 Architectures Software Developer's Manual Combined Volumes 2A, 2B, 2C, and  2D: Instruction Set Reference, A- Z`，页码 `Vol. 2A 3-121` 找到 `E8 cd` 对应的是 `CALL rel32`，也就是 4 字节的 `近址相对位移调用指令`，后面地址的小端就是 `0xfffffffc`，-4 其实还是自己的位置，因为此时 `a.o` 并不知道 `swap()` 的确切位置。

TODO
