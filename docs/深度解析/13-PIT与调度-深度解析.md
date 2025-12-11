# PIT 与进程调度深度解析

## 概述

Checkpoint 5 实现了基于 PIT（Programmable Interval Timer）的进程调度。PIT
定期产生中断，触发调度器在多个终端的进程之间切换，实现"伪并行"的多任务效果。

## PIT 硬件

### PIT 简介

PIT (Intel 8253/8254) 是一个硬件定时器，可以按设定的频率产生中断。

```
PIT 硬件架构:
┌────────────────────────────────────────────┐
│                   PIT                      │
│  ┌─────────────────────────────────────┐   │
│  │  计数器 0 (Channel 0)                │   │
│  │  - 连接到 IRQ 0                      │   │
│  │  - 用于系统定时器                     │   │
│  └─────────────────────────────────────┘   │
│  ┌─────────────────────────────────────┐   │
│  │  计数器 1 (Channel 1) - 历史遗留      │   │
│  └─────────────────────────────────────┘   │
│  ┌─────────────────────────────────────┐   │
│  │  计数器 2 (Channel 2) - PC 扬声器     │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘
              │
              │ IRQ 0
              ▼
┌────────────────────────────────────────────┐
│                  PIC                       │
│         (可编程中断控制器)                    │
└────────────────────────────────────────────┘
              │
              ▼
┌────────────────────────────────────────────┐
│                  CPU                        │
│         pit_interrupt_handler()             │
└────────────────────────────────────────────┘
```

### PIT 端口

| 端口   | 名称               | 用途           |
|------|------------------|--------------|
| 0x40 | Channel 0 Data   | 设置计数器 0 的计数值 |
| 0x41 | Channel 1 Data   | 设置计数器 1 的计数值 |
| 0x42 | Channel 2 Data   | 设置计数器 2 的计数值 |
| 0x43 | Command Register | 配置 PIT 工作模式  |

### 频率计算

PIT 的基础频率是 1,193,180 Hz。通过设置分频值来得到期望的中断频率：

```
中断频率 = 1,193,180 / 分频值

本项目: 分频值 = 11932
中断频率 = 1,193,180 / 11932 ≈ 100 Hz
中断间隔 = 1000ms / 100 = 10ms
```

## PIT 初始化

```c
// pit.h
#define PIT_IRQ_NUM     0        // IRQ 0
#define CMD_PORT        0x43     // 命令端口
#define DATA_PORT0      0x40     // 数据端口
#define CMD_REG_VAL     0x36     // 命令字
#define PIT_MAX_FREQ    1193180  // 基础频率
#define PIT_FREQ        11932    // 分频值 (≈100Hz)
#define FREQ_MASK       0xFF     // 低 8 位掩码

// pit.c
void pit_init() {
    // 1. 写入命令字
    outb(CMD_REG_VAL, CMD_PORT);

    // 2. 写入分频值低 8 位
    outb(PIT_FREQ & FREQ_MASK, DATA_PORT0);

    // 3. 写入分频值高 8 位
    outb(PIT_FREQ >> 8, DATA_PORT0);

    // 4. 启用 IRQ 0
    enable_irq(PIT_IRQ_NUM);
}
```

### 命令字解析 (0x36)

```
CMD_REG_VAL = 0x36 = 0011 0110

位 7-6: 00 = 选择计数器 0
位 5-4: 11 = 先写低字节，再写高字节
位 3-1: 011 = 模式 3 (方波发生器)
位 0:   0 = 二进制计数
```

## 中断处理

### pit_interrupt_handler()

```c
void pit_interrupt_handler() {
    // 1. 发送 EOI (End of Interrupt)
    send_eoi(PIT_IRQ_NUM);

    cli();

    // 2. 检查是否需要调度
    //    只有当终端 1 或 2 有进程时才调度
    if (term[1].running_pid != -1 || term[2].running_pid != -1) {
        next_process = get_next_process();
        schedule(next_process);
    }

    sti();
}
```

### 调度条件

| 条件           | 行为        |
|--------------|-----------|
| 只有终端 0 有进程   | 不调度，继续运行  |
| 终端 1 或 2 有进程 | 触发调度，轮转切换 |

这是因为如果只有一个终端有进程，调度没有意义。

## Round-Robin 调度

### get_next_process() - 寻找下一个进程

```c
uint32_t get_next_process() {
    int i;
    next_term = running_term;

    // 循环查找下一个有进程的终端
    for (i = 0; i < NUM_TERMS; i++) {
        next_term = (next_term + 1) % NUM_TERMS;  // 0→1→2→0→...
        if (term[next_term].running_pid != -1) {
            break;  // 找到有进程的终端
        }
    }

    return term[next_term].running_pid;
}
```

**轮转示例**：

```
假设: 终端0有pid=0, 终端1有pid=1, 终端2无进程

时刻    running_term    next_term    执行的进程
────────────────────────────────────────────────
T0      0               0→1          pid=1
T1      1               1→2→0        pid=0
T2      0               0→1          pid=1
T3      1               1→2→0        pid=0
...

终端2被跳过因为 running_pid == -1
```

## 上下文切换

### schedule() 函数详解

```c
void schedule(uint32_t process) {
    // ========== 1. 视频内存处理 ==========
    uint8_t* screen_start;
    vidmap(&screen_start);  // 获取 132MB 虚拟地址

    term_info new_terminal = term[next_term];

    // 如果切换到非活动终端，重定向视频输出到备份区
    if (new_terminal.term_id != curr_term) {
        remap_vid((int32_t)screen_start,
                  (int32_t)new_terminal.vid_backup);
    }

    // ========== 2. 保存当前进程状态 ==========
    // 保存 TSS
    term[running_term].esp0 = tss.esp0;
    term[running_term].ss0 = tss.ss0;

    // 获取当前和新 PCB
    pcb_t* curr_pcb = get_specific_pcb(term[running_term].running_pid);
    pcb_t* new_pcb = get_specific_pcb(process);

    // 保存当前 ESP/EBP
    asm volatile(
        "movl %%esp, %%eax;"
        "movl %%ebp, %%ebx;"
        : "=a"(curr_pcb->curr_esp), "=b"(curr_pcb->curr_ebp)
    );

    // ========== 3. 切换到新进程 ==========
    running_term = next_term;

    // 切换页表: 128MB 虚拟地址 → 新进程的物理地址
    remap(_128MB, _8MB + process * _4MB);

    // 恢复 TSS
    tss.ss0 = new_terminal.ss0;
    tss.esp0 = new_terminal.esp0;

    // 更新当前 pid
    cur_pid = process;

    // ========== 4. 恢复新进程的栈并返回 ==========
    asm volatile(
        "movl %%eax, %%esp;"
        "movl %%ebx, %%ebp;"
        "leave;"
        "ret;"
        :
        : "a"(new_pcb->curr_esp), "b"(new_pcb->curr_ebp)
    );
}
```

### 上下文切换流程图

```
PIT 中断发生 (每 10ms)
           │
           ▼
┌─────────────────────────────────────────────┐
│  pit_interrupt_handler()                    │
│  1. send_eoi(0)                             │
│  2. 检查是否需要调度                          │
└─────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  get_next_process()                         │
│  循环: (running_term+1) % 3                  │
│  找到下一个有进程的终端                        │
└─────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  schedule(next_process)                     │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │ 1. 视频内存重映射                     │    │
│  │    如果目标是后台终端，输出到备份区     │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │ 2. 保存当前进程                      │    │
│  │    - term[].esp0, ss0 (TSS)         │    │
│  │    - pcb->curr_esp, curr_ebp (栈)    │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │ 3. 切换页表                         │    │
│  │    remap(128MB → 新进程物理地址)      │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │ 4. 恢复新进程                       │    │
│  │    - 恢复 TSS                       │    │
│  │    - 恢复 ESP/EBP                   │    │
│  │    - leave; ret;                   │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
           │
           ▼
    新进程继续执行
```

## 栈切换机制

### 为什么要保存 ESP/EBP？

进程执行时的状态存储在栈中。切换进程 = 切换栈。

```
进程 A 的栈:                   进程 B 的栈:
┌─────────────┐               ┌─────────────┐
│  局部变量    │               │  局部变量    │
├─────────────┤               ├─────────────┤
│  返回地址    │               │  返回地址    │
├─────────────┤               ├─────────────┤
│  保存的EBP  │               │  保存的EBP   │
├─────────────┤               ├─────────────┤
│     ...     │               │     ...     │
└─────────────┘               └─────────────┘
       ↑                             ↑
      ESP                           ESP
```

### leave; ret; 的作用

```asm
leave   ; 等同于: mov esp, ebp
        ;         pop ebp

ret     ; 等同于: pop eip
        ;         (跳转到栈上保存的返回地址)
```

通过恢复新进程的 ESP/EBP，然后 `leave; ret;`，CPU 会：

1. 将 ESP 设置为新进程的栈顶
2. 从新进程的栈上恢复 EBP
3. 从新进程的栈上 pop 返回地址到 EIP
4. 跳转到新进程上次被中断的位置继续执行

## 页表切换

### remap() 函数

切换进程时需要切换页表，让虚拟地址 128MB 指向新进程的物理内存：

```c
remap(_128MB, _8MB + process * _4MB);
```

**内存映射变化**：

```
进程 0 (pid=0):
虚拟 128MB ─────→ 物理 8MB (8+0*4=8)

切换到进程 1 (pid=1):
虚拟 128MB ─────→ 物理 12MB (8+1*4=12)

切换到进程 2 (pid=2):
虚拟 128MB ─────→ 物理 16MB (8+2*4=16)
```

## TSS 更新

### 为什么要更新 TSS？

TSS（Task State Segment）存储了特权级切换时需要的内核栈信息。

当用户程序发生中断/系统调用时，CPU 需要切换到内核栈。TSS 告诉 CPU 内核栈在哪里：

```c
tss.ss0 = KERNEL_DS;              // 内核数据段
tss.esp0 = 内核栈顶地址;           // 内核栈指针
```

每个进程有自己的内核栈，所以切换进程时必须更新 TSS：

```c
// 保存旧进程的 TSS
term[running_term].esp0 = tss.esp0;
term[running_term].ss0 = tss.ss0;

// 恢复新进程的 TSS
tss.ss0 = new_terminal.ss0;
tss.esp0 = new_terminal.esp0;
```

## 视频内存与调度

### 问题：后台进程的输出去哪里？

当调度器切换到后台终端的进程时，该进程的输出不应该显示在屏幕上（会干扰用户正在看的终端）。

### 解决：remap_vid()

```c
if (new_terminal.term_id != curr_term) {
    // 后台终端：输出重定向到视频备份区
    remap_vid((int32_t)screen_start,
              (int32_t)new_terminal.vid_backup);
}
```

这样，当后台进程调用 `terminal_write()` 时：

- vidmap 返回的地址指向备份区而非显存
- 输出被"存储"在备份区
- 用户切换到该终端时才会看到

## 调度时序

```
时间轴 (每格 10ms):
│
├── T0: PIT 中断，调度 终端0→终端1
│       running_term = 1
│       执行终端1的进程
│
├── T1: PIT 中断，调度 终端1→终端2 (假设有进程)
│       running_term = 2
│       执行终端2的进程
│
├── T2: PIT 中断，调度 终端2→终端0
│       running_term = 0
│       执行终端0的进程
│
├── T3: PIT 中断，调度 终端0→终端1
│       ...
│
▼

每个终端的进程获得约 10ms 的 CPU 时间片
3 个终端轮转一圈约 30ms
```

## 调度与终端切换的区别

| 操作            | 触发方式  | 改变什么                 | 用户感知 |
|---------------|-------|----------------------|------|
| 终端切换 (Alt+Fx) | 用户按键  | `curr_term`，显示内容     | 屏幕切换 |
| 进程调度 (PIT)    | 定时器中断 | `running_term`，执行的进程 | 无感知  |

两者独立运作：

- 用户可以看着终端 0，但 CPU 在执行终端 1 的进程
- 调度器不改变用户看到的内容

## PCB 中的调度相关字段

```c
typedef struct pcb {
    // ... 其他字段 ...

    uint32_t curr_esp;    // 保存的栈指针
    uint32_t curr_ebp;    // 保存的基址指针

    // ... 其他字段 ...
} pcb_t;
```

这两个字段在 `schedule()` 中保存和恢复，是上下文切换的核心。

## 总结

PIT 和调度的核心机制：

| 组件                        | 作用                   |
|---------------------------|----------------------|
| PIT 硬件                    | 每 10ms 产生一次 IRQ 0 中断 |
| `pit_init()`              | 初始化 PIT，设置 100Hz 频率  |
| `pit_interrupt_handler()` | 中断处理，触发调度            |
| `get_next_process()`      | Round-Robin 寻找下一个进程  |
| `schedule()`              | 执行上下文切换              |
| `curr_esp/curr_ebp`       | 保存/恢复执行状态            |
| TSS                       | 保存内核栈信息              |
| `remap()`                 | 切换进程的页表              |
| `remap_vid()`             | 重定向后台终端的视频输出         |

关键设计思想：

1. **时间片轮转** - 每个进程获得公平的 CPU 时间
2. **透明切换** - 进程不感知自己被切换
3. **显示与执行分离** - 用户界面和后台执行独立
