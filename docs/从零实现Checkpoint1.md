# 从零实现 Checkpoint 1 - 完全手把手指南

## 🎯 学习目标

不是简单地看懂代码，而是：

1. **理解每一行代码为什么这样写**
2. **自己从空白文件开始实现**
3. **遇到问题自己调试解决**
4. **达到可以给别人讲解的水平**

---

## 📋 实现路线图

```
第1天: 准备工作 + 理论学习
  ↓
第2天: 创建自己的实现分支
  ↓
第3天: 实现IDT框架
  ↓
第4天: 实现异常处理（汇编部分）
  ↓
第5天: 实现异常处理（C语言部分）
  ↓
第6-7天: 实现分页系统
  ↓
第8天: 测试和调试
  ↓
第9天: 深入理解和文档
```

---

## 🗓️ 第一天：准备工作 + 理论学习

### 上午：理解整体架构（3小时）

#### 1. 阅读 OSTEP（必读）

```bash
# 打开浏览器，阅读以下章节
https://pages.cs.wisc.edu/~remzi/OSTEP/

重点章节：
- Chapter 18: Introduction to Paging (30分钟)
- Chapter 19: Translation Lookaside Buffers (30分钟)
- Chapter 20: Advanced Page Tables (30分钟)
```

**做笔记**：用自己的话总结

- 什么是分页？
- 虚拟地址如何转换为物理地址？
- 为什么需要两级页表？

#### 2. 画出系统启动流程图（30分钟）

在纸上或白板上画出：

```
BIOS → GRUB → boot.S → kernel.c → ?
```

思考：

- GRUB做了什么？
- boot.S做了什么？
- kernel.c的entry函数何时被调用？

#### 3. 理解x86保护模式（1小时）

阅读我创建的学习指南：

```bash
open docs/Checkpoint1-学习指南.md
```

重点理解：

- 什么是GDT？
- 什么是IDT？
- 中断门 vs 陷阱门
- 页目录 vs 页表

### 下午：分析现有代码（3-4小时）

#### 任务1：逐行阅读现有实现

打开以下文件，逐行阅读并做注释：

1. **idt_init.c** (约105行)
    - 理解 `initialize_IDT()` 做了什么
    - 理解为什么异常用陷阱门
    - 理解为什么系统调用DPL=3

2. **exception_handler.c** (约6KB)
    - 理解异常处理流程
    - 理解为什么需要打印寄存器

3. **interrupt_handler.S** (约4.5KB)
    - 理解汇编包装函数的作用
    - 理解栈的状态变化

4. **paging.c** (约11KB)
    - 理解页目录和页表的设置
    - 理解为什么视频内存用4KB页
    - 理解为什么内核用4MB页

#### 任务2：画出数据结构图

在纸上画出：

**IDT结构**：

```
IDT (256项)
├─ IDT[0]:  除零异常     → EXCEPTION_0 函数地址
├─ IDT[1]:  调试异常     → EXCEPTION_1 函数地址
├─ ...
├─ IDT[14]: 页错误       → EXCEPTION_14 函数地址
├─ ...
├─ IDT[33]: 键盘中断     → KB_handler 函数地址
├─ ...
└─ IDT[128]: 系统调用    → syscall 函数地址
```

**分页结构**：

```
虚拟地址 0xB8000 (视频内存)
         ↓
DIR=0, TABLE=184, OFFSET=0
         ↓
PageDirectory[0] → PageTable
         ↓
PageTable[184] → 物理地址 0xB8000
```

#### 任务3：理解关键概念

回答以下问题（写在笔记本上）：

1. IDT表项的8个字节是如何组成的？
2. 为什么异常处理程序需要汇编包装？
3. 为什么页目录和页表需要4KB对齐？
4. CR0、CR2、CR3寄存器分别存储什么？
5. 什么时候需要刷新TLB？

### 晚上：准备开发环境（1小时）

#### 1. 创建工作分支

```bash
cd /Users/miles/code/Linux-System-Design
git checkout -b my-checkpoint1-implementation
```

#### 2. 备份现有实现

```bash
cd student-distrib
mkdir -p ../backup-original
cp idt_init.c ../backup-original/
cp exception_handler.c ../backup-original/
cp interrupt_handler.S ../backup-original/
cp paging.c ../backup-original/
```

#### 3. 准备笔记文档

```bash
cd ../docs
touch my-checkpoint1-notes.md
```

在笔记中写下：

- 今天学到了什么？
- 还有哪些不理解的？
- 明天的计划是什么？

---

## 🗓️ 第二天：创建自己的实现框架

### 目标

创建空白文件框架，只保留必要的声明，准备从零开始实现。

### 上午：清理IDT相关代码（2小时）

#### 步骤1：保留idt_init.h的声明

```bash
cd student-distrib
```

编辑 `idt_init.h`，确保有以下声明：

```c
#ifndef _IDT_INIT_H
#define _IDT_INIT_H

void initialize_IDT();
void set_exceptions();
void set_interrupts();

#endif
```

#### 步骤2：清空idt_init.c，从头实现

创建 `idt_init_my.c`：

```bash
touch idt_init_my.c
```

写入基本框架：

```c
#include "idt_init.h"
#include "x86_desc.h"
#include "lib.h"

/* TODO: 外部声明异常处理函数 */


/* TODO: 实现 initialize_IDT() */
void initialize_IDT() {
    // 第一步：初始化所有IDT表项为默认值

    // 第二步：设置异常处理程序

    // 第三步：设置中断处理程序

    // 第四步：设置系统调用

    // 第五步：加载IDT
}

/* TODO: 实现 set_exceptions() */
void set_exceptions() {
    // 设置20个异常处理程序
}

/* TODO: 实现 set_interrupts() */
void set_interrupts() {
    // 设置硬件中断处理程序
}
```

### 下午：清理异常处理代码（2小时）

#### 步骤1：创建exception_handler_my.h

```bash
touch exception_handler_my.h
```

写入：

```c
#ifndef _EXCEPTION_HANDLER_MY_H
#define _EXCEPTION_HANDLER_MY_H

#include "types.h"

/* TODO: 定义寄存器上下文结构 */
typedef struct registers {
    // 补充字段
} registers_t;

/* 异常处理函数声明 */
void exception_handler(registers_t* regs);

#endif
```

#### 步骤2：创建exception_handler_my.c

```bash
touch exception_handler_my.c
```

写入框架：

```c
#include "exception_handler_my.h"
#include "lib.h"

/* TODO: 异常名称数组 */
static const char* exception_names[20] = {
    // 补充异常名称
};

/* TODO: 实现异常处理函数 */
void exception_handler(registers_t* regs) {
    // 1. 获取异常号和错误码

    // 2. 清屏并打印异常信息

    // 3. 特殊处理页错误（读CR2）

    // 4. 打印寄存器状态

    // 5. 停止系统
}
```

#### 步骤3：创建interrupt_handler_my.S

```bash
touch interrupt_handler_my.S
```

写入框架：

```asm
# interrupt_handler_my.S
# 异常和中断的汇编包装函数

.text

# TODO: 导出符号
.globl EXCEPTION_0

# TODO: 定义宏


# TODO: 定义所有异常处理入口


# TODO: 公共异常处理代码
exception_common:
    # 保存寄存器

    # 设置内核数据段

    # 调用C函数

    # 恢复寄存器

    # 返回
```

### 晚上：清理分页代码（2小时）

#### 步骤1：创建paging_my.h

```bash
touch paging_my.h
```

写入框架：

```c
#ifndef _PAGING_MY_H
#define _PAGING_MY_H

#include "types.h"

/* TODO: 定义常量 */
#define NUM_PDE         1024
#define NUM_PTE         1024
// ...

/* TODO: 定义PDE结构 */
typedef struct pde_4kb {
    // 补充位字段
} __attribute__((packed)) pde_4kb_t;

/* TODO: 定义PTE结构 */

/* TODO: 定义页目录和页表结构 */

/* 函数声明 */
void init_paging();
void set_up_PD_PT();
void enable_paging();
void flush_TLB();

#endif
```

#### 步骤2：创建paging_my.c

```bash
touch paging_my.c
```

写入框架：

```c
#include "paging_my.h"
#include "lib.h"

/* TODO: 声明全局变量：页目录和页表 */


/* TODO: 实现 init_paging() */
void init_paging() {
    // 1. 设置页目录和页表

    // 2. 启用分页
}

/* TODO: 实现 set_up_PD_PT() */
void set_up_PD_PT() {
    // 1. 初始化所有页目录项为"不存在"

    // 2. 初始化所有页表项为"不存在"

    // 3. 设置页目录第0项（0-4MB）

    // 4. 设置页表，映射视频内存

    // 5. 设置页目录第1项（4-8MB，内核）
}

/* TODO: 实现 enable_paging() */
void enable_paging() {
    // 1. 加载CR3

    // 2. 设置CR0的PG位
}

/* TODO: 实现 flush_TLB() */
void flush_TLB() {
    // 重新加载CR3
}
```

---

## 🗓️ 第三天：实现IDT初始化

### 目标

完成 `idt_init_my.c` 的实现，能够正确初始化IDT表。

### 上午：实现initialize_IDT函数（3小时）

#### 任务1：理解IDT表项结构

查看 `x86_desc.h` 中的定义：

```c
typedef union idt_desc_t {
    uint32_t val[2];
    struct {
        uint16_t offset_15_00;
        uint16_t seg_selector;
        uint8_t  reserved4;
        uint8_t  reserved3 : 1;
        uint8_t  reserved2 : 1;
        uint8_t  reserved1 : 1;
        uint8_t  size      : 1;
        uint8_t  reserved0 : 1;
        uint8_t  dpl       : 2;
        uint8_t  present   : 1;
        uint16_t offset_31_16;
    } __attribute__((packed));
} idt_desc_t;

extern idt_desc_t idt[NUM_VEC];
```

#### 任务2：理解每个字段的含义

在纸上写下：

```
offset_15_00:  处理函数地址的低16位
seg_selector:  段选择子（KERNEL_CS = 0x0010）
reserved4:     必须为0
reserved3:     0=中断门，1=陷阱门
reserved2:     必须为1
reserved1:     必须为1
size:          1=32位门
reserved0:     必须为0
dpl:           特权级（0-3）
present:       1=有效
offset_31_16:  处理函数地址的高16位
```

#### 任务3：实现初始化循环

填写 `idt_init_my.c`：

```c
void initialize_IDT() {
    int i;

    /* 第一步：初始化所有256个IDT表项 */
    for (i = 0; i < NUM_VEC; i++) {
        idt[i].seg_selector = KERNEL_CS;   // 问自己：为什么是KERNEL_CS？
        idt[i].reserved4 = 0;
        idt[i].size = 1;                   // 问自己：为什么是1？
        idt[i].reserved3 = 0;              // 默认：中断门
        idt[i].reserved2 = 1;
        idt[i].reserved1 = 1;
        idt[i].reserved0 = 0;
        idt[i].dpl = 0;                    // 问自己：为什么是0？
        idt[i].present = 1;

        /* 前32个是异常，使用陷阱门 */
        if (i < 32) {
            idt[i].reserved3 = 1;          // 问自己：为什么异常用陷阱门？
        }

        /* 0x80是系统调用 */
        if (i == 0x80) {
            idt[i].reserved3 = 1;          // 陷阱门
            idt[i].dpl = 3;                // 问自己：为什么DPL=3？
        }
    }

    /* 第二步：设置异常处理程序 */
    set_exceptions();

    /* 第三步：设置中断处理程序 */
    set_interrupts();

    /* 第四步：设置系统调用 */
    SET_IDT_ENTRY(idt[0x80], syscall);
}
```

**理解检查点**：

- 为什么seg_selector要设为KERNEL_CS？
  答：因为处理程序在内核代码段中

- 为什么异常用陷阱门（reserved3=1）？
  答：陷阱门不会自动关中断，允许嵌套异常

- 为什么系统调用DPL=3？
  答：允许用户态（Ring 3）调用

### 下午：实现set_exceptions函数（2小时）

#### 任务1：声明外部函数

在 `idt_init_my.c` 顶部添加：

```c
/* 外部声明异常处理函数（在interrupt_handler_my.S中定义） */
extern void EXCEPTION_0();
extern void EXCEPTION_1();
extern void EXCEPTION_2();
extern void EXCEPTION_3();
extern void EXCEPTION_4();
extern void EXCEPTION_5();
extern void EXCEPTION_6();
extern void EXCEPTION_7();
extern void EXCEPTION_8();
extern void EXCEPTION_9();
extern void EXCEPTION_10();
extern void EXCEPTION_11();
extern void EXCEPTION_12();
extern void EXCEPTION_13();
extern void EXCEPTION_14();
extern void EXCEPTION_16();
extern void EXCEPTION_17();
extern void EXCEPTION_18();
extern void EXCEPTION_19();
```

**问自己**：为什么没有EXCEPTION_15？
答：Intel保留了15号异常，不使用

#### 任务2：实现set_exceptions

```c
void set_exceptions() {
    /* 设置0-19号异常（跳过15） */
    SET_IDT_ENTRY(idt[0],  EXCEPTION_0);
    SET_IDT_ENTRY(idt[1],  EXCEPTION_1);
    SET_IDT_ENTRY(idt[2],  EXCEPTION_2);
    SET_IDT_ENTRY(idt[3],  EXCEPTION_3);
    SET_IDT_ENTRY(idt[4],  EXCEPTION_4);
    SET_IDT_ENTRY(idt[5],  EXCEPTION_5);
    SET_IDT_ENTRY(idt[6],  EXCEPTION_6);
    SET_IDT_ENTRY(idt[7],  EXCEPTION_7);
    SET_IDT_ENTRY(idt[8],  EXCEPTION_8);
    SET_IDT_ENTRY(idt[9],  EXCEPTION_9);
    SET_IDT_ENTRY(idt[10], EXCEPTION_10);
    SET_IDT_ENTRY(idt[11], EXCEPTION_11);
    SET_IDT_ENTRY(idt[12], EXCEPTION_12);
    SET_IDT_ENTRY(idt[13], EXCEPTION_13);
    SET_IDT_ENTRY(idt[14], EXCEPTION_14);
    // 15是保留的
    SET_IDT_ENTRY(idt[16], EXCEPTION_16);
    SET_IDT_ENTRY(idt[17], EXCEPTION_17);
    SET_IDT_ENTRY(idt[18], EXCEPTION_18);
    SET_IDT_ENTRY(idt[19], EXCEPTION_19);

    /* 20-31设为保留（复用EXCEPTION_1） */
    int i;
    for (i = 20; i < 32; i++) {
        if (i != 15) {
            SET_IDT_ENTRY(idt[i], EXCEPTION_1);
        }
    }
}
```

#### 任务3：理解SET_IDT_ENTRY宏

查看 `x86_desc.h` 中的定义：

```c
#define SET_IDT_ENTRY(str, handler)          \
do {                                          \
    str.offset_15_00 = ADDR_LOW(handler);    \
    str.offset_31_16 = ADDR_HIGH(handler);   \
} while(0)
```

**理解**：

- `ADDR_LOW(handler)`：取函数地址的低16位
- `ADDR_HIGH(handler)`：取函数地址的高16位

**问自己**：为什么要分成两部分？
答：IDT表项结构要求，32位地址分散存储

### 晚上：实现set_interrupts函数（1小时）

```c
/* 外部声明中断处理函数 */
extern void KB_handler();
extern void RTC_handler();
extern void PIT_handler();
extern void syscall();

void set_interrupts() {
    SET_IDT_ENTRY(idt[32], PIT_handler);   // IRQ 0 → INT 32
    SET_IDT_ENTRY(idt[33], KB_handler);    // IRQ 1 → INT 33
    SET_IDT_ENTRY(idt[40], RTC_handler);   // IRQ 8 → INT 40
}
```

**理解**：

- IRQ号 + 32 = 中断向量号
- IRQ 0（PIT）→ INT 32
- IRQ 1（键盘）→ INT 33
- IRQ 8（RTC）→ INT 40

---

## 🗓️ 第四天：实现异常处理（汇编部分）

### 目标

完成 `interrupt_handler_my.S`，实现异常包装函数。

### 上午：理解汇编包装的作用（2小时）

#### 任务1：理解为什么需要汇编包装

**问题**：为什么不能直接让IDT指向C函数？

**答案**：

1. C函数不知道如何处理CPU压入的错误码
2. 需要保存所有寄存器
3. 需要构造统一的栈帧给C函数

**CPU自动做的事**：

```
发生异常时，CPU自动：
1. 如果特权级改变（用户态→内核态）：
   - 从TSS获取内核栈地址（SS0:ESP0）
   - 压入SS
   - 压入ESP
2. 压入EFLAGS
3. 压入CS
4. 压入EIP
5. 如果有错误码，压入错误码
6. 跳转到IDT中指定的处理函数
```

**我们需要做的事**：

```
1. 对于没有错误码的异常，压入假错误码（保持栈对齐）
2. 压入异常号
3. 保存所有寄存器
4. 设置内核数据段
5. 调用C函数
6. 恢复所有寄存器
7. 清理栈
8. IRET返回
```

#### 任务2：画出栈的状态变化

在纸上画出：

```
异常发生前（用户态）：
┌─────────────┐
│  用户栈     │
└─────────────┘

CPU自动压入：
┌─────────────┐ ← ESP
│   EIP       │
│   CS        │
│   EFLAGS    │
│   ESP(old)  │ （如果特权级改变）
│   SS(old)   │ （如果特权级改变）
└─────────────┘

有错误码的异常，CPU还会压入：
│ 错误码      │

我们的汇编代码压入：
│ 异常号      │
│ DS          │
│ ES          │
│ FS          │
│ GS          │
│ EDI         │
│ ESI         │
│ EBP         │
│ ESP         │
│ EBX         │
│ EDX         │
│ ECX         │
│ EAX         │
└─────────────┘ ← 传给C函数的指针
```

### 下午：实现汇编宏（3小时）

#### 步骤1：定义KERNEL_DS常量

在 `interrupt_handler_my.S` 顶部：

```asm
# interrupt_handler_my.S

.text

/* 导出符号 */
.globl EXCEPTION_0, EXCEPTION_1, EXCEPTION_2, EXCEPTION_3
.globl EXCEPTION_4, EXCEPTION_5, EXCEPTION_6, EXCEPTION_7
.globl EXCEPTION_8, EXCEPTION_9, EXCEPTION_10, EXCEPTION_11
.globl EXCEPTION_12, EXCEPTION_13, EXCEPTION_14, EXCEPTION_16
.globl EXCEPTION_17, EXCEPTION_18, EXCEPTION_19

/* 常量定义 */
.equ KERNEL_DS, 0x0018
```

**问自己**：为什么KERNEL_DS = 0x0018？
答：查看GDT，第3项（索引=3，3×8=0x18）是内核数据段

#### 步骤2：定义无错误码的异常宏

```asm
/*
 * 宏: 没有错误码的异常
 * 参数: num - 异常号
 */
.macro EXCEPTION_NO_ERROR_CODE num
EXCEPTION_\num:
    pushl $0                    /* 压入假错误码（保持栈对齐）*/
    pushl $\num                 /* 压入异常号 */
    jmp exception_common        /* 跳转到公共处理代码 */
.endm
```

**理解**：

- `\num`：宏参数，会被替换为实际数字
- `EXCEPTION_\num:`：生成标签，如EXCEPTION_0:
- `pushl $0`：压入立即数0（假错误码）
- `pushl $\num`：压入异常号
- `jmp exception_common`：跳转到公共代码

#### 步骤3：定义有错误码的异常宏

```asm
/*
 * 宏: 有错误码的异常
 * 参数: num - 异常号
 * 注意: CPU已经压入错误码，我们不需要压入假错误码
 */
.macro EXCEPTION_WITH_ERROR_CODE num
EXCEPTION_\num:
    /* CPU已经压入错误码，直接压入异常号 */
    pushl $\num                 /* 压入异常号 */
    jmp exception_common        /* 跳转到公共处理代码 */
.endm
```

**问自己**：哪些异常有错误码？
答：8, 10, 11, 12, 13, 14, 17

#### 步骤4：使用宏定义所有异常入口

```asm
/* 定义所有异常处理入口 */
EXCEPTION_NO_ERROR_CODE 0       /* 除零错误 */
EXCEPTION_NO_ERROR_CODE 1       /* 调试异常 */
EXCEPTION_NO_ERROR_CODE 2       /* NMI */
EXCEPTION_NO_ERROR_CODE 3       /* 断点 */
EXCEPTION_NO_ERROR_CODE 4       /* 溢出 */
EXCEPTION_NO_ERROR_CODE 5       /* 越界 */
EXCEPTION_NO_ERROR_CODE 6       /* 无效操作码 */
EXCEPTION_NO_ERROR_CODE 7       /* 设备不可用 */
EXCEPTION_WITH_ERROR_CODE 8     /* 双重错误（有错误码）*/
EXCEPTION_NO_ERROR_CODE 9       /* 协处理器段溢出 */
EXCEPTION_WITH_ERROR_CODE 10    /* 无效TSS（有错误码）*/
EXCEPTION_WITH_ERROR_CODE 11    /* 段不存在（有错误码）*/
EXCEPTION_WITH_ERROR_CODE 12    /* 栈段错误（有错误码）*/
EXCEPTION_WITH_ERROR_CODE 13    /* 通用保护错误（有错误码）*/
EXCEPTION_WITH_ERROR_CODE 14    /* 页错误（有错误码）*/
EXCEPTION_NO_ERROR_CODE 16      /* x87 FPU错误 */
EXCEPTION_WITH_ERROR_CODE 17    /* 对齐检查（有错误码）*/
EXCEPTION_NO_ERROR_CODE 18      /* 机器检查 */
EXCEPTION_NO_ERROR_CODE 19      /* SIMD浮点异常 */
```

### 晚上：实现公共处理代码（2小时）

#### 步骤1：保存寄存器

```asm
/*
 * exception_common
 * 公共异常处理代码
 * 此时栈的状态:
 *   [ESP+0]:  异常号
 *   [ESP+4]:  错误码（或假错误码0）
 *   [ESP+8]:  EIP（CPU压入）
 *   [ESP+12]: CS（CPU压入）
 *   [ESP+16]: EFLAGS（CPU压入）
 */
exception_common:
    /* 保存段寄存器 */
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs

    /* 保存通用寄存器（PUSHAL指令）*/
    pushal              /* 依次压入: EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI */
```

**问自己**：PUSHAL压入寄存器的顺序是什么？
答：EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI

#### 步骤2：设置内核数据段

```asm
    /* 设置内核数据段 */
    movw $KERNEL_DS, %ax    /* 加载KERNEL_DS到AX */
    movw %ax, %ds           /* 设置DS */
    movw %ax, %es           /* 设置ES */
    movw %ax, %fs           /* 设置FS */
    movw %ax, %gs           /* 设置GS */
```

**问自己**：为什么要设置数据段？
答：确保我们访问的是内核数据

#### 步骤3：调用C函数

```asm
    /* 调用C语言异常处理函数 */
    pushl %esp              /* 传递寄存器上下文指针 */
    call exception_handler  /* 调用C函数 */
    addl $4, %esp           /* 清理参数（4字节）*/
```

**理解**：

- `pushl %esp`：将当前ESP（指向寄存器结构）作为参数传递
- `call exception_handler`：调用C函数
- `addl $4, %esp`：清理参数（因为我们压入了4字节）

#### 步骤4：恢复寄存器并返回

```asm
    /* 恢复所有寄存器 */
    popal               /* 恢复通用寄存器 */
    popl %gs
    popl %fs
    popl %es
    popl %ds

    /* 跳过异常号和错误码（共8字节）*/
    addl $8, %esp

    /* 返回（IRET会弹出EIP, CS, EFLAGS, ESP, SS）*/
    iret
```

**问自己**：为什么要跳过8字节？
答：跳过我们压入的异常号（4字节）和错误码（4字节）

#### 步骤5：临时的中断和系统调用处理

```asm
/* 临时的中断处理程序（Checkpoint 2会实现）*/
KB_handler:
    iret

RTC_handler:
    iret

PIT_handler:
    iret

syscall:
    iret
```

---

## 🗓️ 第五天：实现异常处理（C语言部分）

### 目标

完成 `exception_handler_my.c`，实现异常信息打印。

### 上午：定义数据结构（2小时）

#### 步骤1：定义寄存器结构

编辑 `exception_handler_my.h`：

```c
#ifndef _EXCEPTION_HANDLER_MY_H
#define _EXCEPTION_HANDLER_MY_H

#include "types.h"

/* 寄存器上下文结构 */
typedef struct registers {
    /* 段寄存器（我们在汇编中压入的）*/
    uint32_t gs;
    uint32_t fs;
    uint32_t es;
    uint32_t ds;

    /* 通用寄存器（PUSHAL压入的顺序）*/
    uint32_t edi;
    uint32_t esi;
    uint32_t ebp;
    uint32_t esp;
    uint32_t ebx;
    uint32_t edx;
    uint32_t ecx;
    uint32_t eax;

    /* 中断号和错误码（我们在汇编中压入的）*/
    uint32_t int_no;      /* 异常号 */
    uint32_t err_code;    /* 错误码 */

    /* CPU自动压入的 */
    uint32_t eip;
    uint32_t cs;
    uint32_t eflags;
    uint32_t useresp;     /* 用户栈ESP（如果特权级改变）*/
    uint32_t ss;          /* 用户栈SS（如果特权级改变）*/
} registers_t;

/* 异常处理函数 */
void exception_handler(registers_t* regs);

#endif
```

**理解检查**：

- 为什么字段顺序很重要？
  答：必须与汇编中压栈的顺序一致

- 为什么ESP会被PUSHAL压入两次？
  答：PUSHAL会压入压栈前的ESP值

#### 步骤2：定义异常名称数组

编辑 `exception_handler_my.c`：

```c
#include "exception_handler_my.h"
#include "lib.h"

/* 异常名称数组 */
static const char* exception_names[20] = {
    "Divide Error",                    // 0
    "Debug Exception",                 // 1
    "NMI Interrupt",                   // 2
    "Breakpoint",                      // 3
    "Overflow",                        // 4
    "BOUND Range Exceeded",            // 5
    "Invalid Opcode",                  // 6
    "Device Not Available",            // 7
    "Double Fault",                    // 8
    "Coprocessor Segment Overrun",     // 9
    "Invalid TSS",                     // 10
    "Segment Not Present",             // 11
    "Stack Segment Fault",             // 12
    "General Protection Fault",        // 13
    "Page Fault",                      // 14
    "Reserved",                        // 15
    "x87 FPU Floating-Point Error",    // 16
    "Alignment Check",                 // 17
    "Machine Check",                   // 18
    "SIMD Floating-Point Exception"    // 19
};
```

### 下午：实现异常处理函数（3小时）

#### 步骤1：实现基本框架

```c
/*
 * exception_handler
 * 描述: C语言异常处理函数
 * 输入: regs - 寄存器上下文指针
 * 输出: 在屏幕上打印异常信息
 * 返回值: 无（不会返回）
 * 副作用: 系统停止运行
 */
void exception_handler(registers_t* regs) {
    /* 第一步：获取异常信息 */
    uint32_t exception_num = regs->int_no;
    uint32_t error_code = regs->err_code;

    /* 第二步：清屏并打印标题 */
    clear();
    printf("==================================================\n");
    printf("           EXCEPTION OCCURRED!\n");
    printf("==================================================\n\n");

    /* 第三步：打印异常类型 */
    if (exception_num < 20) {
        printf("Exception: %s (INT %d)\n",
               exception_names[exception_num], exception_num);
    } else {
        printf("Exception: Unknown (INT %d)\n", exception_num);
    }

    /* 第四步：打印错误码 */
    printf("Error Code: 0x%x\n", error_code);

    /* 第五步：特殊处理页错误 */
    // TODO: 实现页错误的特殊处理

    /* 第六步：打印寄存器状态 */
    // TODO: 实现寄存器打印

    /* 第七步：停止系统 */
    printf("\n==================================================\n");
    printf("System halted. Please reboot.\n");
    printf("==================================================\n");

    /* 无限循环 */
    while(1) {
        asm volatile ("hlt");  /* Halt指令 */
    }
}
```

#### 步骤2：实现页错误特殊处理

在"第五步"位置添加：

```c
    /* 特殊处理页错误（异常14）*/
    if (exception_num == 14) {
        uint32_t cr2;

        /* 读取CR2寄存器（保存引起页错误的地址）*/
        asm volatile ("movl %%cr2, %0" : "=r" (cr2));

        printf("Faulting Address (CR2): 0x%x\n", cr2);

        /* 解析错误码 */
        printf("Error Details:\n");

        /* Bit 0: P标志 */
        if (error_code & 0x1) {
            printf("  - Page protection violation\n");
        } else {
            printf("  - Page not present\n");
        }

        /* Bit 1: W/R标志 */
        if (error_code & 0x2) {
            printf("  - Write access\n");
        } else {
            printf("  - Read access\n");
        }

        /* Bit 2: U/S标志 */
        if (error_code & 0x4) {
            printf("  - User mode\n");
        } else {
            printf("  - Kernel mode\n");
        }

        /* Bit 3: RSVD标志 */
        if (error_code & 0x8) {
            printf("  - Reserved bit set\n");
        }

        /* Bit 4: I/D标志 */
        if (error_code & 0x10) {
            printf("  - Instruction fetch\n");
        }
    }
```

**理解检查**：

- CR2寄存器存储什么？
  答：引起页错误的虚拟地址

- 页错误的错误码各位含义？
  答：
    - Bit 0: 0=页不存在, 1=页保护违规
    - Bit 1: 0=读访问, 1=写访问
    - Bit 2: 0=内核态, 1=用户态
    - Bit 3: 保留位是否被设置
    - Bit 4: 是否是取指令

#### 步骤3：实现寄存器打印

在"第六步"位置添加：

```c
    /* 打印寄存器状态 */
    printf("\nRegister Dump:\n");
    printf("EAX: 0x%08x  EBX: 0x%08x  ECX: 0x%08x  EDX: 0x%08x\n",
           regs->eax, regs->ebx, regs->ecx, regs->edx);
    printf("ESI: 0x%08x  EDI: 0x%08x  EBP: 0x%08x  ESP: 0x%08x\n",
           regs->esi, regs->edi, regs->ebp, regs->esp);
    printf("EIP: 0x%08x  CS:  0x%04x  EFLAGS: 0x%08x\n",
           regs->eip, regs->cs, regs->eflags);
```

**问自己**：为什么打印这些寄存器？
答：帮助调试，了解异常发生时的CPU状态

---

## 🗓️ 第六-七天：实现分页系统

### 目标

实现完整的分页系统，包括页目录、页表的设置和启用。

### 第六天上午：定义分页数据结构（3小时）

#### 步骤1：理解分页的两级结构

在纸上画出：

```
虚拟地址（32位）：
┌───────────┬────────────┬────────────┐
│   DIR     │   TABLE    │   OFFSET   │
│ (10 bits) │ (10 bits)  │ (12 bits)  │
└───────────┴────────────┴────────────┘
     ↓            ↓            ↓
  PD索引      PT索引       页内偏移
  (0-1023)    (0-1023)     (0-4095)

查找过程：
1. CR3 → 页目录物理地址
2. DIR → PageDirectory[DIR] → 页表物理地址
3. TABLE → PageTable[TABLE] → 页框物理地址
4. 物理地址 = 页框地址 + OFFSET
```

#### 步骤2：定义PDE结构（4KB页）

编辑 `paging_my.h`：

```c
#ifndef _PAGING_MY_H
#define _PAGING_MY_H

#include "types.h"

/* 常量定义 */
#define NUM_PDE         1024    /* 页目录项数量 */
#define NUM_PTE         1024    /* 页表项数量 */
#define PAGE_SIZE       4096    /* 4KB */
#define PAGE_SIZE_4MB   0x400000 /* 4MB */
#define KERNEL_ADDR     0x400000 /* 内核起始地址（4MB）*/
#define VIDEO_MEM       0xB8000  /* 视频内存地址 */

/* 4KB页的PDE（指向页表）*/
typedef struct pde_4kb {
    uint32_t p          : 1;   /* Present（存在位）*/
    uint32_t rw         : 1;   /* Read/Write（读写位）*/
    uint32_t us         : 1;   /* User/Supervisor（用户/内核位）*/
    uint32_t pwt        : 1;   /* Page Write Through（写通）*/
    uint32_t pcd        : 1;   /* Page Cache Disable（禁用缓存）*/
    uint32_t a          : 1;   /* Accessed（访问位）*/
    uint32_t reserved   : 1;   /* Reserved（保留，必须为0）*/
    uint32_t ps         : 1;   /* Page Size（页大小，0=4KB）*/
    uint32_t g          : 1;   /* Global（全局位）*/
    uint32_t avail      : 3;   /* Available（OS可用）*/
    uint32_t page_table_base_addr : 20; /* 页表物理地址[31:12] */
} __attribute__((packed)) pde_4kb_t;
```

**理解每个位**：

```
P (Present):
  0 = 页表不存在，访问会触发#PF
  1 = 页表存在

R/W (Read/Write):
  0 = 只读
  1 = 可读写

U/S (User/Supervisor):
  0 = 只有内核（Ring 0）可访问
  1 = 用户（Ring 3）也可访问

PWT (Page Write Through):
  0 = 写回缓存（Write-Back）
  1 = 写通缓存（Write-Through）

PCD (Page Cache Disable):
  0 = 启用缓存
  1 = 禁用缓存

A (Accessed):
  CPU自动设置，表示页被访问过

PS (Page Size):
  0 = 4KB页（指向页表）
  1 = 4MB页（直接指向物理地址）

G (Global):
  0 = 切换CR3时刷新TLB
  1 = 不刷新（全局页）
```

#### 步骤3：定义PDE结构（4MB页）

继续在 `paging_my.h` 中：

```c
/* 4MB页的PDE（直接指向物理地址）*/
typedef struct pde_4mb {
    uint32_t p          : 1;   /* Present */
    uint32_t rw         : 1;   /* Read/Write */
    uint32_t us         : 1;   /* User/Supervisor */
    uint32_t pwt        : 1;   /* Page Write Through */
    uint32_t pcd        : 1;   /* Page Cache Disable */
    uint32_t a          : 1;   /* Accessed */
    uint32_t d          : 1;   /* Dirty（脏位，页被写过）*/
    uint32_t ps         : 1;   /* Page Size（必须为1）*/
    uint32_t g          : 1;   /* Global */
    uint32_t avail      : 3;   /* Available */
    uint32_t pat        : 1;   /* Page Attribute Table */
    uint32_t reserved   : 9;   /* Reserved（必须为0）*/
    uint32_t page_base_addr : 10; /* 4MB页物理地址[31:22] */
} __attribute__((packed)) pde_4mb_t;
```

**问自己**：为什么4MB页的地址只有10位？
答：因为4MB对齐，低22位总是0，只需存储高10位

#### 步骤4：定义PDE联合体和PTE

```c
/* PDE联合体（可以是4KB或4MB页）*/
typedef union pde {
    uint32_t val;      /* 作为32位整数访问 */
    pde_4kb_t kb;      /* 作为4KB页访问 */
    pde_4mb_t mb;      /* 作为4MB页访问 */
} pde_t;

/* PTE结构（总是4KB页）*/
typedef struct pte {
    uint32_t p          : 1;   /* Present */
    uint32_t rw         : 1;   /* Read/Write */
    uint32_t us         : 1;   /* User/Supervisor */
    uint32_t pwt        : 1;   /* Page Write Through */
    uint32_t pcd        : 1;   /* Page Cache Disable */
    uint32_t a          : 1;   /* Accessed */
    uint32_t d          : 1;   /* Dirty */
    uint32_t pat        : 1;   /* Page Attribute Table */
    uint32_t g          : 1;   /* Global */
    uint32_t avail      : 3;   /* Available */
    uint32_t page_base_addr : 20; /* 页框物理地址[31:12] */
} __attribute__((packed)) pte_t;
```

#### 步骤5：定义页目录和页表结构

```c
/* 页目录（1024个PDE，4KB对齐）*/
typedef struct page_directory {
    pde_t page_directory[NUM_PDE];
} __attribute__((aligned(PAGE_SIZE))) page_directory_t;

/* 页表（1024个PTE，4KB对齐）*/
typedef struct page_table {
    pte_t page_table[NUM_PTE];
} __attribute__((aligned(PAGE_SIZE))) page_table_t;

/* 全局变量声明 */
extern page_directory_t page_directory_array[1];
extern page_table_t page_table_array[1];
extern uint32_t page_dir_addr;

/* 函数声明 */
void init_paging();
void set_up_PD_PT();
void enable_paging();
void flush_TLB();

#endif /* _PAGING_MY_H */
```

**问自己**：

- 为什么要4KB对齐？
  答：CR3寄存器要求页目录地址低12位为0

- 为什么只定义一个页目录和一个页表？
  答：Checkpoint 1只需要映射0-4MB（页表）和4-8MB（内核），一个页表够用

### 第六天下午：实现set_up_PD_PT（4小时）

#### 步骤1：定义全局变量

编辑 `paging_my.c`：

```c
#include "paging_my.h"
#include "lib.h"

/* 全局变量：页目录和页表（4KB对齐）*/
page_directory_t page_directory_array[1] __attribute__((aligned(PAGE_SIZE)));
page_table_t page_table_array[1] __attribute__((aligned(PAGE_SIZE)));

/* 页目录物理地址 */
uint32_t page_dir_addr;
```

#### 步骤2：初始化所有页目录项为"不存在"

```c
/*
 * set_up_PD_PT
 * 描述: 设置页目录和页表
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 初始化页目录和页表
 */
void set_up_PD_PT() {
    int i;

    /* 保存页目录物理地址 */
    page_dir_addr = (uint32_t)page_directory_array;

    /* ===== 第一步：初始化所有页目录项为"不存在" ===== */
    for (i = 0; i < NUM_PDE; i++) {
        page_directory_array[0].page_directory[i].val = 0;
        page_directory_array[0].page_directory[i].kb.p = 0;  /* Not present */
    }

    printf("Initialized %d page directory entries\n", NUM_PDE);
```

**问自己**：为什么要先设置为"不存在"？
答：未映射的内存不应该被访问，访问会触发页错误

#### 步骤3：初始化所有页表项为"不存在"

```c
    /* ===== 第二步：初始化所有页表项为"不存在" ===== */
    for (i = 0; i < NUM_PTE; i++) {
        page_table_array[0].page_table[i].p = 0;  /* Not present */
        page_table_array[0].page_table[i].rw = 0;
        page_table_array[0].page_table[i].us = 0;
        page_table_array[0].page_table[i].page_base_addr = 0;
    }

    printf("Initialized %d page table entries\n", NUM_PTE);
```

#### 步骤4：设置页目录第0项（0-4MB，使用页表）

```c
    /* ===== 第三步：设置页目录第0项（0-4MB，使用页表）===== */
    page_directory_array[0].page_directory[0].kb.p = 1;      /* Present */
    page_directory_array[0].page_directory[0].kb.rw = 1;     /* Read/Write */
    page_directory_array[0].page_directory[0].kb.us = 0;     /* Supervisor */
    page_directory_array[0].page_directory[0].kb.pwt = 0;    /* Write-back */
    page_directory_array[0].page_directory[0].kb.pcd = 0;    /* Cache enabled */
    page_directory_array[0].page_directory[0].kb.a = 0;      /* Not accessed */
    page_directory_array[0].page_directory[0].kb.reserved = 0;
    page_directory_array[0].page_directory[0].kb.ps = 0;     /* 4KB page */
    page_directory_array[0].page_directory[0].kb.g = 0;      /* Not global */
    page_directory_array[0].page_directory[0].kb.avail = 0;

    /* 页表物理地址（高20位）*/
    page_directory_array[0].page_directory[0].kb.page_table_base_addr =
        ((uint32_t)page_table_array[0].page_table) >> 12;

    printf("Set up PD[0] pointing to page table at 0x%x\n",
           (uint32_t)page_table_array[0].page_table);
```

**理解**：

- 右移12位（>> 12）：因为地址是4KB对齐的，低12位总是0
- `page_table_array[0].page_table`：获取页表数组的地址

#### 步骤5：设置页表，映射视频内存（0xB8000）

```c
    /* ===== 第四步：设置页表，映射视频内存（0xB8000）===== */
    /* 0xB8000 / 4096 = 184，所以是页表的第184项 */
    int video_page_index = VIDEO_MEM / PAGE_SIZE;  /* 184 */

    page_table_array[0].page_table[video_page_index].p = 1;         /* Present */
    page_table_array[0].page_table[video_page_index].rw = 1;        /* Read/Write */
    page_table_array[0].page_table[video_page_index].us = 0;        /* Supervisor */
    page_table_array[0].page_table[video_page_index].pwt = 0;
    page_table_array[0].page_table[video_page_index].pcd = 0;
    page_table_array[0].page_table[video_page_index].a = 0;
    page_table_array[0].page_table[video_page_index].d = 0;
    page_table_array[0].page_table[video_page_index].pat = 0;
    page_table_array[0].page_table[video_page_index].g = 0;
    page_table_array[0].page_table[video_page_index].avail = 0;

    /* 视频内存物理地址（高20位）*/
    page_table_array[0].page_table[video_page_index].page_base_addr =
        VIDEO_MEM >> 12;  /* 0xB8000 >> 12 = 0xB8 */

    printf("Mapped video memory: virtual 0x%x -> physical 0x%x\n",
           VIDEO_MEM, VIDEO_MEM);
```

**计算验证**：

```
0xB8000 = 0000 0000 0000 1011 1000 0000 0000 0000
DIR    = 0000000000 (0)    ← 页目录第0项
TABLE  = 0010111000 (184)  ← 页表第184项（十进制）
OFFSET = 000000000000 (0)

0xB8 = 10111000 (二进制) = 184 (十进制) ✓
```

#### 步骤6：设置页目录第1项（4-8MB，内核，4MB大页）

```c
    /* ===== 第五步：设置页目录第1项（4-8MB，内核，4MB大页）===== */
    page_directory_array[0].page_directory[1].mb.p = 1;      /* Present */
    page_directory_array[0].page_directory[1].mb.rw = 1;     /* Read/Write */
    page_directory_array[0].page_directory[1].mb.us = 0;     /* Supervisor only */
    page_directory_array[0].page_directory[1].mb.pwt = 0;
    page_directory_array[0].page_directory[1].mb.pcd = 0;
    page_directory_array[0].page_directory[1].mb.a = 0;
    page_directory_array[0].page_directory[1].mb.d = 0;
    page_directory_array[0].page_directory[1].mb.ps = 1;     /* 4MB page */
    page_directory_array[0].page_directory[1].mb.g = 0;
    page_directory_array[0].page_directory[1].mb.avail = 0;
    page_directory_array[0].page_directory[1].mb.pat = 0;
    page_directory_array[0].page_directory[1].mb.reserved = 0; /* Must be 0 */

    /* 内核物理地址 = 4MB = 0x400000 */
    /* 4MB页的地址是bits[31:22]，即 0x400000 >> 22 = 1 */
    page_directory_array[0].page_directory[1].mb.page_base_addr =
        KERNEL_ADDR >> 22;  /* 0x400000 >> 22 = 1 */

    printf("Mapped kernel: virtual 0x%x -> physical 0x%x (4MB page)\n",
           KERNEL_ADDR, KERNEL_ADDR);

    printf("Page directory and page table setup complete!\n");
}
```

**计算验证**：

```
0x400000 = 0100 0000 0000 0000 0000 0000
右移22位: 0000 0000 0000 0000 0000 0001 = 1 ✓
```

### 第七天上午：实现enable_paging（2小时）

#### 步骤1：实现enable_paging函数

```c
/*
 * enable_paging
 * 描述: 启用分页
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 设置CR3和CR0，启用分页
 */
void enable_paging() {
    printf("Loading CR3 with page directory address: 0x%x\n", page_dir_addr);

    /* 第一步：加载页目录地址到CR3 */
    asm volatile (
        "movl %0, %%eax         \n"  /* 将page_dir_addr加载到EAX */
        "movl %%eax, %%cr3      \n"  /* 将EAX的值写入CR3 */
        : /* no output */
        : "r" (page_dir_addr)        /* 输入：page_dir_addr */
        : "eax"                      /* 破坏：EAX寄存器 */
    );

    printf("CR3 loaded successfully\n");
    printf("Enabling paging (setting CR0.PG bit)...\n");

    /* 第二步：启用CR0的PG位（bit 31）*/
    asm volatile (
        "movl %%cr0, %%eax      \n"  /* 读取CR0到EAX */
        "orl $0x80000000, %%eax \n"  /* 设置第31位（PG位）*/
        "movl %%eax, %%cr0      \n"  /* 写回CR0 */
        : /* no output */
        : /* no input */
        : "eax"                      /* 破坏：EAX寄存器 */
    );

    printf("Paging enabled!\n");
}
```

**理解内联汇编**：

```c
asm volatile (
    "汇编指令"
    : 输出操作数列表
    : 输入操作数列表
    : 破坏列表
);
```

- `volatile`：告诉编译器不要优化这段汇编
- `"r"`：约束符，表示使用任意寄存器
- `%%`：在内联汇编中，寄存器名前要用两个%

**问自己**：

- 为什么要用0x80000000？
  答：二进制是10000000...（第31位是1），用于设置PG位

- 为什么要先读CR0再写？
  答：保留其他位不变，只修改PG位

#### 步骤2：实现flush_TLB函数

```c
/*
 * flush_TLB
 * 描述: 刷新TLB（Translation Lookaside Buffer）
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 清空TLB缓存
 */
void flush_TLB() {
    /* 重新加载CR3会自动刷新TLB（除了Global页）*/
    asm volatile (
        "movl %%cr3, %%eax      \n"  /* 读取CR3到EAX */
        "movl %%eax, %%cr3      \n"  /* 写回CR3 */
        : /* no output */
        : /* no input */
        : "eax"
    );
}
```

**问自己**：什么时候需要刷新TLB？
答：

1. 修改页表后
2. 切换进程（改变页目录）后
3. 修改页权限后

#### 步骤3：实现init_paging主函数

```c
/*
 * init_paging
 * 描述: 初始化分页系统（主函数）
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 设置并启用分页
 */
void init_paging() {
    printf("\n");
    printf("==================================================\n");
    printf("  Initializing Paging System\n");
    printf("==================================================\n");

    /* 第一步：设置页目录和页表 */
    printf("\nStep 1: Setting up page directory and page table...\n");
    set_up_PD_PT();

    /* 第二步：启用分页 */
    printf("\nStep 2: Enabling paging...\n");
    enable_paging();

    printf("\n");
    printf("==================================================\n");
    printf("  Paging System Initialized Successfully!\n");
    printf("==================================================\n");
    printf("\n");
}
```

### 第七天下午：修改Makefile和kernel.c（2小时）

#### 步骤1：更新Makefile

编辑 `student-distrib/Makefile`，将你的新文件加入编译：

```makefile
# 添加你的源文件
SRCS += idt_init_my.c
SRCS += exception_handler_my.c
SRCS += interrupt_handler_my.S
SRCS += paging_my.c
```

或者直接替换原来的文件名。

#### 步骤2：修改kernel.c

编辑 `kernel.c`，在entry函数中调用你的初始化函数：

```c
#include "idt_init.h"      /* 你的IDT初始化 */
#include "paging_my.h"     /* 你的分页初始化 */

void entry(unsigned long magic, unsigned long addr) {
    /* ... Multiboot信息打印 ... */

    /* ... GDT, LDT, TSS初始化 ... */

    /* 初始化IDT */
    printf("\n=== Initializing IDT ===\n");
    initialize_IDT();
    printf("IDT initialized successfully!\n");

    /* 初始化分页 */
    printf("\n=== Initializing Paging ===\n");
    init_paging();

    /* 其他初始化... */

    /* 运行测试 */
    #ifdef RUN_TESTS
    launch_tests();
    #endif

    printf("\nKernel initialization complete!\n");

    /* 主循环 */
    while(1);
}
```

---

## 🗓️ 第八天：测试和调试

### 目标

编译、运行并通过所有Checkpoint 1的测试。

### 上午：编译并修复编译错误（2-3小时）

#### 步骤1：编译内核

```bash
cd student-distrib
make clean
make
```

#### 步骤2：常见编译错误和修复

**错误1**: `undefined reference to 'EXCEPTION_0'`

```
原因：汇编文件没有正确编译或链接
解决：检查Makefile，确保interrupt_handler_my.S被编译
```

**错误2**: `conflicting types for 'exception_handler'`

```
原因：头文件和实现的函数签名不一致
解决：确保.h和.c中的函数声明完全一致
```

**错误3**: `page_directory_array: undefined reference`

```
原因：全局变量没有定义，只有声明
解决：在.c文件中定义全局变量
```

**错误4**: 链接顺序错误

```
解决：在Makefile中调整.o文件的顺序
```

### 下午：运行测试（3小时）

#### 步骤1：启用测试

编辑 `kernel.c`：

```c
#define RUN_TESTS  /* 取消注释 */
```

编辑 `tests.c` 的 `launch_tests()` 函数：

```c
void launch_tests() {
    /* Checkpoint 1 tests */
    TEST_OUTPUT("idt_test", idt_test());
    TEST_OUTPUT("paging_valid_test", paging_valid_test());
    TEST_OUTPUT("paging_dereference_test", paging_dereference_test());

    /* 先不要运行异常测试（会触发异常停止系统）*/
    // TEST_OUTPUT("exception_test", exception_test());
}
```

#### 步骤2：运行QEMU

```bash
cd student-distrib
make
cd ..
qemu-system-i386 -hda mp3.img -m 64
```

**期望输出**：

```
Initializing IDT...
IDT initialized successfully!

Initializing Paging System...
Step 1: Setting up page directory and page table...
...
Step 2: Enabling paging...
...
Paging System Initialized Successfully!

[TEST idt_test] Result = PASS
[TEST paging_valid_test] Result = PASS
[TEST paging_dereference_test] Result = PASS
```

#### 步骤3：测试异常处理

修改 `tests.c`，启用异常测试：

```c
void launch_tests() {
    /* 其他测试... */

    /* 测试除零异常 */
    TEST_OUTPUT("exception_test", exception_test());
}
```

重新编译运行：

```bash
make
qemu-system-i386 -hda ../mp3.img -m 64
```

**期望输出**：

```
==================================================
           EXCEPTION OCCURRED!
==================================================

Exception: Divide Error (INT 0)
Error Code: 0x0

Register Dump:
EAX: 0x...  EBX: 0x...  ...

==================================================
System halted. Please reboot.
==================================================
```

### 晚上：使用GDB调试（2小时）

#### 步骤1：启动QEMU等待GDB

```bash
# Terminal 1
qemu-system-i386 -s -S -hda ../mp3.img -m 64
```

#### 步骤2：连接GDB

```bash
# Terminal 2
cd student-distrib
gdb
(gdb) target remote localhost:1234
(gdb) symbol-file kernel
```

#### 步骤3：设置断点并调试

```
# 在init_paging设置断点
(gdb) break init_paging
Breakpoint 1 at 0x...

# 继续执行到断点
(gdb) continue

# 单步执行
(gdb) step

# 查看变量
(gdb) print page_dir_addr
(gdb) print page_directory_array[0].page_directory[0]

# 查看内存
(gdb) x/10x page_directory_array

# 查看汇编
(gdb) disassemble enable_paging

# 查看寄存器
(gdb) info registers

# 查看CR3
(gdb) print $cr3
```

---

## 🗓️ 第九天：深入理解和文档

### 目标

确保真正理解每一行代码，能够给别人讲解。

### 上午：自我检查（3小时）

#### 回答以下问题（写在笔记中）：

**关于IDT**：

1. IDT表项的8个字节如何组成？画出结构图
2. 中断门和陷阱门的区别是什么？各用在哪里？
3. 为什么系统调用DPL=3，而异常DPL=0？
4. CPU如何找到IDT？（IDTR寄存器）
5. SET_IDT_ENTRY宏做了什么？

**关于异常处理**：

6. 为什么需要汇编包装函数？
7. 异常发生时，栈的状态如何变化？画出图
8. PUSHAL指令压入哪些寄存器？顺序如何？
9. 为什么有些异常有错误码，有些没有？
10. 页错误时如何获取引起错误的地址？

**关于分页**：

11. 虚拟地址如何转换为物理地址？画出转换过程
12. 为什么内核用4MB大页，视频内存用4KB页？
13. PDE和PTE的Present位为0时会发生什么？
14. 什么是TLB？为什么需要刷新？
15. CR0、CR2、CR3寄存器分别存储什么？

**关于实现细节**：

16. 为什么页目录和页表要4KB对齐？
17. 地址右移12位和右移22位分别代表什么？
18. 为什么要将所有未使用的页设为"不存在"？
19. 如何计算视频内存的页表索引？
20. 内联汇编的语法是什么？

### 下午：编写学习笔记（3小时）

在 `docs/my-checkpoint1-notes.md` 中写下：

#### 1. 实现总结

```markdown
# Checkpoint 1 实现总结

## 实现的功能

- [ ] IDT初始化（256项）
- [ ] 20个异常处理程序
- [ ] 汇编异常包装函数
- [ ] 分页系统（0-4MB和4-8MB）
- [ ] 所有测试通过

## 遇到的问题和解决

1. 问题：编译时找不到EXCEPTION_0
   解决：...

2. 问题：页错误发生在...
   解决：...

## 学到的知识点

1. x86保护模式的工作原理
2. 中断和异常的区别
3. 虚拟内存和物理内存的映射
4. ...
```

#### 2. 代码注释

给关键代码添加详细注释，解释为什么这样写。

#### 3. 画图总结

画出：

- IDT结构图
- 异常处理流程图
- 分页转换流程图
- 栈状态变化图

### 晚上：优化和扩展（2小时）

#### 可选优化：

1. **添加更多调试信息**

```c
#ifdef DEBUG
    printf("DEBUG: Setting PDE[%d]\n", i);
#endif
```

2. **改进异常信息显示**
    - 不同异常用不同颜色
    - 显示代码反汇编
    - 显示栈回溯

3. **添加页错误详细分析**
    - 分析是哪个函数引起的
    - 显示附近的内存内容

4. **性能测试**
    - 测试TLB命中率
    - 测试页表查找时间

---

## ✅ 最终验收清单

### 功能验收

- [ ] 系统成功启动
- [ ] IDT正确初始化（256项）
- [ ] 能够捕获和显示除零异常
- [ ] 分页启用成功
- [ ] 能访问视频内存（0xB8000）
- [ ] 能访问内核内存（4-8MB）
- [ ] 访问未映射内存触发页错误
- [ ] idt_test通过
- [ ] paging_valid_test通过
- [ ] paging_dereference_test通过

### 理解验收

能够清楚回答：

- [ ] GDT、IDT、TSS的作用
- [ ] 中断门和陷阱门的区别
- [ ] 虚拟地址如何转换为物理地址
- [ ] 为什么需要两级页表
- [ ] TLB的作用和刷新时机
- [ ] 页错误的常见原因
- [ ] CR0、CR2、CR3的作用

### 代码质量

- [ ] 代码有详细注释
- [ ] 函数有完整的文档说明
- [ ] 变量命名清晰
- [ ] 没有硬编码的魔数
- [ ] 使用宏定义常量

---

## 🎓 完成后的自我评估

### Level 1: 基础（完成实现）

- 代码能编译运行
- 所有测试通过
- 能说出每个函数的作用

### Level 2: 理解（深入理解）

- 理解每行代码为什么这样写
- 能画出数据结构和流程图
- 能解释关键概念

### Level 3: 精通（可以讲解）

- 能给别人讲清楚整个实现
- 能回答任何细节问题
- 能独立设计类似系统

**目标：达到Level 3！**

---

## 📚 额外学习资源

### 深入阅读

1. **Intel手册** Volume 3A
    - Chapter 3: Protected-Mode Memory Management
    - Chapter 4: Paging
    - Chapter 6: Interrupt and Exception Handling

2. **OSDev Wiki**
    - https://wiki.osdev.org/Paging
    - https://wiki.osdev.org/Interrupt_Descriptor_Table
    - https://wiki.osdev.org/Exceptions

3. **xv6源码分析**
    - vm.c: 虚拟内存管理
    - trap.c: 中断和异常处理

### 推荐视频

- MIT 6.828 Lecture 2-3: 内存管理和分页
- UC Berkeley CS162: Virtual Memory

---

## 🚀 下一步：Checkpoint 2

完成Checkpoint 1后，你将实现：

- 8259 PIC（中断控制器）
- 键盘驱动
- RTC驱动
- 终端驱动

**准备工作**：

- 阅读OSTEP第36章（I/O设备）
- 浏览OSDev Wiki的8259 PIC、Keyboard、RTC页面
- 理解端口I/O（inb/outb指令）

---

**恭喜！完成从零实现Checkpoint 1！** 🎉

你现在不仅会写代码，更重要的是**真正理解了操作系统的底层工作原理**！

记住：
> "我听到的会忘记，我看到的能记住，我做过的才真正理解！"

现在你已经**做过了**，所以你**真正理解了**！ 💪
