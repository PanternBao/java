cpu 多线程

分时

内存

如何将有限的物理内存分配给多个程序？

1. 地址空间不隔离：多个程序可以访问整个物理空间，安全性问题
2. 内存使用率低
3. 程序运行的地址不确定

虚拟内存地址 - 隔离

如何转换？os + 硬件（MMU，memory management unit）

分段：地址空间按照段来划分

分页：地址空间按照页来划分

物理地址空间





编译：

- 预处理
- 编译
- 汇编
- 链接



汇编

链接：

模块化

如何跨模块访问函数

如何跨模块访问变量





##目标文件的格式

目前，PC平台流行的 **可执行文件格式（Executable）** 主要包含如下两种，它们都是 **COFF（Common File Format）** 格式的变种。

- Windows下的 **PE（Portable Executable）**
- Linux下的 **ELF（Executable Linkable Format）**

**目标文件就是源代码经过编译后但未进行连接的那些中间文件（Windows的`.obj`和Linux的`.o`），它与可执行文件的格式非常相似，所以一般跟可执行文件格式一起采用同一种格式存储**。在Windows下采用**PE-COFF**文件格式；Linux下采用**ELF**文件格式。

事实上，除了**可执行文件**外，**动态链接库（DDL，Dynamic Linking Library）**、**静态链接库（Static Linking Library）** 均采用可执行文件格式存储。它们在Window下均按照PE-COFF格式存储；Linux下均按照ELF格式存储。只是文件名后缀不同而已。

- 动态链接库：Windows的`.dll`、Linux的`.so`
- 静态链接库：Windows的`.lib`、Linux的`.a`

### ELF文件结构

#### ELF Header

```shell
[root@72791a2eaff9 baofengqi]# readelf -h simpleSession.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          400 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         13
  Section header string table index: 10
```

其中关注几个字段：

```java
// 表示 ELF Header 的大小
Size of this header:               64 (bytes)

Start of section headers:          400 (bytes into file)
Size of section headers:           64 (bytes)
Number of section headers:         13
Section header string table index: 10
```



ELF Section Header Table

可以通过下面的命令查看所有的section：

```shell
># readelf -S simpleSession.o
```

.rela.text 重定位

ELF 节头表是一个节头数组。每一个节头都描述了其所对应的节的信息，如节名、节大小、在文件中的偏移、读写权限等。**编译器、链接器、装载器都是通过节头表来定位和访问各个节的属性的。**

ELF Sections

可重定位文件 。o

可执行文件

共享目标文件 。so

核心转储文件



目标文件组成：

程序段：

1. 运行时数据
2. 静态数据

代码段：

函数名，字段名怎么存储？

两步链接：

空间与地址分配

符号解析与重定位