
* [Operating System Basic Notes](#operating-system-basic-notes)
	* [GCC 内联汇编](#gcc-内联汇编)
	* [基本概念](#基本概念)
		* [操作系统内核特征](#操作系统内核特征)
		* [操作系统的演变](#操作系统的演变)
	* [启动](#启动)
		* [BIOS](#bios)
		* [启动顺序](#启动顺序)
			* [寄存器](#寄存器)
			* [BIOS](#bios-1)
			* [Bootloader](#bootloader)
				* [标志(lab1/tools/sign.c)](#标志lab1toolssignc)
				* [基本功能](#基本功能)
					* [切换到保护模式, 启动段机制](#切换到保护模式-启动段机制)
					* [从硬盘上加载 某种(kernel in ELF) 格式的 os kernel(在硬盘中紧邻 MBR) 至内存的固定区域](#从硬盘上加载-某种kernel-in-elf-格式的-os-kernel在硬盘中紧邻-mbr-至内存的固定区域)
					* [跳转到 os kernel 的入口点(entry point), 转移控制权至 os](#跳转到-os-kernel-的入口点entry-point-转移控制权至-os)
				* [保护模式与段机制](#保护模式与段机制)
	* [物理内存管理](#物理内存管理)
		* [基本概念](#基本概念-1)
			* [基本目标](#基本目标)
			* [基本管理方式](#基本管理方式)
			* [地址生成](#地址生成)
				* [地址生成时机](#地址生成时机)
			* [地址映射(软硬件结合)](#地址映射软硬件结合)
			* [地址检查(软硬件结合)](#地址检查软硬件结合)
		* [连续内存分配(malloc/free)](#连续内存分配mallocfree)
			* [内存碎片](#内存碎片)
			* [动态分配策略](#动态分配策略)
			* [碎片整理策略](#碎片整理策略)
				* [紧凑(compaction)](#紧凑compaction)
				* [分区对换(swapping in/out)](#分区对换swapping-inout)
			* [malloc 实现策略](#malloc-实现策略)
				* [启发式(Heuristic)编程](#启发式heuristic编程)
				* [伙伴系统(Buddy System)](#伙伴系统buddy-system)
					* [合并空闲块](#合并空闲块)
		* [非连续内存分配](#非连续内存分配)
			* [段式存储管理](#段式存储管理)
			* [页式存储管理](#页式存储管理)
				* [虚拟地址](#虚拟地址)
				* [物理地址](#物理地址)
				* [页表](#页表)
					* [页表结构](#页表结构)
				* [性能问题](#性能问题)
					* [TLB(translation lookaside buffer)](#tlbtranslation-lookaside-buffer)
					* [多级页表](#多级页表)
					* [反置页表](#反置页表)
			* [段页式存储管理](#段页式存储管理)
		* [特权级](#特权级)
			* [特权级检查](#特权级检查)
			* [特权级切换](#特权级切换)
				* [ring 0 to ring 3](#ring-0-to-ring-3)
				* [ring 3 to ring 0 (特权级提升)](#ring-3-to-ring-0-特权级提升)
			* [TSS(Task State Segment)](#tsstask-state-segment)

# Operating System Basic Notes

## GCC 内联汇编

```c
asm ( assembler template	// assembly language
	:=output operands		// 约定输出
	:input operands			// 约定输入
	:clobbers				// 约定插入
);
```

constraints:

-   m/v/o = memory
-   r = register
-   Q = ea/b/c/dx
-   a = eax
-   b = ebx
-   c = ecx
-   d = edx
-   D = edi
-   S = esi
-   0/n = fisrt/nth constraints

## 基本概念

### 操作系统内核特征

-   并发
-   共享: 共享内存, 互斥共享
-   虚拟
-   异步

### 操作系统的演变

单用户系统(45-55) -> 批处理系统(55-65) -> 多道系统(65-80) -> 分时系统(70-) -> 分布式系统

## 启动

### BIOS

-   基本输入输出程序
-   系统设置信息
-   开机后自检程序
-   系统自启动程序

### 启动顺序

#### 寄存器

-   CF: 初值 F000H
-   EIP: 初值 FFF0H

(FFF)F0000H+FFF0H = FFFFFFF0H, BIOS 的 EPROM(Erasable Programmable Read Only Memory) 处
加电后第一条指令一般是 ljmp(实模式下, 内存 !MB), 跳转地址为 CF<<4+EIP, 跳转至 BIOS 例行程序起始点.

#### BIOS

BIOS 根据设置(硬盘/U盘/网络启动), 加载存储设备的主引导扇区(Master Boot Record)(第一个扇区)的 512 字节至内存 0x7c00 处, 开始执行第一条指令(bootloader)

#### Bootloader

##### 标志(lab1/tools/sign.c)

-    有效字节小于 510 bytes
-    结尾为 0x55aa
-    总计字节小于 512 bytes

##### 基本功能

###### 切换到保护模式, 启动段机制

-   开启 A20, 获取足够内存空间
-   置 cr0 保护模式标志位(bit0) 为1
-   加载全局描述符表
-   设置各个通用寄存器与段寄存器

###### 从硬盘上加载 某种(kernel in ELF) 格式的 os kernel(在硬盘中紧邻 MBR) 至内存的固定区域

###### 跳转到 os kernel 的入口点(entry point), 转移控制权至 os

##### 保护模式与段机制

-   CS -> 全局描述符表(其起始地址与表大小位于 gdt 寄存器中)某项(每项存有 base/limit 等信息) -> 局部描述符表 -> 段选择子(段的基本信息) -> 基址+EIP -> 线性地址 ---页机制---> 物理地址
-   将 cr0 寄存器 bit0 置为1, 表示进入了保护模式, 段机制开始起作用

## 物理内存管理

### 基本概念

#### 基本目标

-   抽象: 逻辑地址空间(线性物理地址映射)
-   保护: 独立地址空间(进程间互不影响)
-   共享: 访问相同内存(内核空间与共享库)
-   虚拟化: 独占内存空间假象

#### 基本管理方式

-   重定位(relocation)
-   分段(segmentation): 代码段/数据段
-   分页(paging)
-   虚拟存储(virtual memory): 内存视作硬盘的缓存, 硬盘视作虚拟内存

#### 地址生成

##### 地址生成时机

-    编译时
-    链接时
-    加载时 
-    执行时(相对寻址)

#### 地址映射(软硬件结合)

逻辑地址 ---> 物理地址

-   硬件(CPU/MMU)完成映射地址
-   操作系统建立映射规则(页表)

#### 地址检查(软硬件结合)

-   操作系统设置的段机制和段长度
-   硬件(CPU/MMU)根据段信息进行地址检查(内存访问是否异常)

### 连续内存分配(malloc/free)

#### 内存碎片

-   外部碎片: 已分配单元间无法利用空闲单元(请求内存单元过大)
-   内部碎片: 已分配单元内空闲单元

#### 动态分配策略

-   Fist-fit(最先匹配): 使用第一个可利用空闲块 - 容易产生外部碎片, 分配大块效率低
-   Best-fit(最佳匹配): 使用利用度最高的空闲块 - 外部碎片过小, 释放时需重新排列空闲块链表(升序)
-   Worst-fit(最差匹配): 使用利用度最差的空闲块 - 分配中块效率高, 释放时需重新排列空闲块链表(降序)

#### 碎片整理策略

##### 紧凑(compaction)

将不同进程占用内存单元移至较为集中的地方:

-   只可移动 可动态重定位程序
-   只可移动 等待状态进程

##### 分区对换(swapping in/out)

将等待状态进程的分区对换至外存,以增大可用内存单元

> e.g Linux Swap 分区: 安装系统时一般切割大小为内存大小的50%~100% 的外存作为 Swao 分区

#### malloc 实现策略

##### 启发式(Heuristic)编程

-   建立已分配void指针表,free函数执行时,只回收表中存在的指针;不存在则报错
-   对heap进行分区 - 小/中/大块内存请求,分别从不同区域(8/16/32最小单位区)分配
-   记录当前堆块的信息，如长度，空闲状态
-   记录周围环境信息，如保留上/下一堆块的指针或记录上/下堆块空闲状态

##### 伙伴系统(Buddy System)

-   可分配内存单元总大小为 2^n
-   总是将大小 **小于**请求大小的**2倍**且为**2的幂次方** 的某块内存单元分配出去(大小为 2^(i-1))

###### 合并空闲块

-   空闲块相邻
-   空闲块等大
-   低地址空闲块的 起始地址 必须为 空闲块大小 的 2的幂次方倍(2倍以上)


### 非连续内存分配

-   支持一个程序使用非连续的物理地址空间
-   支持共享代码与数据
-   支持动态加载与动态链接

#### 段式存储管理

将逻辑地址划分, 低位表示段内偏移, 高位(取摸)表示段号(类比缓存中的内存地址)

#### 页式存储管理

##### 虚拟地址

> TLB(translation lookaside buffer in cpu/pm)

-   Virtual Address = 2^(bits of virtual page offset) * virtual page number + virtual page offset
-   VPN(virtual page number point to PPN) - VPO(virtual page offset = PPO)
-   根据 VPN 在页表中找到对应表项(VPN 表示项号), 每项保存着 PPN
-   TLBT(tag) - TLBI(index) - VPO

##### 物理地址

> C(cache) PPO = VPO

-   Page Frame(帧): 高位为帧号, 低位为偏移
-   Physical Address = 2^(bits of physical page offset) * physical page number/page frame number + physical page offset
-   PPN(physical page number) - PPO(physical page offset = VPO)
-   CT(tag) - CI(index) - CO(offset)

##### 页表

-   页表由操作系统建立, 硬件(CPU/MMU)根据页表信息将虚拟地址映射为物理地址

###### 页表结构

-   FN/PPN
-   标志位: resident bit(存在位)/dirty bit(修改位)/reference(clock) bit(引用位)

##### 性能问题
    
-   两次访存: 第一次获取页表项, 第二次访问实际数据
-   页表占据大量内存单元

###### TLB(translation lookaside buffer)

缓存页表项 - key: VPN, value: PPN　不用访问页表

###### 多级页表

切割页表: 建立子页表

> CR3 寄存器: 保存一级页表的基址

-   父页表表项保存子页表起始地址, 逻辑地址某部分保存偏移地址(子表项号)
-   存在位为 0 时, 不用保存子表, 节省内存单元

###### 反置页表

-   PPN 作为页表索引, 页表项保存 VPN(或者 Hash(VPN|PID))
-   将 VPN 映射为 PPN 时, 需遍历整个页表(但此页表只占用少量内存单元)

#### 段页式存储管理

在页式存储管理基础上, 引入段式存储管理

<--- vsn --- vpn --- vpo ---> 映射为 <--- ppn --- ppo --->

> sn: segment number, pn:page number, po: page offset

-   以 vsn 为索引在进程段表中找到段表项, 获取段基址与段大小信息(base limit)
-   以 vpn 为索引在进程页表(页表基址 = 段基址)中找到页表项, 获取 ppn
-   ppn + vpo(ppo) 为实际物理地址

### 特权级

0: 最高特权级, 3: 最低特权级

#### 特权级检查

-   CPL/RPL: 访问者特权级
-   DRL: 段描述符/门描述符(中断/陷阱门)中保存的特权级, 表示被访问段/中断服务/陷阱的特权级
-   CPL/RPL <= DRL e.g 0 < 3

#### 特权级切换

##### ring 0 to ring 3

-   interrupt/trap: push SS(**RPL=3**) -> ESP -> EFLAGS -> CS(**RPL=3**) -> EIP -> Error Code
_   iret: pop above variables, move to ring 3

##### ring 3 to ring 0 (特权级提升)


-   interrupt/trap: stack switch
-   push EFLAGS -> CS(**RPL=0**) -> EIP -> Error Code
-   iret: pop above variables, move to ring 0

#### TSS(Task State Segment)

保存不同特权级的堆栈信息(SS/ESP)

全局描述符表中保存一个 TSS Descriptor(TSS base + TSS limit): allocate TSS memory -> init TSS -> fill TSS descriptor in GDT -> set TSS selector(task register)