# C Book 摘要
:::success
摘选自泰晓科技整理的[C语言编程透视](https://tinylab.gitbooks.io/cbook/content/)
:::

## VIM用法
indent -kr -gnu -i8 ***

## gcc编译
### 预处理

### 翻译
在编译之前C语言编译器会进行词法分析、语法分析，接着会把源代码翻译成中间语言（汇编语言）
    
:::info
- 可以通过gcc -S 看到这个中间结果
- 诸如 Shell 等解释语言也会经历一个词法分析和语法分析的阶段，不过之后并不会进行“翻译”，而是“解释”，边解释边执行。
:::

语法检查也可以通过gcc来配置 不展开

编译器优化也是在这个阶段 通过 -o 来进行配置。 总体来说是一个trade-off的过程。

### 汇编
汇编实际上还是翻译过程，只不过把作为中间结果的汇编代码翻译成了机器代码，即目标代码
生成可重定位的ELF文件，需要通过ld进行进一步链接成可执行程序和共享库。

ELF文件分为以下几个部分
- LF Header(ELF文件头)
- Program Headers Table(程序头表，实际上叫段表好一些，用于描述可执行文件和可共享库)
- Section 1 2 3 ...
- Section Headers Table(节区头部表，用于链接可重定位文件成可执行文件或共享库)

动态链接库在链接时，库文件本身并没有添加到可执行文件中，只是在可执行文件中加入了该库的名字等信息，以便在可执行文件运行过程中引用库中的函数时由动态连接器去查找相关函数的地址，并调用他们。
:::info 
- 通过 readelf 和 objdump 这两个辅助工具来认识下ELF文件。
    - readelf -h myprintf.o  Header 
    - readelf -l myprintf.o  头部表（段表）
    - readelf -s myprintf.o  节区表
    - readelf -x .data/.bss/.comment/.text myprintf.o
    - objdump -d -j .text myprintf.o  看具体段
:::

### 链接
- [x] 主要讨论静态链接 还没细看

通过修改汇编程序的 return 语句为 `_exit(0)` 和修改程序的入口为 `_start`，我们的代码不链接 gcc 默认链接的那些额外的文件同样可以工作得很好。
:::warning
 :+1:  这个知识还蛮有意思.. 具体可以看看这篇文章[Before main](http://linux.kutx.cn/linux/linux457.htm)
:::



## 程序执行的一刹那
主要介绍命令行接口 /bin/bash

## 