# RTC 驱动深度解析

## 概述

RTC (Real Time Clock，实时时钟) 是一个独立于 CPU 的硬件时钟芯片。它可以：

1. 保持当前时间（即使电脑关机，靠 CMOS 电池供电）
2. 产生周期性中断（用于定时任务）

在这个 OS 中，RTC 主要用于**周期性中断**功能。

## 硬件架构

### RTC 与 CMOS

```
┌─────────────────────────────────────────────────────────────┐
│                      主板 (Motherboard)                      │
│  ┌─────────────┐         ┌─────────────┐         ┌────────┐│
│  │    CPU      │◄────────│     PIC     │◄────────│  RTC   ││
│  │             │  IRQ8   │   (8259)    │  IRQ8   │ (MC146818)│
│  └─────────────┘         └─────────────┘         └────────┘│
│                                                      │      │
│                                                      │      │
│                               ┌──────────────────────┘      │
│                               │                             │
│                               ▼                             │
│                          ┌────────┐                         │
│                          │  CMOS  │◄─── 电池供电            │
│                          │  RAM   │     (保持时间/设置)      │
│                          └────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

### I/O 端口

| 端口   | 名称        | 功能               |
|------|-----------|------------------|
| 0x70 | RTC_PORT  | 索引寄存器（选择要访问的寄存器） |
| 0x71 | CMOS_PORT | 数据寄存器（读写选中的寄存器）  |

### 访问 RTC 寄存器的方式

```c
// 两步访问法：
// 1. 写索引到 0x70
// 2. 读/写数据到 0x71

// 读取寄存器 X:
outb(X, 0x70);           // 选择寄存器 X
value = inb(0x71);       // 读取值

// 写入寄存器 X:
outb(X, 0x70);           // 选择寄存器 X
outb(value, 0x71);       // 写入值
```

## RTC 寄存器详解

### 寄存器 A (0x0A) - 频率控制

```
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
├───┴───┴───┴───┼───┴───┴───┴───┤
│    DV[2:0]    │    RS[3:0]    │
│   振荡器设置   │   频率选择     │
└───────────────┴───────────────┘
```

**RS[3:0] 频率对照表**：

| RS 值 | 二进制  | 频率      |
|------|------|---------|
| 0x06 | 0110 | 1024 Hz |
| 0x07 | 0111 | 512 Hz  |
| 0x08 | 1000 | 256 Hz  |
| 0x09 | 1001 | 128 Hz  |
| 0x0A | 1010 | 64 Hz   |
| 0x0B | 1011 | 32 Hz   |
| 0x0C | 1100 | 16 Hz   |
| 0x0D | 1101 | 8 Hz    |
| 0x0E | 1110 | 4 Hz    |
| 0x0F | 1111 | 2 Hz    |

**公式**：`频率 = 32768 >> (RS - 1)`（RS = 1 时为 32768 Hz）

### 寄存器 B (0x0B) - 中断使能

```
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
├───┼───┼───┼───┼───┼───┼───┼───┤
│SET│PIE│AIE│UIE│SQWE│DM │24/12│DSE│
└───┴───┴───┴───┴───┴───┴───┴───┘
```

| 位     | 名称      | 功能                                     |
|-------|---------|----------------------------------------|
| 7     | SET     | 设置模式（1=停止更新，允许设置时间）                    |
| **6** | **PIE** | **周期性中断使能**（Periodic Interrupt Enable） |
| 5     | AIE     | 闹钟中断使能                                 |
| 4     | UIE     | 更新结束中断使能                               |
| 3     | SQWE    | 方波输出使能                                 |
| 2     | DM      | 数据模式（0=BCD，1=二进制）                      |
| 1     | 24/12   | 小时格式（0=12小时，1=24小时）                    |
| 0     | DSE     | 夏令时使能                                  |

**本项目只使用 PIE (bit 6)**：

```c
#define RTC_PIE 0x40  // 0100 0000，开启周期性中断
```

### 寄存器 C (0x0C) - 中断标志

```
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
├───┼───┼───┼───┼───┴───┴───┴───┤
│IRQF│PF│AF│UF│    Reserved     │
└───┴───┴───┴───┴───────────────┘
```

| 位 | 名称   | 功能              |
|---|------|-----------------|
| 7 | IRQF | IRQ 标志（有任何中断发生） |
| 6 | PF   | 周期性中断标志         |
| 5 | AF   | 闹钟中断标志          |
| 4 | UF   | 更新结束中断标志        |

**重要**：读取寄存器 C 会**清除所有中断标志**，这是确认中断所必需的！

### NMI 位 (bit 7 of port 0x70)

```c
#define NMI_BIT 0x80  // 1000 0000
```

当写入端口 0x70 时，bit 7 控制 NMI (Non-Maskable Interrupt)：

- bit 7 = 1：禁用 NMI
- bit 7 = 0：启用 NMI

```c
outb(RTC_REG_A | NMI_BIT, RTC_PORT);  // 选择寄存器 A，同时禁用 NMI
```

## RTC 初始化

```c
void rtc_init() {
    char prevA, prevB;

    // 1. 先禁用 RTC 中断
    disable_irq(RTC_IRQ_NUM);  // IRQ 8

    // 2. 设置频率（寄存器 A）
    outb(RTC_REG_A | NMI_BIT, RTC_PORT);  // 选择寄存器 A，禁用 NMI
    prevA = inb(CMOS_PORT);               // 读取当前值
    outb(RTC_REG_A, RTC_PORT);            // 再次选择寄存器 A
    outb(MAX_FREQ | prevA, CMOS_PORT);    // 设置频率为 1024 Hz

    // 3. 开启周期性中断（寄存器 B）
    outb(RTC_REG_B | NMI_BIT, RTC_PORT);  // 选择寄存器 B
    prevB = inb(CMOS_PORT);               // 读取当前值
    outb(RTC_REG_B | NMI_BIT, RTC_PORT);  // 重新选择（读操作会重置索引）
    outb(prevB | RTC_PIE, CMOS_PORT);     // 设置 PIE 位

    // 4. 设置初始频率
    rtc_set_freq(MAX_RTC_FREQ);           // 1024 Hz

    // 5. 启用中断
    sti();
    enable_irq(RTC_IRQ_NUM);
}
```

### 初始化流程图

```
rtc_init()
    │
    ├── disable_irq(8)         // 暂时禁用 RTC 中断
    │
    ├── 设置寄存器 A
    │   ├── outb(0x8A, 0x70)   // 选择 regA，禁用 NMI
    │   ├── inb(0x71)          // 读取当前值
    │   ├── outb(0x0A, 0x70)   // 重新选择 regA
    │   └── outb(rate, 0x71)   // 写入频率设置
    │
    ├── 设置寄存器 B
    │   ├── outb(0x8B, 0x70)   // 选择 regB
    │   ├── inb(0x71)          // 读取当前值
    │   ├── outb(0x8B, 0x70)   // 重新选择（重要！）
    │   └── outb(prev|0x40, 0x71) // 开启 PIE
    │
    ├── rtc_set_freq(1024)     // 设置频率
    │
    └── enable_irq(8)          // 启用 RTC 中断
```

## 中断处理

### rtc_interrupt_handler

```c
volatile int rtc_interrupt_occurred[NUM_TERM] = {0, 0, 0};

void rtc_interrupt_handler() {
    cli();

    // 1. 读取寄存器 C 以确认中断（必须！否则不会再产生中断）
    outb(RTC_REG_C, RTC_PORT);
    inb(CMOS_PORT);  // 丢弃内容，只为清除中断标志

    // 2. 设置所有终端的中断标志
    for (int i = 0; i < NUM_TERM; i++) {
        rtc_interrupt_occurred[i] = 1;
    }

    // 3. 发送 EOI
    send_eoi(RTC_IRQ_NUM);

    sti();
}
```

**为什么要读寄存器 C？**

RTC 有一个"确认中断"机制：如果不读取寄存器 C，RTC 会认为中断还没被处理，不会产生下一次中断。这是硬件级别的设计，防止中断丢失或重复处理。

## 频率虚拟化

### 问题

硬件 RTC 只能设置 2 的幂次方频率（2, 4, 8, ... 1024 Hz）。但不同用户程序可能需要不同的频率，而且频繁改变硬件频率会影响系统性能。

### 解决方案：软件虚拟化

**核心思想**：

- 硬件固定运行在最高频率（1024 Hz）
- 软件通过计数器模拟低频率

```
硬件频率: 1024 Hz (每秒 1024 次中断)

用户程序 A 想要 4 Hz:
  需要等待 1024/4 = 256 次硬件中断才返回一次

用户程序 B 想要 32 Hz:
  需要等待 1024/32 = 32 次硬件中断才返回一次
```

### 实现

每个终端有自己的虚拟 RTC 频率：

```c
// 在 term_info 结构中
typedef struct {
    ...
    uint32_t rtc_freq;  // 该终端的虚拟 RTC 频率
    ...
} term_info;
```

### rtc_open - 初始化为 2 Hz

```c
int rtc_open(const uint8_t* filename) {
    term[running_term].rtc_freq = MIN_RTC_FREQ;  // 默认 2 Hz
    return 0;
}
```

### rtc_write - 设置虚拟频率

```c
int rtc_write(int32_t fd, const void* buf, int32_t nbytes) {
    int32_t interrupt_freq = *((int32_t*)buf);

    // 检查频率范围
    if (interrupt_freq < MIN_RTC_FREQ || interrupt_freq > MAX_RTC_FREQ)
        return -1;

    // 不改变硬件频率，只改变该终端的虚拟频率
    term[running_term].rtc_freq = interrupt_freq;
    return 4;  // 返回写入的字节数
}
```

### rtc_read - 等待虚拟中断

```c
int rtc_read(int32_t fd, void* buf, int32_t nbytes) {
    int count, i, num_running;

    // 1. 计算需要等待多少次硬件中断
    num_running = 1;
    count = MAX_RTC_FREQ / term[running_term].rtc_freq;

    // 2. 考虑多终端调度
    if (term[1].running_pid != -1) num_running++;
    if (term[2].running_pid != -1) num_running++;
    count /= num_running;

    // 3. 等待 count 次硬件中断
    for (i = 0; i < count; i++) {
        while (!rtc_interrupt_occurred[running_term]) {}  // 自旋等待
        rtc_interrupt_occurred[running_term] = 0;
    }

    return 0;
}
```

### 虚拟化计算示例

```
场景: 只有终端 0 运行，程序设置 rtc_freq = 64

count = 1024 / 64 = 16
num_running = 1
count = 16 / 1 = 16

rtc_read 需要等待 16 次硬件中断才返回
实际效果: 1024 Hz / 16 = 64 Hz ✓


场景: 三个终端都运行，终端 0 的程序设置 rtc_freq = 64

count = 1024 / 64 = 16
num_running = 3
count = 16 / 3 = 5

rtc_read 只需等待 5 次硬件中断就返回
这是为了补偿调度带来的延迟
```

### 为什么要除以 num_running？

因为有多终端调度，CPU 时间被多个终端分享。如果终端 0 的程序设置了 64 Hz，但 CPU 只有 1/3 的时间执行终端 0，那实际频率会变成约
21 Hz。

通过除以活动终端数，可以近似补偿这个问题：

- 等待更少的中断次数
- 让用户感知的频率接近设置值

## 完整流程图

### RTC 中断流程

```
                    RTC 芯片
                       │
                       │ 1024 Hz 中断
                       ▼
                    IRQ 8
                       │
                       ▼
              PIC (从片 + 主片)
                       │
                       ▼
                    CPU
                       │
                       ▼
           rtc_interrupt_handler()
                       │
                       ├── 读取 Reg C (确认中断)
                       │
                       ├── rtc_interrupt_occurred[*] = 1
                       │
                       └── send_eoi(8)
```

### rtc_read 等待流程

```
用户程序调用 rtc_read()
           │
           ▼
    计算 count = 1024 / freq
           │
           ▼
    ┌──────────────────────┐
    │  for (i = 0; i < count)│
    │          │            │
    │          ▼            │
    │  ┌───────────────────┐│
    │  │ while(!occurred)  ││◄── 自旋等待中断
    │  └───────────────────┘│
    │          │            │
    │          ▼            │
    │  occurred = 0         │
    │          │            │
    └──────────┼────────────┘
               │
               ▼
        返回 0 (成功)
```

## rtc_set_freq 函数

这个函数用于直接设置硬件频率（初始化时使用）：

```c
int rtc_set_freq(int32_t freq) {
    char rate;
    int valid = 0;

    // 保存寄存器 A 的当前值
    outb(RTC_REG_A, RTC_PORT);
    unsigned char prevA = inb(CMOS_PORT);

    // 根据频率确定 rate 值
    if (freq == 1024) { rate = 0x06; valid = 1; }
    if (freq == 512)  { rate = 0x07; valid = 1; }
    if (freq == 256)  { rate = 0x08; valid = 1; }
    if (freq == 128)  { rate = 0x09; valid = 1; }
    if (freq == 64)   { rate = 0x0A; valid = 1; }
    if (freq == 32)   { rate = 0x0B; valid = 1; }
    if (freq == 16)   { rate = 0x0C; valid = 1; }
    if (freq == 8)    { rate = 0x0D; valid = 1; }
    if (freq == 4)    { rate = 0x0E; valid = 1; }
    if (freq == 2)    { rate = 0x0F; valid = 1; }

    // 写入新的频率设置（保留高 4 位，修改低 4 位）
    outb(RTC_REG_A, RTC_PORT);
    outb(((prevA >> 4) << 4) | rate, CMOS_PORT);

    if (valid) return 4;
    return -1;
}
```

**位操作解释**：

```c
((prevA >> 4) << 4) | rate

prevA = 0010 0110  (原值)
            ↓
prevA >> 4 = 0000 0010
            ↓
<< 4 = 0010 0000  (保留高4位，低4位清零)
            ↓
| rate = 0010 0110  (新频率写入低4位)
```

## 常见问题解答

### Q1: 为什么 RTC 用 IRQ 8 而不是 IRQ 0-7？

IRQ 0-7 在主 PIC 上，IRQ 8-15 在从 PIC 上。RTC 传统上连接到从 PIC 的 IRQ 0（即系统 IRQ 8）。

```
主 PIC (IRQ 0-7):
  IRQ 0 - PIT (定时器)
  IRQ 1 - 键盘
  IRQ 2 - 级联到从 PIC
  ...

从 PIC (IRQ 8-15):
  IRQ 8 - RTC
  IRQ 9-15 - 其他设备
```

### Q2: 为什么读取 Reg C 后要丢弃内容？

```c
outb(RTC_REG_C, RTC_PORT);
inb(CMOS_PORT);  // 丢弃
```

我们只需要"读取"这个动作来清除中断标志，内容本身不重要。如果需要知道具体是哪种中断（周期/闹钟/更新），才需要分析返回值。

### Q3: 为什么 rtc_interrupt_occurred 是数组？

```c
volatile int rtc_interrupt_occurred[NUM_TERM] = {0, 0, 0};
```

每个终端独立跟踪 RTC 中断状态。这样：

- 终端 0 等待 RTC 时，不会因为终端 1 清除了标志而受影响
- 每个终端可以有自己的 RTC 等待节奏

### Q4: volatile 在这里的作用？

```c
volatile int rtc_interrupt_occurred[NUM_TERM];
```

`rtc_interrupt_occurred` 被中断处理程序和 `rtc_read` 同时访问：

- 中断处理程序：写入 1
- rtc_read：读取并写入 0

没有 `volatile`，编译器可能优化掉 while 循环中的重复读取。

### Q5: 为什么要保留 Reg A/B 的其他位？

```c
outb(prevB | RTC_PIE, CMOS_PORT);  // 只修改 PIE 位
```

Reg A 和 Reg B 的其他位控制 RTC 的其他功能（振荡器、时间格式等）。我们只想修改特定位，不应该破坏其他设置。

## 调试技巧

### 测试 RTC 中断

```c
// 在 rtc_interrupt_handler 中添加：
void rtc_interrupt_handler() {
    static int count = 0;
    if (++count % 1024 == 0) {
        printf("RTC: 1 second passed\n");  // 每秒打印一次
    }
    // ... 原有代码
}
```

### 检查 RTC 频率

```c
// 用户程序中：
int freq = 32;
rtc_write(fd, &freq, 4);

int start = get_time();
for (int i = 0; i < 32; i++) {
    rtc_read(fd, NULL, 0);
}
int end = get_time();
// end - start 应该约等于 1 秒
```

## 总结

RTC 驱动的核心机制：

| 功能    | 实现方式                                      |
|-------|-------------------------------------------|
| 硬件访问  | 通过 I/O 端口 0x70/0x71 访问 CMOS 寄存器           |
| 频率控制  | 寄存器 A 的低 4 位 (RS[3:0])                    |
| 中断使能  | 寄存器 B 的 bit 6 (PIE)                       |
| 中断确认  | 读取寄存器 C                                   |
| 频率虚拟化 | 硬件固定 1024Hz，软件计数模拟低频率                     |
| 多终端支持 | 每个终端独立的 rtc_freq 和 rtc_interrupt_occurred |

核心代码流程：

```
rtc_init() → 设置 1024Hz，开启 PIE
     │
     ▼
硬件每秒产生 1024 次中断
     │
     ▼
rtc_interrupt_handler() → 设置标志，确认中断
     │
     ▼
rtc_read() → 等待 (1024/虚拟频率) 次中断后返回
```
