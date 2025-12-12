# 常见 Bug 案例集

## 概述

本文档收集 ECE 391 MP3 实验中常见的 Bug 及其解决方案。每个案例包含：
- 症状描述
- 根本原因
- 调试过程
- 修复方法
- 预防措施

---

## Checkpoint 1: 基础设施

### Bug 1.1: 三重故障重启

**症状**：
- QEMU 不断重启
- 或 QEMU 直接退出
- 看不到任何输出

**可能原因**：

1. **IDT 未正确加载**
```c
// 错误：忘记调用 lidt
void init_idt() {
    // 设置 IDT 入口...
    // 忘记了 lidt(&idt_desc_ptr);
}

// 正确：
void init_idt() {
    // 设置 IDT 入口...
    lidt(idt_desc_ptr);  // 加载 IDT
}
```

2. **IDT 入口地址错误**
```c
// 错误：使用函数地址而不是汇编入口
SET_IDT_ENTRY(idt[0x21], keyboard_handler);  // C 函数

// 正确：使用汇编包装器
SET_IDT_ENTRY(idt[0x21], keyboard_wrapper);  // 汇编入口
```

3. **分页配置错误**
```c
// 错误：内核页未设置 Present 位
page_directory[0] = 0x00000082;  // 缺少 Present 位

// 正确：
page_directory[0] = 0x00000083;  // Present | R/W | 4MB
```

4. **GDT/TSS 错误**
```c
// 错误：TSS 描述符设置错误
// 正确：确保 TSS 描述符的 type = 0x89 (32-bit TSS available)
```

**调试方法**：
```bash
# 启用 QEMU 调试输出
qemu-system-i386 -d int,cpu_reset -no-reboot ...
```

**预防措施**：
- 在 kernel.c 中逐步添加初始化代码，每步加 printf 确认
- 使用 GDB 在关键位置设断点

---

### Bug 1.2: 页错误死循环

**症状**：
- 触发页错误后系统挂起
- 或不断触发页错误

**原因**：页错误处理函数本身触发了页错误

```c
// 错误：页错误处理函数访问未映射地址
void page_fault_handler() {
    printf(...);  // printf 可能访问未映射的显存
}
```

**修复**：
```c
// 确保页错误处理中使用的所有地址都已映射
void page_fault_handler() {
    // 使用直接显存写入而不是 printf
    char* video = (char*)0xB8000;
    video[0] = 'P';
    video[1] = 0x0F;
    while(1);  // 停止
}
```

---

### Bug 1.3: 中断处理后系统挂起

**症状**：
- 第一次键盘按键后系统无响应
- RTC 中断只触发一次

**原因**：忘记发送 EOI

```c
// 错误：忘记 send_eoi
void keyboard_handler() {
    uint8_t scancode = inb(0x60);
    // 处理按键...
    // 忘记 send_eoi(1);
}

// 正确：
void keyboard_handler() {
    uint8_t scancode = inb(0x60);
    // 处理按键...
    send_eoi(1);  // IRQ1 是键盘
}
```

**对于 Slave PIC 的 IRQ (8-15)**：
```c
// 错误：只发送一次 EOI
void rtc_handler() {
    // ...
    send_eoi(8);  // 这只发给 slave
}

// 正确：send_eoi 应该处理级联
void send_eoi(uint32_t irq_num) {
    if (irq_num >= 8) {
        outb(EOI | (irq_num - 8), SLAVE_8259_PORT);
        outb(EOI | 2, MASTER_8259_PORT);  // IRQ2 是级联
    } else {
        outb(EOI | irq_num, MASTER_8259_PORT);
    }
}
```

---

### Bug 1.4: PIC 掩码设置错误

**症状**：
- 某些中断不触发
- 中断优先级异常

**原因**：enable_irq/disable_irq 逻辑错误

```c
// 错误：启用和禁用逻辑反了
void enable_irq(uint32_t irq_num) {
    uint8_t mask = inb(MASTER_DATA);
    mask |= (1 << irq_num);  // 这是禁用！
    outb(mask, MASTER_DATA);
}

// 正确：
void enable_irq(uint32_t irq_num) {
    uint8_t mask = inb(MASTER_DATA);
    mask &= ~(1 << irq_num);  // 清除位=启用
    outb(mask, MASTER_DATA);
}
```

---

## Checkpoint 2: 设备驱动

### Bug 2.1: 键盘按键重复或丢失

**症状**：
- 按一次键显示多个字符
- 某些按键不响应

**原因1**：未处理 release 扫描码

```c
// 错误：没有检查 release
void keyboard_handler() {
    uint8_t scancode = inb(0x60);
    char c = scancode_to_char[scancode];
    putc(c);  // release 时也会输出
}

// 正确：
void keyboard_handler() {
    uint8_t scancode = inb(0x60);
    if (scancode & 0x80) {
        // Release 事件，忽略（或处理 Shift/Ctrl 释放）
        send_eoi(1);
        return;
    }
    // Press 事件
    char c = scancode_to_char[scancode];
    putc(c);
    send_eoi(1);
}
```

**原因2**：Shift/Caps Lock 状态未正确维护

```c
// 需要维护状态
static int shift_pressed = 0;
static int caps_lock = 0;

void keyboard_handler() {
    uint8_t scancode = inb(0x60);

    // 更新修饰键状态
    if (scancode == 0x2A || scancode == 0x36) {  // Shift press
        shift_pressed = 1;
    } else if (scancode == 0xAA || scancode == 0xB6) {  // Shift release
        shift_pressed = 0;
    } else if (scancode == 0x3A) {  // Caps Lock press
        caps_lock = !caps_lock;
    }
    // ...
}
```

---

### Bug 2.2: terminal_read 不返回

**症状**：
- 调用 read(stdin) 后程序挂起
- 按 Enter 不返回

**原因**：等待条件错误或没有清除缓冲区

```c
// 错误：等待条件永远不满足
int terminal_read(...) {
    while (enter_pressed == 0);  // 永远不会变成 1
    // ...
}

// 正确：使用 volatile 并在键盘处理中设置
volatile int enter_pressed = 0;

void keyboard_handler() {
    if (scancode == 0x1C) {  // Enter
        enter_pressed = 1;
    }
}

int terminal_read(...) {
    while (!enter_pressed);
    enter_pressed = 0;  // 清除标志
    // 复制缓冲区...
}
```

---

### Bug 2.3: RTC 频率设置无效

**症状**：
- rtc_write 后频率没变化
- 或设置任意频率都一样

**原因**：频率到分频值的转换错误

```c
// 错误：直接写入频率值
void rtc_write(int freq) {
    outb(0x8A, 0x70);
    outb(freq, 0x71);  // 错误！应该是 rate
}

// 正确：转换为 rate
// frequency = 32768 >> (rate - 1)
// rate = log2(32768/frequency) + 1
void rtc_write(int freq) {
    int rate;
    switch (freq) {
        case 2:    rate = 15; break;
        case 4:    rate = 14; break;
        case 8:    rate = 13; break;
        case 16:   rate = 12; break;
        // ...
        case 1024: rate = 6;  break;
        default: return -1;
    }

    cli();
    outb(0x8A, 0x70);
    uint8_t prev = inb(0x71);
    outb(0x8A, 0x70);
    outb((prev & 0xF0) | rate, 0x71);
    sti();
}
```

---

### Bug 2.4: 文件系统读取错误

**症状**：
- 文件内容错误或乱码
- 读取大文件时崩溃

**原因1**：块号计算错误

```c
// 错误：忘记跳过直接块
int read_data(uint32_t inode, uint32_t offset, uint8_t* buf, uint32_t length) {
    uint32_t block_idx = offset / 4096;  // 这是相对文件的块索引
    uint32_t block_num = inode_ptr->data_blocks[block_idx];
    // 读取...
}

// 正确：检查边界
int read_data(uint32_t inode, uint32_t offset, uint8_t* buf, uint32_t length) {
    inode_t* inode_ptr = get_inode(inode);

    // 检查 offset 是否超出文件大小
    if (offset >= inode_ptr->length) {
        return 0;
    }

    // 调整读取长度
    if (offset + length > inode_ptr->length) {
        length = inode_ptr->length - offset;
    }

    uint32_t bytes_read = 0;
    while (bytes_read < length) {
        uint32_t block_idx = (offset + bytes_read) / 4096;
        uint32_t block_offset = (offset + bytes_read) % 4096;
        uint32_t block_num = inode_ptr->data_blocks[block_idx];

        // 读取块...
    }
}
```

**原因2**：数据块地址计算错误

```c
// 错误：没有正确定位数据块起始地址
uint8_t* block_addr = (uint8_t*)(fs_start + block_num * 4096);

// 正确：数据块在 inode 块之后
uint8_t* block_addr = (uint8_t*)(data_blocks_start + block_num * 4096);
// 其中 data_blocks_start = fs_start + (1 + num_inodes) * 4096
```

---

## Checkpoint 3: 系统调用与进程

### Bug 3.1: execute 后立即崩溃

**症状**：
- 执行用户程序后三重故障
- 或回到 shell 没有任何输出

**原因1**：用户栈设置错误

```c
// 错误：用户栈指向错误地址
uint32_t user_esp = 0x08000000;  // 这是程序起始，不是栈！

// 正确：用户栈在 132MB - 4
uint32_t user_esp = 0x08400000 - 4;  // 132MB - 4
```

**原因2**：IRET 栈帧错误

```c
// 错误：压栈顺序错误
push USER_CS
push user_eip
push eflags
push USER_DS
push user_esp

// 正确顺序（从高地址到低地址）：
// [高地址]
// SS
// ESP
// EFLAGS
// CS
// EIP
// [低地址] <- 当前 ESP

push USER_DS      // SS
push user_esp     // ESP
push eflags       // EFLAGS (记得设置 IF 位！)
push USER_CS      // CS
push user_eip     // EIP
iret
```

**原因3**：EFLAGS 没有设置 IF 位

```c
// 错误：EFLAGS = 0，中断被禁用
push 0x00000000

// 正确：设置 IF 位启用中断
push 0x00000200   // IF = 1
```

---

### Bug 3.2: halt 返回到错误位置

**症状**：
- 子进程 halt 后不返回父进程
- 返回后父进程行为异常

**原因**：父进程上下文未正确保存/恢复

```c
// 在 execute 中保存父进程上下文
int execute(...) {
    pcb_t* parent_pcb = get_current_pcb();
    pcb_t* child_pcb = get_specific_pcb(new_pid);

    // 保存父进程的 ESP 和 EBP
    asm volatile(
        "movl %%esp, %0\n"
        "movl %%ebp, %1\n"
        : "=r"(parent_pcb->saved_esp), "=r"(parent_pcb->saved_ebp)
    );

    // 设置子进程的父指针
    child_pcb->parent = parent_pcb;

    // ...
}

// 在 halt 中恢复
int halt(uint8_t status) {
    pcb_t* current_pcb = get_current_pcb();
    pcb_t* parent_pcb = current_pcb->parent;

    if (parent_pcb == NULL) {
        // 这是 shell，重新执行
        execute("shell");
    }

    // 恢复父进程的页表
    // 恢复 TSS
    // 恢复 ESP/EBP 并返回
    asm volatile(
        "movl %0, %%esp\n"
        "movl %1, %%ebp\n"
        "movl %2, %%eax\n"
        "leave\n"
        "ret\n"
        :
        : "r"(parent_pcb->saved_esp),
          "r"(parent_pcb->saved_ebp),
          "r"((uint32_t)status)
    );

    return 0;  // 不会执行到这里
}
```

---

### Bug 3.3: 文件描述符泄漏

**症状**：
- 多次 open 后 open 返回 -1
- 只能打开 6 个文件

**原因**：close 未正确清除 fd

```c
// 错误：close 没有标记 fd 为可用
int close(int fd) {
    // 只是返回 0
    return 0;
}

// 正确：
int close(int fd) {
    if (fd < 2 || fd > 7) return -1;

    pcb_t* pcb = get_current_pcb();

    // 检查 fd 是否在使用
    if (pcb->fd_array[fd].flags == 0) {
        return -1;  // fd 未打开
    }

    // 调用设备的 close
    if (pcb->fd_array[fd].ops->close != NULL) {
        pcb->fd_array[fd].ops->close(fd);
    }

    // 标记为可用
    pcb->fd_array[fd].flags = 0;

    return 0;
}
```

---

### Bug 3.4: getargs 返回乱码

**症状**：
- 程序参数是乱码
- 或参数截断

**原因**：参数未正确保存到 PCB

```c
// 错误：使用指针而不是复制
int execute(const char* command) {
    // 解析命令和参数
    char* args = strchr(command, ' ');

    pcb->args = args;  // 错误：command 在栈上，会被覆盖
}

// 正确：复制参数
int execute(const char* command) {
    char* args = strchr(command, ' ');
    if (args != NULL) {
        args++;  // 跳过空格
        strncpy(pcb->args_buf, args, MAX_ARGS_LEN);
    } else {
        pcb->args_buf[0] = '\0';
    }
}
```

---

## Checkpoint 4: 多终端

### Bug 4.1: 终端切换后显示错乱

**症状**：
- 切换回来后屏幕内容混乱
- 光标位置错误

**原因**：视频内存备份/恢复不完整

```c
// 错误：只保存部分显存
void switch_terminal(int new_term) {
    memcpy(term[curr_term].vid_backup, (void*)0xB8000, 2000);
    // 错误！应该是 4000 字节 (80*25*2)
}

// 正确：
void switch_terminal(int new_term) {
    // 保存当前终端显存 (4000 字节)
    memcpy(term[curr_term].vid_backup, (void*)0xB8000, 4000);

    // 保存当前光标
    term[curr_term].cursor_x = cursor_x;
    term[curr_term].cursor_y = cursor_y;

    // 恢复新终端显存
    memcpy((void*)0xB8000, term[new_term].vid_backup, 4000);

    // 恢复光标
    cursor_x = term[new_term].cursor_x;
    cursor_y = term[new_term].cursor_y;
    update_cursor();

    curr_term = new_term;
}
```

---

### Bug 4.2: 键盘输入到错误终端

**症状**：
- 在终端 1 输入，字符出现在终端 0
- 输入被多个终端接收

**原因**：键盘缓冲区没有按终端隔离

```c
// 错误：全局键盘缓冲区
char keyboard_buffer[128];

// 正确：每个终端独立缓冲区
typedef struct {
    char keyboard_buffer[128];
    int buffer_index;
    // ...
} term_info_t;

term_info_t term[3];

void keyboard_handler() {
    // 写入当前显示终端的缓冲区
    term[curr_term].keyboard_buffer[term[curr_term].buffer_index++] = c;
}
```

---

### Bug 4.3: 后台终端输出到前台

**症状**：
- 后台进程的 printf 出现在当前屏幕
- 多个终端输出混在一起

**原因**：putc 直接写入显存而不检查终端

```c
// 错误：直接写入 0xB8000
void putc(char c) {
    char* video = (char*)0xB8000;
    video[offset] = c;
}

// 正确：根据进程所属终端决定写入位置
void putc(char c) {
    pcb_t* pcb = get_current_pcb();
    int term_id = pcb->term_id;

    char* video;
    if (term_id == curr_term) {
        // 当前显示终端，写入真实显存
        video = (char*)0xB8000;
    } else {
        // 后台终端，写入备份缓冲区
        video = term[term_id].vid_backup;
    }

    video[offset] = c;
}
```

---

## Checkpoint 5: 调度

### Bug 5.1: 调度导致系统挂起

**症状**：
- PIT 中断后系统停止响应
- 或第一次调度后崩溃

**原因1**：上下文切换栈设置错误

```c
// 错误：保存/恢复 ESP 不匹配
void schedule() {
    // 保存当前进程
    asm volatile("movl %%esp, %0" : "=r"(current->saved_esp));

    // 切换到下一进程
    next = get_next_process();

    // 恢复下一进程
    asm volatile("movl %0, %%esp" : : "r"(next->saved_esp));
    // 缺少 EBP 恢复！
}

// 正确：保存和恢复都要完整
void schedule() {
    asm volatile(
        "movl %%esp, %0\n"
        "movl %%ebp, %1\n"
        : "=r"(current->saved_esp), "=r"(current->saved_ebp)
    );

    next = get_next_process();

    asm volatile(
        "movl %0, %%esp\n"
        "movl %1, %%ebp\n"
        :
        : "r"(next->saved_esp), "r"(next->saved_ebp)
    );
}
```

**原因2**：TSS.esp0 未更新

```c
void schedule() {
    // 切换进程后必须更新 TSS
    tss.esp0 = 0x800000 - next_pid * 0x2000 - 4;
}
```

---

### Bug 5.2: PIT 频率异常

**症状**：
- 调度太快或太慢
- 系统时钟不准

**原因**：PIT 分频值计算错误

```c
// 错误：直接使用频率
outb(100, PIT_DATA);  // 错误！

// 正确：计算分频值
// divisor = 1193180 / desired_frequency
#define PIT_FREQUENCY 1193180
#define SCHEDULE_FREQ 100  // 100 Hz

uint16_t divisor = PIT_FREQUENCY / SCHEDULE_FREQ;  // 11932
outb(divisor & 0xFF, PIT_DATA);         // 低字节
outb((divisor >> 8) & 0xFF, PIT_DATA);  // 高字节
```

---

### Bug 5.3: 调度时键盘无响应

**症状**：
- 调度启动后键盘不工作
- 但 Ctrl+Alt+Del 有效

**原因**：PIT 中断处理时间太长，阻塞其他中断

```c
// 错误：在中断处理中做太多事情
void pit_handler() {
    // 大量计算...
    schedule();  // 可能很慢
    send_eoi(0);
}

// 正确：尽快发送 EOI，减少中断禁用时间
void pit_handler() {
    send_eoi(0);  // 先发送 EOI
    schedule();   // 然后调度
}
```

---

## 通用 Bug

### Bug G.1: 栈溢出

**症状**：
- 随机崩溃
- 数据结构被覆盖

**检测方法**：
```c
// 在栈底放置金丝雀值
#define STACK_CANARY 0xDEADBEEF

void init_process(int pid) {
    uint32_t* stack_bottom = (uint32_t*)(0x800000 - (pid + 1) * 0x2000);
    *stack_bottom = STACK_CANARY;
}

void check_stack(int pid) {
    uint32_t* stack_bottom = (uint32_t*)(0x800000 - (pid + 1) * 0x2000);
    if (*stack_bottom != STACK_CANARY) {
        printf("Stack overflow for pid %d!\n", pid);
    }
}
```

---

### Bug G.2: 竞态条件

**症状**：
- 问题间歇性出现
- 添加 printf 后问题消失

**常见场景**：
```c
// 错误：中断和主代码同时访问
volatile int flag = 0;

void main_code() {
    if (flag == 0) {
        // 中断可能在这里修改 flag
        flag = 1;  // 竞态！
    }
}

// 正确：使用 cli/sti 保护
void main_code() {
    cli();
    if (flag == 0) {
        flag = 1;
    }
    sti();
}
```

---

### Bug G.3: 空指针解引用

**症状**：
- 页错误，fault address = 0x00000000 附近

**预防**：
```c
// 添加空指针检查
void use_pcb(pcb_t* pcb) {
    if (pcb == NULL) {
        printf("NULL PCB!\n");
        return;
    }
    // ...
}
```

---

### Bug G.4: 整数溢出

**症状**：
- 计算结果异常
- 地址跳转到奇怪的位置

**示例**：
```c
// 错误：可能溢出
uint32_t addr = base + offset * size;  // 如果 offset * size 溢出...

// 正确：先检查
if (offset > MAX_OFFSET || size > MAX_SIZE) {
    return -1;
}
uint32_t addr = base + offset * size;
```

---

## 调试技巧总结

### 二分法定位

1. 注释掉一半代码
2. 如果问题消失，bug 在被注释的部分
3. 如果问题仍在，bug 在未注释的部分
4. 重复直到定位

### 添加检查点

```c
void suspicious_function() {
    printf("CHECKPOINT 1\n");
    // 代码块 1

    printf("CHECKPOINT 2\n");
    // 代码块 2

    printf("CHECKPOINT 3\n");
    // 代码块 3
}
```

### 打印关键变量

```c
void debug_context() {
    printf("cur_pid = %d\n", cur_pid);
    printf("ESP = 0x%x\n", get_esp());
    printf("EBP = 0x%x\n", get_ebp());

    pcb_t* pcb = get_current_pcb();
    printf("PCB at 0x%x\n", pcb);
    printf("PCB->parent = 0x%x\n", pcb->parent);
}
```

### 使用断言

```c
#define ASSERT(condition) do { \
    if (!(condition)) { \
        printf("ASSERT FAILED: %s at %s:%d\n", \
               #condition, __FILE__, __LINE__); \
        while(1); \
    } \
} while(0)

// 使用
void some_function(int fd) {
    ASSERT(fd >= 0 && fd < 8);
    // ...
}
```

---

## 总结

| 类型 | 常见原因 | 预防方法 |
|-----|---------|---------|
| 三重故障 | IDT/GDT/分页错误 | 逐步初始化，每步验证 |
| 页错误 | 未映射/权限错误 | 检查 CR2，验证页表 |
| 中断问题 | EOI/掩码/IF 标志 | 检查 PIC 状态 |
| 进程切换 | TSS/栈/上下文错误 | 打印切换前后状态 |
| 竞态条件 | 共享数据未保护 | 使用 cli/sti |
| 内存错误 | 溢出/越界/空指针 | 使用金丝雀/断言 |

调试心态：
1. **不要猜测** - 用数据说话
2. **最小复现** - 找到触发 bug 的最简步骤
3. **一次改一处** - 不要同时修改多处
4. **记录尝试** - 避免重复尝试失败的方法
5. **休息一下** - 疲劳时容易忽略明显问题
