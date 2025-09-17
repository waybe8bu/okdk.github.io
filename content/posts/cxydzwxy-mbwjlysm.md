+++
title = '程序员的自我修养 - 目标文件里有什么'
date = 2024-01-10T00:00:00+00:00
+++

原文示例代码：

```c
int printf(const char *format, ...);                                                                                                                                                                      
  
int global_init_var = 84; 
int global_uninit_var;

void func1(int i)
{
    printf("%d\n", i); 
}

int main(void)
{
    static int static_var = 85; 
    static int static_var2;

    int a = 1;
    int b;

    func1(static_var + static_var2 + a + b); 

    return a;
}
```

简单起见编译到 x86 arch，mac homebrew 安装 `i686-elf-gcc`/`i686-elf-binutils` 两个，编译：

```
/opt/homebrew/bin/i686-elf-gcc -c 1.c
```

手搓个 js 感受下字节码内部结构的相对位置：

```js
const FS = require('node:fs'),
    TOOLS = {
        read: (hex, offsetBytes, sizeBytes) => {
            return hex.slice(offsetBytes * 2, offsetBytes * 2 + sizeBytes * 2);
        },
        readLE: (hex, offsetBytes, sizeBytes) => {
            return hex.slice(offsetBytes * 2, offsetBytes * 2 + sizeBytes * 2).match(/../g).reverse().join('');
        },
        splitStringByLength: (str, lengthBytes) => {
            const length = lengthBytes * 2;
            return Array.from(
                { length: Math.ceil(str.length / length) }, 
                (v, i) => str.slice(i * length, i * length + length)
            );
        },
        replaceAt: (str, indexBytes, lengthBytes, replacement) => {
            return str.slice(0, indexBytes * 2) + replacement + str.slice((indexBytes + lengthBytes) * 2);
        }
    };

const cnt = FS.readFileSync('./1.o').toString('hex');
if (cnt.slice(0, 8) !== '7f454c46') {
  throw new Error('not elf file');
}
let cntColored = cnt.slice();

// header
const headerSize = parseInt(TOOLS.readLE(cnt, 0x28, 2), 16);
const header = TOOLS.read(cnt, 0, headerSize);
cntColored = TOOLS.replaceAt(cntColored, 0, headerSize, 'G'.repeat(headerSize * 2));

// sections，sht=section header table
const shtOff = parseInt(TOOLS.readLE(cnt, 0x20, 4), 16),
    shtEntrySize = parseInt(TOOLS.readLE(cnt, 0x2E, 2), 16),
    shtEntryNum = parseInt(TOOLS.readLE(cnt, 0x30, 2), 16),
    shtStringTableIndex = parseInt(TOOLS.readLE(cnt, 0x32, 2), 16);
const sht = cnt.slice(shtOff * 2, (shtOff + shtEntrySize * shtEntryNum) * 2);
const shtArr = TOOLS.splitStringByLength(sht, shtEntrySize);
cntColored = TOOLS.replaceAt(cntColored, shtOff, shtEntrySize * shtEntryNum, 'H'.repeat(shtEntrySize * shtEntryNum * 2));
shtArr.forEach((section, index) => {
    const shOffset = parseInt(TOOLS.readLE(section, 0x10, 4), 16),
        shSize = parseInt(TOOLS.readLE(section, 0x14, 4), 16);
    if (shSize === 0) return;
    cntColored = TOOLS.replaceAt(cntColored, shOffset, shSize, String.fromCharCode('I'.charCodeAt(0) + index).repeat(shSize * 2));
});

// replace with color
cntColored = cntColored.replace(/GG+/g, '\x1b[36m' + header + '\x1b[0m'); // cyan
cntColored = cntColored.replace(/HH+/g, '\x1b[32m' + sht + '\x1b[0m'); // green
shtArr.forEach((section, index) => {
    const shOffset = parseInt(TOOLS.readLE(section, 0x10, 4), 16),
        shSize = parseInt(TOOLS.readLE(section, 0x14, 4), 16);
    if (shSize === 0) return;
    cntColored = cntColored.replace(String.fromCharCode('I'.charCodeAt(0) + index).repeat(shSize * 2), 
        '\x1b[35m' + TOOLS.read(cnt, shOffset, shSize) + '\x1b[0m');
});
console.log(cnt);
console.log(cntColored);
```

蓝色为头，绿色是 section header table，紫色是所有的 sections（不同的段没有用颜色区分，因为段有重叠）：

![](/images/1758091659979.png)

主要目的是有个直观的感受，可以看到其实也很简单，除了对齐需要导致的几个字节的浪费，基本利用率也很高。

结构可以去 linux 自带的头文件里找 struct，也可以直接在 [Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 对照。

TODO
