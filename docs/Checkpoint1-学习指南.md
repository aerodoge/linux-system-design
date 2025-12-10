# Checkpoint 1 深度学习指南

## 🎯 学习目标

通过Checkpoint 1，你将深入理解：

1. **x86保护模式**：CPU如何从实模式切换到保护模式
2. **内存分段机制**：GDT、LDT、段选择子的作用
3. **中断机制**：IDT如何工作，中断门和陷阱门的区别
4. **异常处理**：如何捕获和处理20种CPU异常
5. **虚拟内存**：分页的原理和实现
6. **地址转换**：虚拟地址如何通过页表转换为物理地址

---

## 📚 第一步：理解整体架构

### 操作系统启动流程

```
┌─────────────┐
│  BIOS启动   │ ← 计算机开机
└──────┬──────┘
       │ 加载引导扇区
       ↓
┌─────────────┐
│  GRUB加载   │ ← 读取内核镜像到内存
└──────┬──────┘
       │ 跳转到内核入口
       ↓
┌─────────────┐
│   boot.S    │ ← 设置栈，跳转到C代码
└──────┬──────┘
       │ 调用entry()
       ↓
┌─────────────┐
│  kernel.c   │ ← 内核初始化
│  entry()    │   1. 初始化GDT
└──────┬──────┘   2. 初始化IDT ← Checkpoint 1
       │          3. 初始化分页 ← Checkpoint 1
       │          4. 启用中断
       ↓
┌─────────────┐
│ 内核主循环  │ ← 等待中断和执行任务
└─────────────┘
```

### Checkpoint 1 在整体中的位置

```
你的任务是实现以下三大模块：

1. IDT（中断描述符表）
   ├─ 初始化256个中断向量
   ├─ 设置20个异常处理程序
   ├─ 预留中断和系统调用入口
   └─ 加载IDTR寄存器

2. 异常处理
   ├─ 编写汇编包装函数
   ├─ 实现C语言处理函数
   └─ 在屏幕上打印异常信息

3. 分页系统
   ├─ 创建页目录（Page Directory）
   ├─ 创建页表（Page Table）
   ├─ 设置内存映射
   ├─ 启用CR0的PG位
   └─ 加载CR3寄存器
```

---

## 🧠 概念深入理解

### 1️⃣ 为什么需要保护模式？

**实模式的问题**：

- 只能访问1MB内存（20位地址线）
- 没有内存保护，任何程序都能访问任何内存
- 没有虚拟内存
- 一个程序崩溃会导致整个系统崩溃

**保护模式的优势**：

- 可以访问4GB内存（32位地址线）
- 内存分段和分页保护
- 特权级（Ring 0-3）隔离
- 虚拟内存支持

---

### 2️⃣ 内存分段（Segmentation）

#### 全局描述符表（Global Descriptor Table, GDT）是什么？

```c
// GDT表项结构（8字节）
struct gdt_entry {
    uint16_t limit_low;      // 段界限（低16位）
    uint16_t base_low;       // 段基址（低16位）
    uint8_t  base_middle;    // 段基址（中8位）
    uint8_t  access;         // 访问权限
    uint8_t  granularity;    // 粒度和高4位界限
    uint8_t  base_high;      // 段基址（高8位）
};
```

**GDT的作用**：

- 定义内存段的起始地址、大小、权限
- 将逻辑地址转换为线性地址
- 实现特权级保护

**你的项目的GDT布局**（已提供）：

```
索引  段选择子      类型          基址    大小      特权级
0     0x0000      NULL段        -       -         -
1     0x0000      NULL段        -       -         -
2     0x0010      内核代码段     0       4GB      Ring 0
3     0x0018      内核数据段     0       4GB      Ring 0
4     0x0023      用户代码段     0       4GB      Ring 3
5     0x002B      用户数据段     0       4GB      Ring 3
6     0x0030      TSS           -       -         -
7     0x0038      LDT           -       -         -
```

**注意**：段选择子的计算公式是 `Index × 8 + TI + RPL`

- 例如：KERNEL_CS = 索引2 × 8 + 0 + 0 = 0x0010
- 例如：USER_CS = 索引4 × 8 + 0 + 3 = 0x0020 + 3 = 0x0023

**段选择子（Segment Selector）**：

```
15                    3  2 1 0
┌─────────────────────┬──┬────┐
│  Index (13 bits)    │TI│RPL │
└─────────────────────┴──┴────┘
  ↑                    ↑   ↑
  GDT索引            表标志 请求特权级
                     0=GDT
                     1=LDT
```

**重要的段选择子常量**：

```c
#define KERNEL_CS   0x0010   // 内核代码段
#define KERNEL_DS   0x0018   // 内核数据段
#define USER_CS     0x0023   // 用户代码段 (RPL=3)
#define USER_DS     0x002B   // 用户数据段 (RPL=3)
```

#### 为什么要平坦内存模型？

现代OS（包括你的项目）使用**平坦内存模型**：

- 所有段的基址 = 0
- 所有段的界限 = 4GB
- 相当于"关闭"了分段机制
- 主要依赖分页来管理内存

**分段地址转换**：

```
逻辑地址 = 段选择子 : 偏移量
              ↓
         查GDT获取段基址
              ↓
线性地址 = 段基址 + 偏移量
              ↓
        （如果启用分页）
              ↓
         通过页表转换
              ↓
          物理地址
```

在平坦模型中：段基址=0，所以 **线性地址 = 偏移量**

---

### 3️⃣ 中断描述符表（Interrupt Descriptor Table, IDT）

#### IDT是什么？

IDT是一个有256个表项的数组，每个表项指向一个中断/异常处理程序。

```c
// IDT表项结构（8字节）
struct idt_entry {
    uint16_t offset_low;     // 处理程序地址（低16位）
    uint16_t seg_selector;   // 段选择子（通常是KERNEL_CS）
    uint8_t  reserved;       // 保留（必须为0）
    uint8_t  type_attr;      // 类型和属性
    uint16_t offset_high;    // 处理程序地址（高16位）
};
```

**IDT布局**：

```
索引      用途                     你需要实现
0-19     CPU异常                   ✅ 实现异常处理
20-31    保留（Intel预留）         设为通用处理
32       PIT定时器中断             ⏰ Checkpoint 5
33       键盘中断                  ⏰ Checkpoint 2
...
40       RTC中断                   ⏰ Checkpoint 2
...
128(0x80) 系统调用                 ⏰ Checkpoint 4
```

#### 中断门 vs 陷阱门

**type_attr字段**：

```
7     6 5     4   3 2 1 0
┌──┬─────┬────┬───────────┐
│P │ DPL │ 0  │  Type     │
└──┴─────┴────┴───────────┘
 ↑    ↑         ↑
 Present  特权级   门类型
 1=有效   0-3     1110=中断门
                  1111=陷阱门
```

**中断门（Interrupt Gate）**：

- 进入时**自动关闭中断**（CLI）
- 用于硬件中断（键盘、定时器等）
- Type = 0x8E（P=1, DPL=0, Type=0xE）

**陷阱门（Trap Gate）**：

- 进入时**不关闭中断**
- 用于异常和系统调用
- Type = 0x8F（P=1, DPL=0, Type=0xF）

**系统调用门**：

- 特殊的陷阱门
- DPL=3（允许用户态调用）
- Type = 0xEF（P=1, DPL=3, Type=0xF）

#### CPU如何使用IDT？

```
1. 中断/异常发生
   ↓
2. CPU读取IDTR寄存器获取IDT地址
   ↓
3. 根据中断向量号n，找到IDT[n]
   ↓
4. 从IDT[n]中读取处理程序地址
   ↓
5. 保存当前状态（SS, ESP, EFLAGS, CS, EIP）
   ↓
6. 切换到内核栈（通过TSS）
   ↓
7. 跳转到处理程序执行
   ↓
8. 处理程序执行完毕，调用IRET
   ↓
9. 恢复之前的状态，继续执行
```

---

### 4️⃣ x86异常

#### 20种异常详解

| 向量 | 名称  | 类型         | 错误码  | 含义        | 常见原因           |
|----|-----|------------|------|-----------|----------------|
| 0  | #DE | Fault      | 无    | 除零错误      | 除数为0           |
| 1  | #DB | Fault/Trap | 无    | 调试异常      | 单步执行           |
| 2  | NMI | Interrupt  | 无    | 不可屏蔽中断    | 硬件错误           |
| 3  | #BP | Trap       | 无    | 断点        | INT 3指令        |
| 4  | #OF | Trap       | 无    | 溢出        | INTO指令         |
| 5  | #BR | Fault      | 无    | 越界        | BOUND指令        |
| 6  | #UD | Fault      | 无    | 无效操作码     | 执行非法指令         |
| 7  | #NM | Fault      | 无    | 设备不可用     | FPU指令但CR0.EM=1 |
| 8  | #DF | Abort      | 有(0) | 双重错误      | 异常处理时又发生异常     |
| 10 | #TS | Fault      | 有    | 无效TSS     | 任务切换失败         |
| 11 | #NP | Fault      | 有    | 段不存在      | 访问P=0的段        |
| 12 | #SS | Fault      | 有    | 栈段错误      | 栈操作越界          |
| 13 | #GP | Fault      | 有    | 通用保护错误    | 违反保护机制         |
| 14 | #PF | Fault      | 有    | 页错误       | 访问未映射/无权限页     |
| 16 | #MF | Fault      | 无    | x87 FPU错误 | 浮点运算异常         |
| 17 | #AC | Fault      | 有(0) | 对齐检查      | 未对齐的内存访问       |
| 18 | #MC | Abort      | 无    | 机器检查      | CPU硬件错误        |
| 19 | #XM | Fault      | 无    | SIMD浮点异常  | SSE指令错误        |

**最常见的异常**：

- **#PF (14)**: 页错误 - 你会经常遇到！
- **#GP (13)**: 通用保护错误 - 权限问题
- **#DE (0)**: 除零错误 - 用于测试

#### 错误码（Error Code）

某些异常会压入错误码：

```
┌─────┬─────┬─────┬─────────────┐
│ EXT │ IDT │ TI  │    Index    │
└─────┴─────┴─────┴─────────────┘
  31    3     2    1            0

EXT: 0=内部事件, 1=外部事件
IDT: 1=中断门引起
TI:  0=GDT, 1=LDT
Index: 段选择子索引
```

**页错误(#PF)的错误码**：

```
位    含义
0     P   = 1: 页保护违规, 0: 页不存在
1     W/R = 1: 写访问, 0: 读访问
2     U/S = 1: 用户态, 0: 内核态
3     RSVD= 1: 保留位被设置
4     I/D = 1: 取指令, 0: 数据访问
```

**CR2寄存器**：页错误发生时，CR2保存引起错误的虚拟地址！

---

### 5️⃣ 分页系统

#### 为什么需要分页？

**不用分页的问题**：

- 内存碎片：大块连续内存难以分配
- 无法隔离进程：进程A可以访问进程B的内存
- 无法实现虚拟内存：物理内存不够时无法使用硬盘

**分页的优势**：

- 每个进程独立的虚拟地址空间（0-4GB）
- 物理内存可以不连续
- 按需加载（页不在物理内存时从磁盘加载）
- 写时复制（fork时共享页）
- 内存保护（读/写/执行权限）

#### 两级页表结构

```
32位虚拟地址的分解：

31        22 21        12 11         0
┌───────────┬────────────┬────────────┐
│   DIR     │   TABLE    │   OFFSET   │
│ (10 bits) │ (10 bits)  │ (12 bits)  │
└───────────┴────────────┴────────────┘
     ↓            ↓            ↓
  页目录索引   页表索引    页内偏移
```

**地址转换过程**：

```
1. 从CR3寄存器获取页目录物理地址

2. 使用DIR（高10位）作为索引：
   PDE = PageDirectory[DIR]

3. 从PDE中获取页表物理地址

4. 使用TABLE（中10位）作为索引：
   PTE = PageTable[TABLE]

5. 从PTE中获取页框物理地址

6. 物理地址 = 页框地址 + OFFSET（低12位）
```

**示例**：转换虚拟地址 0xB8000

```
0xB8000 = 0000 0000 0000 1011 1000 0000 0000 0000

DIR    = 0000000000 (0)    ← 页目录第0项
TABLE  = 0010111000 (184)  ← 页表第184项
OFFSET = 000000000000 (0)  ← 页内偏移0

1. PageDirectory[0] → 页表A的物理地址
2. PageTable_A[184] → 页框的物理地址 = 0xB8000
3. 物理地址 = 0xB8000 + 0 = 0xB8000（恰好是视频内存）
```

#### 页目录项（PDE）和页表项（PTE）

**4KB页的PDE（指向页表）**：

```
31              12 11  9 8 7 6 5 4 3 2 1 0
┌─────────────────┬────┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│ Page Table Addr │Avail│G│0│0│A│C│W│U│R│P│
└─────────────────┴────┴─┴─┴─┴─┴─┴─┴─┴─┴─┘

P: Present（存在位）         = 1表示页表存在
R/W: Read/Write             = 1可写，0只读
U/S: User/Supervisor        = 1用户可访问，0仅内核
PWT: Page Write Through     = 缓存策略
PCD: Page Cache Disable     = 1禁用缓存
A: Accessed                 = CPU自动设置
PS: Page Size               = 0表示4KB页
G: Global                   = 1全局页（不刷新TLB）
Avail: 操作系统可用位
Page Table Addr: 页表物理地址（4KB对齐，低12位=0）
```

**4MB页的PDE（直接指向物理地址）**：

```
31        22 21      13 12 11  9 8 7 6 5 4 3 2 1 0
┌───────────┬─────────┬──┬────┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│Page Addr  │Reserved │PA│Avail│G│1│D│A│C│W│U│R│P│
└───────────┴─────────┴──┴────┴─┴─┴─┴─┴─┴─┴─┴─┴─┘

PS: Page Size = 1表示4MB大页
D: Dirty = 页被写过
Page Addr: 4MB对齐的物理地址（低22位=0）
```

**PTE结构（类似PDE，但总是4KB）**：

```
31              12 11  9 8 7 6 5 4 3 2 1 0
┌─────────────────┬────┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│  Page Frame Addr│Avail│G│0│D│A│C│W│U│R│P│
└─────────────────┴────┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
```

#### CR寄存器

**CR0**: 控制寄存器

```
位31: PG = 1启用分页
位16: WP = 1内核不能写只读页
位0:  PE = 1启用保护模式
```

**CR3**: 页目录基址寄存器（PDBR）

```
31              12 11         0
┌─────────────────┬────────────┐
│Page Dir Address │   Flags    │
└─────────────────┴────────────┘

写入CR3会刷新TLB（除了Global页）
```

**CR2**: 页错误线性地址

```
发生#PF时，CPU自动将引起错误的虚拟地址写入CR2
```

#### TLB（Translation Lookaside Buffer）

**什么是TLB**：

- 虚拟地址到物理地址的缓存
- 硬件实现，非常快
- 避免每次访问内存都查页表

**为什么需要刷新TLB**：

- 修改页表后，TLB中的旧映射失效
- 进程切换时，需要使用新进程的页表

**如何刷新TLB**：

```asm
; 方法1: 重新加载CR3
mov eax, cr3
mov cr3, eax

; 方法2: 使用INVLPG指令（刷新单个页）
invlpg [address]
```

---

## 🔨 实现步骤

### 步骤0: 阅读现有代码

先熟悉已提供的代码框架：

```bash
student-distrib/
├── boot.S              # 启动代码（已提供）
├── kernel.c            # 内核入口（已提供）
├── x86_desc.h          # x86描述符定义（已提供）
├── x86_desc.S          # GDT/IDT/TSS数据（已提供）
├── idt_init.c          # ❌ 你要实现：IDT初始化
├── idt_init.h          # ❌ 你要实现：头文件
├── exception_handler.c # ❌ 你要实现：异常处理
├── exception_handler.h # ❌ 你要实现：头文件
├── interrupt_handler.S # ❌ 你要实现：中断包装函数
├── paging.c            # ❌ 你要实现：分页
├── paging.h            # ❌ 你要实现：头文件
├── lib.c               # 屏幕打印函数（已提供）
└── tests.c             # 测试代码（已提供）
```

---

### 步骤1: 实现IDT初始化

#### 1.1 理解数据结构

查看 `x86_desc.h` 中的IDT定义：

```c
// IDT表项结构
typedef union idt_desc_t {
    uint32_t val[2];
    struct {
        uint16_t offset_15_00;    // 处理程序地址低16位
        uint16_t seg_selector;    // 段选择子
        uint8_t  reserved4;       // 保留
        uint8_t  reserved3 : 1;   // 0
        uint8_t  reserved2 : 1;   // 1
        uint8_t  reserved1 : 1;   // 1
        uint8_t  size      : 1;   // 1=32位
        uint8_t  reserved0 : 1;   // 0
        uint8_t  dpl       : 2;   // 特权级
        uint8_t  present   : 1;   // 存在位
        uint16_t offset_31_16;    // 处理程序地址高16位
    } __attribute__((packed));
} idt_desc_t;

// IDT表（256项）
idt_desc_t idt[NUM_VEC];  // NUM_VEC = 256
```

#### 1.2 设置IDT表项的宏

在 `x86_desc.h` 中已定义：

```c
#define SET_IDT_ENTRY(str, handler)          \
do {                                          \
    str.offset_15_00 = ADDR_LOW(handler);    \
    str.offset_31_16 = ADDR_HIGH(handler);   \
    str.seg_selector = KERNEL_CS;            \
} while(0)

#define ADDR_LOW(addr)  ((uint32_t)(addr) & 0xFFFF)
#define ADDR_HIGH(addr) (((uint32_t)(addr) >> 16) & 0xFFFF)
```

#### 1.3 编写初始化函数

在 `idt_init.c` 中：

```c
#include "idt_init.h"
#include "x86_desc.h"
#include "lib.h"

/* 外部声明异常处理函数（在interrupt_handler.S中定义） */
extern void EXCEPTION_0();   // 除零
extern void EXCEPTION_1();   // 调试
extern void EXCEPTION_2();   // NMI
extern void EXCEPTION_3();   // 断点
extern void EXCEPTION_4();   // 溢出
extern void EXCEPTION_5();   // 越界
extern void EXCEPTION_6();   // 无效操作码
extern void EXCEPTION_7();   // 设备不可用
extern void EXCEPTION_8();   // 双重错误
extern void EXCEPTION_9();   // 协处理器段溢出
extern void EXCEPTION_10();  // 无效TSS
extern void EXCEPTION_11();  // 段不存在
extern void EXCEPTION_12();  // 栈段错误
extern void EXCEPTION_13();  // 通用保护错误
extern void EXCEPTION_14();  // 页错误
extern void EXCEPTION_16();  // x87 FPU错误
extern void EXCEPTION_17();  // 对齐检查
extern void EXCEPTION_18();  // 机器检查
extern void EXCEPTION_19();  // SIMD浮点异常

/* 外部声明中断处理函数（Checkpoint 2实现） */
extern void KB_handler();    // 键盘中断
extern void RTC_handler();   // RTC中断
extern void PIT_handler();   // PIT中断

/* 外部声明系统调用（Checkpoint 4实现） */
extern void syscall();       // 系统调用

/*
 * initialize_IDT
 * 描述: 初始化IDT表
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 初始化所有256个IDT表项
 */
void initialize_IDT() {
    int i;

    /* 第一步: 初始化所有IDT表项为默认值 */
    for (i = 0; i < NUM_VEC; i++) {
        idt[i].seg_selector = KERNEL_CS;   // 内核代码段
        idt[i].reserved4 = 0;
        idt[i].reserved3 = 0;              // 中断门
        idt[i].reserved2 = 1;
        idt[i].reserved1 = 1;
        idt[i].size = 1;                   // 32位
        idt[i].reserved0 = 0;
        idt[i].dpl = 0;                    // Ring 0
        idt[i].present = 1;                // 存在

        /* 异常使用陷阱门（不自动关中断） */
        if (i < 32) {
            idt[i].reserved3 = 1;          // 陷阱门
        }

        /* 系统调用使用陷阱门，DPL=3（用户态可调用） */
        if (i == 0x80) {
            idt[i].reserved3 = 1;          // 陷阱门
            idt[i].dpl = 3;                // Ring 3
        }
    }

    /* 第二步: 设置异常处理程序 */
    set_exceptions();

    /* 第三步: 设置中断处理程序（Checkpoint 2会用到） */
    set_interrupts();

    /* 第四步: 设置系统调用（Checkpoint 4会用到） */
    SET_IDT_ENTRY(idt[0x80], syscall);

    /* 第五步: 加载IDT */
    lidt(idt_desc_ptr);
}

/*
 * set_exceptions
 * 描述: 在IDT中设置所有异常处理程序
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 填充IDT的0-19项
 */
void set_exceptions() {
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
    // 15是保留的，不设置
    SET_IDT_ENTRY(idt[16], EXCEPTION_16);
    SET_IDT_ENTRY(idt[17], EXCEPTION_17);
    SET_IDT_ENTRY(idt[18], EXCEPTION_18);
    SET_IDT_ENTRY(idt[19], EXCEPTION_19);

    /* 20-31设为通用异常处理（保留） */
    for (int i = 20; i < 32; i++) {
        if (i != 15) {
            SET_IDT_ENTRY(idt[i], EXCEPTION_1);  // 复用EXCEPTION_1
        }
    }
}

/*
 * set_interrupts
 * 描述: 在IDT中设置硬件中断处理程序
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 填充IDT的中断部分
 */
void set_interrupts() {
    SET_IDT_ENTRY(idt[32], PIT_handler);   // IRQ 0: PIT
    SET_IDT_ENTRY(idt[33], KB_handler);    // IRQ 1: 键盘
    SET_IDT_ENTRY(idt[40], RTC_handler);   // IRQ 8: RTC
}
```

---

### 步骤2: 实现异常处理程序

#### 2.1 汇编包装函数

在 `interrupt_handler.S` 中：

```asm
# interrupt_handler.S
# 异常和中断的汇编包装函数

.text
.globl EXCEPTION_0, EXCEPTION_1, EXCEPTION_2, EXCEPTION_3
.globl EXCEPTION_4, EXCEPTION_5, EXCEPTION_6, EXCEPTION_7
.globl EXCEPTION_8, EXCEPTION_9, EXCEPTION_10, EXCEPTION_11
.globl EXCEPTION_12, EXCEPTION_13, EXCEPTION_14, EXCEPTION_16
.globl EXCEPTION_17, EXCEPTION_18, EXCEPTION_19

.globl KB_handler, RTC_handler, PIT_handler, syscall

# 宏: 没有错误码的异常
# 参数: 异常号
.macro EXCEPTION_NO_ERROR_CODE num
EXCEPTION_\num:
    pushl $0                    # 压入假错误码（保持栈对齐）
    pushl $\num                 # 压入异常号
    jmp exception_common        # 跳转到公共处理
.endm

# 宏: 有错误码的异常
# 参数: 异常号
.macro EXCEPTION_WITH_ERROR_CODE num
EXCEPTION_\num:
    # CPU已经压入错误码
    pushl $\num                 # 压入异常号
    jmp exception_common        # 跳转到公共处理
.endm

# 定义所有异常处理入口
EXCEPTION_NO_ERROR_CODE 0       # 除零
EXCEPTION_NO_ERROR_CODE 1       # 调试
EXCEPTION_NO_ERROR_CODE 2       # NMI
EXCEPTION_NO_ERROR_CODE 3       # 断点
EXCEPTION_NO_ERROR_CODE 4       # 溢出
EXCEPTION_NO_ERROR_CODE 5       # 越界
EXCEPTION_NO_ERROR_CODE 6       # 无效操作码
EXCEPTION_NO_ERROR_CODE 7       # 设备不可用
EXCEPTION_WITH_ERROR_CODE 8     # 双重错误（有错误码）
EXCEPTION_NO_ERROR_CODE 9       # 协处理器段溢出
EXCEPTION_WITH_ERROR_CODE 10    # 无效TSS（有错误码）
EXCEPTION_WITH_ERROR_CODE 11    # 段不存在（有错误码）
EXCEPTION_WITH_ERROR_CODE 12    # 栈段错误（有错误码）
EXCEPTION_WITH_ERROR_CODE 13    # 通用保护错误（有错误码）
EXCEPTION_WITH_ERROR_CODE 14    # 页错误（有错误码）
EXCEPTION_NO_ERROR_CODE 16      # x87 FPU错误
EXCEPTION_WITH_ERROR_CODE 17    # 对齐检查（有错误码）
EXCEPTION_NO_ERROR_CODE 18      # 机器检查
EXCEPTION_NO_ERROR_CODE 19      # SIMD浮点异常

# 公共异常处理代码
exception_common:
    # 此时栈的状态:
    # [ESP+0]:  异常号
    # [ESP+4]:  错误码（或假错误码0）
    # [ESP+8]:  EIP（CPU压入）
    # [ESP+12]: CS（CPU压入）
    # [ESP+16]: EFLAGS（CPU压入）
    # [ESP+20]: ESP（如果特权级改变，CPU压入）
    # [ESP+24]: SS（如果特权级改变，CPU压入）

    # 保存所有寄存器
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    pushal                      # 压入EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI

    # 设置内核数据段
    movw $KERNEL_DS, %ax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %fs
    movw %ax, %gs

    # 调用C语言异常处理函数
    # 参数1: 异常号（在栈上）
    # 参数2: 错误码（在栈上）
    pushl %esp                  # 传递寄存器上下文指针
    call exception_handler      # 调用C函数
    addl $4, %esp               # 清理参数

    # 恢复所有寄存器
    popal
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # 跳过异常号和错误码
    addl $8, %esp

    # 返回
    iret

# 临时的中断处理程序（Checkpoint 2会实现）
KB_handler:
    iret

RTC_handler:
    iret

PIT_handler:
    iret

syscall:
    iret
```

#### 2.2 C语言处理函数

在 `exception_handler.c` 中：

```c
#include "exception_handler.h"
#include "lib.h"
#include "x86_desc.h"

/* 异常名称数组 */
static const char* exception_names[20] = {
    "Divide Error",
    "Debug Exception",
    "NMI Interrupt",
    "Breakpoint",
    "Overflow",
    "BOUND Range Exceeded",
    "Invalid Opcode",
    "Device Not Available",
    "Double Fault",
    "Coprocessor Segment Overrun",
    "Invalid TSS",
    "Segment Not Present",
    "Stack Segment Fault",
    "General Protection Fault",
    "Page Fault",
    "Reserved",
    "x87 FPU Floating-Point Error",
    "Alignment Check",
    "Machine Check",
    "SIMD Floating-Point Exception"
};

/*
 * exception_handler
 * 描述: C语言异常处理函数
 * 输入: regs - 寄存器上下文指针
 * 输出: 在屏幕上打印异常信息
 * 返回值: 无
 * 副作用: 系统停止（无限循环）
 */
void exception_handler(registers_t* regs) {
    uint32_t exception_num = regs->int_no;
    uint32_t error_code = regs->err_code;

    /* 清屏并打印异常信息 */
    clear();
    printf("==================================================\n");
    printf("           EXCEPTION OCCURRED!\n");
    printf("==================================================\n\n");

    /* 打印异常类型 */
    if (exception_num < 20) {
        printf("Exception: %s (INT %d)\n", exception_names[exception_num], exception_num);
    } else {
        printf("Exception: Unknown (INT %d)\n", exception_num);
    }

    /* 打印错误码 */
    printf("Error Code: 0x%x\n", error_code);

    /* 特殊处理页错误 */
    if (exception_num == 14) {  // Page Fault
        uint32_t cr2;
        asm volatile ("movl %%cr2, %0" : "=r" (cr2));
        printf("Faulting Address (CR2): 0x%x\n", cr2);

        /* 解析错误码 */
        printf("Error Details:\n");
        printf("  - %s\n", (error_code & 0x1) ? "Page protection violation" : "Page not present");
        printf("  - %s access\n", (error_code & 0x2) ? "Write" : "Read");
        printf("  - %s mode\n", (error_code & 0x4) ? "User" : "Kernel");
        if (error_code & 0x8)
            printf("  - Reserved bit set\n");
        if (error_code & 0x10)
            printf("  - Instruction fetch\n");
    }

    /* 打印寄存器状态 */
    printf("\nRegister Dump:\n");
    printf("EAX: 0x%x  EBX: 0x%x  ECX: 0x%x  EDX: 0x%x\n",
           regs->eax, regs->ebx, regs->ecx, regs->edx);
    printf("ESI: 0x%x  EDI: 0x%x  EBP: 0x%x  ESP: 0x%x\n",
           regs->esi, regs->edi, regs->ebp, regs->esp);
    printf("EIP: 0x%x  CS: 0x%x  EFLAGS: 0x%x\n",
           regs->eip, regs->cs, regs->eflags);

    printf("\n==================================================\n");
    printf("System halted. Please reboot.\n");
    printf("==================================================\n");

    /* 停止系统 */
    while(1) {
        asm volatile ("hlt");  // Halt指令
    }
}
```

在 `exception_handler.h` 中：

```c
#ifndef _EXCEPTION_HANDLER_H
#define _EXCEPTION_HANDLER_H

#include "types.h"

/* 寄存器上下文结构 */
typedef struct registers {
    /* 段寄存器 */
    uint32_t ds;
    uint32_t es;
    uint32_t fs;
    uint32_t gs;

    /* 通用寄存器（PUSHAL压入的顺序） */
    uint32_t edi;
    uint32_t esi;
    uint32_t ebp;
    uint32_t esp;
    uint32_t ebx;
    uint32_t edx;
    uint32_t ecx;
    uint32_t eax;

    /* 中断号和错误码 */
    uint32_t int_no;
    uint32_t err_code;

    /* CPU自动压入 */
    uint32_t eip;
    uint32_t cs;
    uint32_t eflags;
    uint32_t useresp;
    uint32_t ss;
} registers_t;

/* 异常处理函数 */
void exception_handler(registers_t* regs);

#endif /* _EXCEPTION_HANDLER_H */
```

---

### 步骤3: 实现分页系统

#### 3.1 定义数据结构

在 `paging.h` 中：

```c
#ifndef _PAGING_H
#define _PAGING_H

#include "types.h"

/* 常量定义 */
#define NUM_PDE         1024    // 页目录项数量
#define NUM_PTE         1024    // 页表项数量
#define PAGE_SIZE       4096    // 4KB
#define PAGE_SIZE_4MB   0x400000 // 4MB

#define KERNEL_ADDR     0x400000  // 内核起始地址（4MB）
#define VIDEO_MEM       0xB8000   // 视频内存地址

/* 4KB页的PDE（指向页表） */
typedef struct pde_4kb {
    uint32_t p          : 1;   // Present
    uint32_t rw         : 1;   // Read/Write
    uint32_t us         : 1;   // User/Supervisor
    uint32_t pwt        : 1;   // Page Write Through
    uint32_t pcd        : 1;   // Page Cache Disable
    uint32_t a          : 1;   // Accessed
    uint32_t reserved   : 1;   // Reserved (0)
    uint32_t ps         : 1;   // Page Size (0=4KB)
    uint32_t g          : 1;   // Global
    uint32_t avail      : 3;   // Available for OS
    uint32_t page_table_base_addr : 20; // 页表物理地址[31:12]
} __attribute__((packed)) pde_4kb_t;

/* 4MB页的PDE（直接指向物理地址） */
typedef struct pde_4mb {
    uint32_t p          : 1;   // Present
    uint32_t rw         : 1;   // Read/Write
    uint32_t us         : 1;   // User/Supervisor
    uint32_t pwt        : 1;   // Page Write Through
    uint32_t pcd        : 1;   // Page Cache Disable
    uint32_t a          : 1;   // Accessed
    uint32_t d          : 1;   // Dirty
    uint32_t ps         : 1;   // Page Size (1=4MB)
    uint32_t g          : 1;   // Global
    uint32_t avail      : 3;   // Available for OS
    uint32_t pat        : 1;   // Page Attribute Table
    uint32_t reserved   : 9;   // Reserved (must be 0)
    uint32_t page_base_addr : 10; // 4MB页物理地址[31:22]
} __attribute__((packed)) pde_4mb_t;

/* PDE联合体 */
typedef union pde {
    uint32_t val;
    pde_4kb_t kb;
    pde_4mb_t mb;
} pde_t;

/* PTE结构 */
typedef struct pte {
    uint32_t p          : 1;   // Present
    uint32_t rw         : 1;   // Read/Write
    uint32_t us         : 1;   // User/Supervisor
    uint32_t pwt        : 1;   // Page Write Through
    uint32_t pcd        : 1;   // Page Cache Disable
    uint32_t a          : 1;   // Accessed
    uint32_t d          : 1;   // Dirty
    uint32_t pat        : 1;   // Page Attribute Table
    uint32_t g          : 1;   // Global
    uint32_t avail      : 3;   // Available for OS
    uint32_t page_base_addr : 20; // 页框物理地址[31:12]
} __attribute__((packed)) pte_t;

/* 页目录（1024个PDE，4KB对齐） */
typedef struct page_directory {
    pde_t page_directory[NUM_PDE];
} __attribute__((aligned(PAGE_SIZE))) page_directory_t;

/* 页表（1024个PTE，4KB对齐） */
typedef struct page_table {
    pte_t page_table[NUM_PTE];
} __attribute__((aligned(PAGE_SIZE))) page_table_t;

/* 全局变量声明 */
extern page_directory_t page_directory_array[1];
extern page_table_t page_table_array[1];

/* 函数声明 */
void init_paging();
void set_up_PD_PT();
void enable_paging();
void flush_TLB();
void remap(int32_t virtual_addr, int32_t physical_addr);
void remap_vid(int32_t virtual_addr, int32_t physical_addr);

#endif /* _PAGING_H */
```

#### 3.2 实现分页函数

在 `paging.c` 中：

```c
#include "paging.h"
#include "lib.h"

/* 全局变量：页目录和页表（4KB对齐） */
page_directory_t page_directory_array[1] __attribute__((aligned(PAGE_SIZE)));
page_table_t page_table_array[1] __attribute__((aligned(PAGE_SIZE)));

/* 页目录物理地址 */
uint32_t page_dir_addr;

/*
 * init_paging
 * 描述: 初始化分页系统（主函数）
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 设置页表并启用分页
 */
void init_paging() {
    printf("Initializing paging...\n");

    /* 第一步: 设置页目录和页表 */
    set_up_PD_PT();

    /* 第二步: 启用分页 */
    enable_paging();

    printf("Paging enabled successfully!\n");
}

/*
 * set_up_PD_PT
 * 描述: 设置页目录和页表
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 初始化所有页目录项和页表项
 */
void set_up_PD_PT() {
    int i;

    /* 保存页目录物理地址 */
    page_dir_addr = (uint32_t)page_directory_array;

    /* ===== 第一步: 初始化所有页目录项为"不存在" ===== */
    for (i = 0; i < NUM_PDE; i++) {
        page_directory_array[0].page_directory[i].val = 0;
        page_directory_array[0].page_directory[i].kb.p = 0;  // Not present
    }

    /* ===== 第二步: 初始化所有页表项为"不存在" ===== */
    for (i = 0; i < NUM_PTE; i++) {
        page_table_array[0].page_table[i].p = 0;  // Not present
    }

    /* ===== 第三步: 设置页目录第0项（0-4MB，使用页表） ===== */
    page_directory_array[0].page_directory[0].kb.p = 1;      // Present
    page_directory_array[0].page_directory[0].kb.rw = 1;     // Read/Write
    page_directory_array[0].page_directory[0].kb.us = 0;     // Supervisor
    page_directory_array[0].page_directory[0].kb.pwt = 0;    // Write-back
    page_directory_array[0].page_directory[0].kb.pcd = 0;    // Cache enabled
    page_directory_array[0].page_directory[0].kb.a = 0;      // Not accessed
    page_directory_array[0].page_directory[0].kb.ps = 0;     // 4KB page
    page_directory_array[0].page_directory[0].kb.g = 0;      // Not global
    page_directory_array[0].page_directory[0].kb.avail = 0;  // Available bits
    /* 页表物理地址（高20位） */
    page_directory_array[0].page_directory[0].kb.page_table_base_addr =
        ((uint32_t)page_table_array[0].page_table) >> 12;

    /* ===== 第四步: 设置页表，映射视频内存（0xB8000） ===== */
    /* 0xB8000 / 4096 = 184，所以是页表的第184项 */
    int video_page_index = VIDEO_MEM / PAGE_SIZE;  // 184

    page_table_array[0].page_table[video_page_index].p = 1;         // Present
    page_table_array[0].page_table[video_page_index].rw = 1;        // Read/Write
    page_table_array[0].page_table[video_page_index].us = 0;        // Supervisor
    page_table_array[0].page_table[video_page_index].pwt = 0;       // Write-back
    page_table_array[0].page_table[video_page_index].pcd = 0;       // Cache enabled
    page_table_array[0].page_table[video_page_index].a = 0;         // Not accessed
    page_table_array[0].page_table[video_page_index].d = 0;         // Not dirty
    page_table_array[0].page_table[video_page_index].pat = 0;       // PAT
    page_table_array[0].page_table[video_page_index].g = 0;         // Not global
    page_table_array[0].page_table[video_page_index].avail = 0;     // Available
    /* 视频内存物理地址（高20位）*/
    page_table_array[0].page_table[video_page_index].page_base_addr =
        VIDEO_MEM >> 12;  // 0xB8

    /* ===== 第五步: 设置页目录第1项（4-8MB，内核，4MB大页） ===== */
    page_directory_array[0].page_directory[1].mb.p = 1;      // Present
    page_directory_array[0].page_directory[1].mb.rw = 1;     // Read/Write
    page_directory_array[0].page_directory[1].mb.us = 0;     // Supervisor only
    page_directory_array[0].page_directory[1].mb.pwt = 0;    // Write-back
    page_directory_array[0].page_directory[1].mb.pcd = 0;    // Cache enabled
    page_directory_array[0].page_directory[1].mb.a = 0;      // Not accessed
    page_directory_array[0].page_directory[1].mb.d = 0;      // Not dirty
    page_directory_array[0].page_directory[1].mb.ps = 1;     // 4MB page
    page_directory_array[0].page_directory[1].mb.g = 0;      // Not global
    page_directory_array[0].page_directory[1].mb.avail = 0;  // Available
    page_directory_array[0].page_directory[1].mb.pat = 0;    // PAT
    page_directory_array[0].page_directory[1].mb.reserved = 0; // Must be 0
    /* 内核物理地址 = 4MB = 0x400000 */
    /* 4MB页的物理地址是bits[31:22]，即 0x400000 >> 22 = 1 */
    page_directory_array[0].page_directory[1].mb.page_base_addr =
        KERNEL_ADDR >> 22;  // 1
}

/*
 * enable_paging
 * 描述: 启用分页
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 设置CR3和CR0，启用分页
 */
void enable_paging() {
    /* 加载页目录地址到CR3 */
    asm volatile (
        "movl %0, %%eax         \n"
        "movl %%eax, %%cr3      \n"
        : /* no output */
        : "r" (page_dir_addr)
        : "eax"
    );

    /* 启用CR0的PG位（bit 31） */
    asm volatile (
        "movl %%cr0, %%eax      \n"
        "orl $0x80000000, %%eax \n"  // 设置PG位
        "movl %%eax, %%cr0      \n"
        : /* no output */
        : /* no input */
        : "eax"
    );
}

/*
 * flush_TLB
 * 描述: 刷新TLB（Translation Lookaside Buffer）
 * 输入: 无
 * 输出: 无
 * 返回值: 无
 * 副作用: 重新加载CR3，刷新TLB
 */
void flush_TLB() {
    asm volatile (
        "movl %%cr3, %%eax      \n"
        "movl %%eax, %%cr3      \n"
        : /* no output */
        : /* no input */
        : "eax"
    );
}

/*
 * remap
 * 描述: 重新映射虚拟地址到物理地址（4MB页）
 * 输入: virtual_addr - 虚拟地址
 *       physical_addr - 物理地址
 * 输出: 无
 * 返回值: 无
 * 副作用: 修改页目录，刷新TLB
 * 用途: Checkpoint 4用于映射用户程序
 */
void remap(int32_t virtual_addr, int32_t physical_addr) {
    int32_t pde = virtual_addr / PAGE_SIZE_4MB;  // 页目录索引

    /* 设置4MB页 */
    page_directory_array[0].page_directory[pde].mb.p = 1;
    page_directory_array[0].page_directory[pde].mb.rw = 1;
    page_directory_array[0].page_directory[pde].mb.us = 1;  // User accessible
    page_directory_array[0].page_directory[pde].mb.pwt = 0;
    page_directory_array[0].page_directory[pde].mb.pcd = 0;
    page_directory_array[0].page_directory[pde].mb.a = 0;
    page_directory_array[0].page_directory[pde].mb.d = 0;
    page_directory_array[0].page_directory[pde].mb.ps = 1;  // 4MB
    page_directory_array[0].page_directory[pde].mb.g = 0;
    page_directory_array[0].page_directory[pde].mb.avail = 0;
    page_directory_array[0].page_directory[pde].mb.pat = 0;
    page_directory_array[0].page_directory[pde].mb.reserved = 0;
    page_directory_array[0].page_directory[pde].mb.page_base_addr =
        physical_addr >> 22;

    flush_TLB();
}

/*
 * remap_vid
 * 描述: 重新映射视频内存到用户空间（4KB页）
 * 输入: virtual_addr - 虚拟地址
 *       physical_addr - 物理地址（通常是0xB8000）
 * 输出: 无
 * 返回值: 无
 * 副作用: 修改页目录和页表，刷新TLB
 * 用途: Checkpoint 4的vidmap系统调用
 */
void remap_vid(int32_t virtual_addr, int32_t physical_addr) {
    int32_t pde = virtual_addr / PAGE_SIZE_4MB;  // 页目录索引

    /* 设置页目录项（指向页表） */
    page_directory_array[0].page_directory[pde].kb.p = 1;
    page_directory_array[0].page_directory[pde].kb.rw = 1;
    page_directory_array[0].page_directory[pde].kb.us = 1;  // User accessible
    page_directory_array[0].page_directory[pde].kb.pwt = 0;
    page_directory_array[0].page_directory[pde].kb.pcd = 0;
    page_directory_array[0].page_directory[pde].kb.a = 0;
    page_directory_array[0].page_directory[pde].kb.ps = 0;  // 4KB pages
    page_directory_array[0].page_directory[pde].kb.g = 0;
    page_directory_array[0].page_directory[pde].kb.avail = 0;
    page_directory_array[0].page_directory[pde].kb.page_table_base_addr =
        ((uint32_t)page_table_array[0].page_table) >> 12;

    /* 设置页表项（映射到视频内存） */
    page_table_array[0].page_table[0].p = 1;
    page_table_array[0].page_table[0].rw = 1;
    page_table_array[0].page_table[0].us = 1;  // User accessible
    page_table_array[0].page_table[0].page_base_addr = physical_addr >> 12;

    flush_TLB();
}
```

---

### 步骤4: 在kernel.c中调用初始化函数

修改 `kernel.c` 的 `entry()` 函数：

```c
void entry(unsigned long magic, unsigned long addr) {
    /* ... Multiboot信息打印代码 ... */

    /* 初始化IDT */
    printf("Initializing IDT...\n");
    initialize_IDT();
    printf("IDT initialized.\n");

    /* 初始化分页 */
    printf("Initializing paging...\n");
    init_paging();
    printf("Paging enabled.\n");

    /* 启用中断（Checkpoint 2需要） */
    // sti();  // 暂时不要启用，等Checkpoint 2实现中断处理

    /* 运行测试 */
    #ifdef RUN_TESTS
    launch_tests();
    #endif

    /* 内核主循环 */
    while(1);
}
```

---

### 步骤5: 编写和运行测试

#### 5.1 查看现有测试

查看 `tests.c` 中的Checkpoint 1测试：

```c
/* Checkpoint 1 tests */

// IDT测试
int idt_test();

// 分页有效性测试
int paging_valid_test();

// 分页解引用测试
int paging_dereference_test();

// 异常测试（会触发异常）
int exception_test();
```

#### 5.2 运行测试

修改 `kernel.c` 启用测试：

```c
#define RUN_TESTS    // 取消注释这一行
```

修改 `tests.c` 的 `launch_tests()` 函数：

```c
void launch_tests() {
    // 测试IDT
    TEST_OUTPUT("idt_test", idt_test());

    // 测试分页有效性
    TEST_OUTPUT("paging_valid_test", paging_valid_test());

    // 测试分页解引用
    TEST_OUTPUT("paging_dereference_test", paging_dereference_test());

    // 测试异常（注意：这会触发异常，系统会停止）
    // TEST_OUTPUT("exception_test", exception_test());
}
```

#### 5.3 编译和运行

```bash
cd student-distrib
make clean
make

# 运行QEMU
qemu-system-i386 -hda ../mp3.img -m 64
```

#### 5.4 测试异常处理

取消 `exception_test()` 的注释，测试异常处理：

```c
void launch_tests() {
    // ... 其他测试 ...

    // 测试异常（会触发除零异常）
    TEST_OUTPUT("exception_test", exception_test());
}
```

重新编译运行，应该看到红屏的异常信息。

---

## 📝 调试技巧

### 使用QEMU + GDB调试

```bash
# Terminal 1: 启动QEMU并等待GDB连接
qemu-system-i386 -s -S -hda ../mp3.img -m 64

# Terminal 2: 启动GDB
gdb
(gdb) target remote localhost:1234
(gdb) symbol-file kernel
(gdb) break init_paging
(gdb) continue
(gdb) step
(gdb) print page_directory_array[0].page_directory[0]
(gdb) x/10x page_directory_array
```

### 打印调试信息

在关键位置添加 `printf`：

```c
void set_up_PD_PT() {
    printf("Setting up page directory at 0x%x\n", page_dir_addr);
    printf("Setting up page table at 0x%x\n", (uint32_t)page_table_array);

    /* ... */

    printf("PD[0] = 0x%x\n", page_directory_array[0].page_directory[0].val);
    printf("PD[1] = 0x%x\n", page_directory_array[0].page_directory[1].val);
    printf("PT[184] = 0x%x\n", page_table_array[0].page_table[184].page_base_addr);
}
```

---

## ✅ 验收标准

Checkpoint 1 完成后，你应该能够：

### 功能验收

- [ ] 系统成功启动，进入保护模式
- [ ] IDT正确初始化，所有256个表项设置完毕
- [ ] 分页系统启用，能够访问视频内存（0xB8000）
- [ ] 能够访问内核内存（4-8MB）
- [ ] 访问未映射内存会触发页错误
- [ ] 触发除零异常时正确显示异常信息
- [ ] 所有Checkpoint 1测试通过

### 理解验收

你应该能够回答：

1. GDT的作用是什么？段选择子如何使用？
2. IDT的作用是什么？中断门和陷阱门的区别？
3. 页目录和页表如何协同工作？
4. 虚拟地址如何转换为物理地址？
5. 为什么需要TLB？何时需要刷新TLB？
6. 页错误的常见原因有哪些？
7. CR0、CR2、CR3寄存器的作用分别是什么？

---

## 📚 推荐阅读（按顺序）

### 第1周：理论学习

1. **OSTEP**: 第13-15章（地址空间、地址转换API、机制）- 3小时
2. **OSTEP**: 第18-20章（分页介绍、TLB、更小的表）- 5小时
3. **Intel手册**: Volume 3A, Chapter 3（保护模式内存管理）- 5小时
4. **Intel手册**: Volume 3A, Chapter 4（分页）- 5小时
5. **Intel手册**: Volume 3A, Chapter 6（中断和异常处理）- 4小时

### 第2周：实践与参考

6. **xv6 book**: Chapter 2（页表）- 2小时
7. **xv6源码**: `vm.c`（虚拟内存实现）- 3小时
8. **xv6源码**: `trap.c`（中断处理）- 2小时
9. **OSDev Wiki**: Paging, IDT, Exceptions教程 - 3小时

**总计约32小时理论学习 + 实践时间**

---

## 🎓 深入理解问题

完成代码后，思考这些问题以加深理解：

### 关于IDT

1. 为什么系统调用要用陷阱门而不是中断门？
2. 为什么系统调用的DPL=3，而异常的DPL=0？
3. 如果IDT表项的present位为0会发生什么？
4. CPU如何知道IDT在哪里？（IDTR寄存器）

### 关于异常

5. 为什么有些异常有错误码，有些没有？
6. 页错误时，如何知道是哪个地址引起的？（CR2）
7. 双重错误（Double Fault）是什么时候发生的？
8. 为什么需要汇编包装函数，不能直接用C函数？

### 关于分页

9. 为什么内核用4MB大页，而视频内存用4KB页？
10. 如果不刷新TLB会发生什么问题？
11. 为什么页目录和页表要4KB对齐？
12. 两级页表相比单级页表有什么优势？
13. 如何计算一个虚拟地址对应的PDE和PTE索引？
14. Present位、Read/Write位、User/Supervisor位分别什么作用？

---

## 🚀 下一步

完成Checkpoint 1后，你将进入Checkpoint 2：设备驱动。

你将学习：

- 8259 PIC（可编程中断控制器）
- 键盘驱动（扫描码处理）
- RTC驱动（实时时钟）
- 终端驱动（输入输出）

**准备工作**：

- 阅读OSTEP第36章（I/O设备）
- 浏览OSDev Wiki的8259 PIC、Keyboard、RTC页面

---

**加油！系统编程的旅程才刚刚开始！** 🎉
