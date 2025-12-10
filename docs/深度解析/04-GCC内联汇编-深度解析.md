# GCC 内联汇编 (Inline Assembly) 深度解析

## 1. 什么是内联汇编？

内联汇编允许在C代码中直接嵌入汇编指令，用于：

- 访问CPU特殊寄存器（如CR0, CR3, CR4）
- 执行特权指令（如 `lidt`, `lgdt`, `cli`, `sti`）
- 性能关键代码优化
- 硬件操作

---

## 2. 基本语法

### 2.1 最简形式

```c
asm("汇编指令");
```

### 2.2 完整形式

```c
asm volatile (
    "汇编指令"
    : 输出操作数
    : 输入操作数
    : 破坏描述
);
```

### 2.3 语法结构图

```
asm volatile (
    "指令1;"
    "指令2;"
    "指令3"
    : "约束"(输出变量), "约束"(输出变量)    ← 输出操作数
    : "约束"(输入变量), "约束"(输入变量)    ← 输入操作数
    : "寄存器", "寄存器", "memory", "cc"   ← 破坏描述
);

四个部分用冒号 : 分隔
每个部分内多个项目用逗号 , 分隔
```

---

## 3. 关键字解析

### 3.1 asm和__asm__

```c
asm("...");      // 标准写法
__asm__("...");  // 兼容写法（避免与变量名冲突）
```

两者完全等价，`__asm__`用于避免命名冲突。

### 3.2 volatile和__volatile__

```c
asm volatile ("...");      // 标准写法
__asm__ __volatile__("..."); // 兼容写法
```

**volatile的作用：禁止编译器优化这段代码**

```
┌─────────────────────────────────────────────────────────────────┐
│  没有volatile：                                                  │
│    编译器可能：                                                   │
│    - 删除"看起来无用"的代码                                        │
│    - 重排指令顺序                                                 │
│    - 合并多次相同调用                                              │
│                                                                 │
│  有volatile：                                                    │
│    编译器必须：                                                   │
│    - 保留这段代码                                                 │
│    - 保持指令顺序                                                 │
│    - 每次调用都执行                                               │
└─────────────────────────────────────────────────────────────────┘
```

**何时需要volatile？**

| 场景              | 是否需要 volatile |
|-----------------|---------------|
| 访问硬件寄存器         | ✓ 需要          |
| 有副作用的指令（如刷新TLB） | ✓ 需要          |
| 纯计算（有输出变量）      | 通常不需要         |

---

## 4. 汇编语法：AT&T vs Intel

GCC内联汇编默认使用 **AT&T语法**：

```
┌─────────────────────────────────────────────────────────────────┐
│  AT&T 语法（GCC 默认）           Intel 语法（NASM）                │
├─────────────────────────────────────────────────────────────────┤
│  mov  源, 目的                   mov  目的, 源                    │
│  mov  %eax, %ebx                mov  ebx, eax                   │
│  (eax → ebx)                    (eax → ebx)                     │
├─────────────────────────────────────────────────────────────────┤
│  寄存器前加 %                     寄存器不加前缀                    │
│  %eax, %cr3                     eax, cr3                        │
├─────────────────────────────────────────────────────────────────┤
│  立即数前加 $                     立即数不加前缀                    │
│  mov $0x10, %eax                mov eax, 0x10                   │
├─────────────────────────────────────────────────────────────────┤
│  内存操作数用 ()                  内存操作数用 []                   │
│  mov (%ebx), %eax               mov eax, [ebx]                  │
└─────────────────────────────────────────────────────────────────┘
```

**常用指令对比：**

| 操作    | AT&T 语法             | Intel 语法           |
|-------|---------------------|--------------------|
| 寄存器传送 | `mov %eax, %ebx`    | `mov ebx, eax`     |
| 立即数加载 | `mov $100, %eax`    | `mov eax, 100`     |
| 内存读取  | `mov (%ebx), %eax`  | `mov eax, [ebx]`   |
| 带偏移   | `mov 4(%ebx), %eax` | `mov eax, [ebx+4]` |

---

## 5. 寄存器命名：为什么用%%？

```
┌───────────────────────────────────────────────────────────────┐
│  在GCC内联汇编中：                                              │
│                                                               │
│  %0, %1, %2 ... = 引用输出/输入操作数（占位符）                    │
│  %%eax         = 真正的eax寄存器                                │
│                                                               │
│  为了区分，寄存器名需要用%%转义                                    │
└───────────────────────────────────────────────────────────────┘
```

**对比：**

| 场景            | 写法      | 含义     |
|---------------|---------|--------|
| 普通汇编文件(.S)    | `%eax`  | eax寄存器 |
| GCC内联汇编（无操作数） | `%%eax` | eax寄存器 |
| GCC内联汇编（有操作数） | `%0`    | 第一个操作数 |

**示例：**

```c
// 没有输入输出操作数，用%%引用寄存器
asm volatile (
    "mov %%cr3, %%eax;"
    "mov %%eax, %%cr3;"
    :::"%eax"
);

// 有输入输出操作数，用%0, %1引用
uint32_t value;
asm volatile (
    "mov %%cr2, %0"      // %0 = value
    : "=r"(value)
);
```

---

## 6. 输出操作数

### 6.1 基本格式

```c
: "约束"(C变量)
```

### 6.2 输出约束必须以=或+开头

| 前缀  | 含义 | 说明            |
|-----|----|---------------|
| `=` | 只写 | 汇编代码只写入，不读取原值 |
| `+` | 读写 | 汇编代码先读后写      |

### 6.3 示例

```c
// 只写输出
uint32_t result;
asm volatile (
    "mov %%cr0, %0"
    : "=r"(result)    // result = cr0（只写）
);

// 读写输出
uint32_t value = 10;
asm volatile (
    "add $5, %0"
    : "+r"(value)     // value = value + 5（先读后写）
);
```

---

## 7. 输入操作数

### 7.1 基本格式

```c
: "约束"(C变量或常量)
```

### 7.2 示例

```c
uint32_t new_cr3 = 0x100000;
asm volatile (
    "mov %0, %%cr3"
    :                     // 无输出
    : "r"(new_cr3)        // 输入：new_cr3
);
```

---

## 8. 约束字符详解

### 8.1 寄存器约束

| 约束  | 含义        | 说明      |
|-----|-----------|---------|
| `r` | 任意通用寄存器   | 编译器自动选择 |
| `a` | eax/ax/al |         |
| `b` | ebx/bx/bl |         |
| `c` | ecx/cx/cl |         |
| `d` | edx/dx/dl |         |
| `S` | esi/si    |         |
| `D` | edi/di    |         |

### 8.2 内存约束

| 约束  | 含义        |
|-----|-----------|
| `m` | 内存操作数     |
| `o` | 可偏移的内存操作数 |

### 8.3 立即数约束

| 约束  | 含义          |
|-----|-------------|
| `i` | 立即数（整数）     |
| `n` | 已知值的立即数     |
| `I` | 0-31 范围的立即数 |
| `J` | 0-63 范围的立即数 |

### 8.4 组合约束

```c
"rm"   // 寄存器或内存
"ri"   // 寄存器或立即数
"rmi"  // 寄存器、内存或立即数
```

### 8.5 约束修饰符

| 修饰符 | 位置 | 含义                 |
|-----|----|--------------------|
| `=` | 输出 | 只写                 |
| `+` | 输出 | 读写                 |
| `&` | 输出 | 早期破坏（earlyclobber） |

---

## 9. 破坏描述 (Clobber List)

### 9.1 作用

告诉编译器：这段汇编代码修改了哪些寄存器/内存，编译器需要保护这些值。

### 9.2 常用破坏描述

| 描述         | 含义                     |
|------------|------------------------|
| `"%eax"`   | eax 寄存器被修改             |
| `"%ebx"`   | ebx 寄存器被修改             |
| `"memory"` | 内存被修改（编译器不能缓存内存值）      |
| `"cc"`     | 条件码/标志寄存器 (EFLAGS) 被修改 |

### 9.3 示例

```c
asm volatile (
    "mov %%cr3, %%eax;"
    "mov %%eax, %%cr3;"
    :
    :
    : "%eax"              // 告诉编译器eax被修改了
);

asm volatile (
    "cli"                 // 关中断
    :
    :
    : "memory", "cc"      // 可能影响内存和标志位
);
```

---

## 10. 操作数引用

### 10.1 数字引用

```c
asm ("add %1, %0"
    : "=r"(result)    // %0 = result
    : "r"(value)      // %1 = value
);
```

### 10.2 符号引用（GCC 扩展）

```c
asm ("add %[val], %[res]"
    : [res] "=r"(result)    // %[res] = result
    : [val] "r"(value)      // %[val] = value
);
```

符号引用更易读，推荐使用。

---

## 11. 完整示例

### 11.1 读取控制寄存器

```c
// 读取CR0
uint32_t get_cr0(void) {
    uint32_t cr0;
    asm volatile (
        "mov %%cr0, %0"
        : "=r"(cr0)       // 输出：cr0
        :                 // 无输入
        :                 // 无破坏
    );
    return cr0;
}

// 读取CR2（页错误地址）
uint32_t get_cr2(void) {
    uint32_t cr2;
    asm volatile (
        "mov %%cr2, %0"
        : "=r"(cr2)
    );
    return cr2;
}

// 读取CR3（页目录基址）
uint32_t get_cr3(void) {
    uint32_t cr3;
    asm volatile (
        "mov %%cr3, %0"
        : "=r"(cr3)
    );
    return cr3;
}
```

### 11.2 写入控制寄存器

```c
// 写入CR0
void set_cr0(uint32_t cr0) {
    asm volatile (
        "mov %0, %%cr0"
        :                 // 无输出
        : "r"(cr0)        // 输入：cr0
    );
}

// 写入CR3（切换页目录）
void set_cr3(uint32_t cr3) {
    asm volatile (
        "mov %0, %%cr3"
        :
        : "r"(cr3)
    );
}
```

### 11.3 刷新TLB

```c
// 方法1：读写CR3
void flush_TLB(void) {
    asm volatile (
        "mov %%cr3, %%eax;"
        "mov %%eax, %%cr3;"
        :
        :
        : "%eax"
    );
}

// 方法2：使用INVLPG刷新单个页
void flush_TLB_single(uint32_t addr) {
    asm volatile (
        "invlpg (%0)"
        :
        : "r"(addr)
        : "memory"
    );
}
```

### 11.4 开关中断

```c
// 关中断
void cli(void) {
    asm volatile ("cli" ::: "memory");
}

// 开中断
void sti(void) {
    asm volatile ("sti" ::: "memory");
}

// 保存标志并关中断
uint32_t save_flags_and_cli(void) {
    uint32_t flags;
    asm volatile (
        "pushfl;"
        "popl %0;"
        "cli"
        : "=r"(flags)
        :
        : "memory"
    );
    return flags;
}

// 恢复标志
void restore_flags(uint32_t flags) {
    asm volatile (
        "pushl %0;"
        "popfl"
        :
        : "r"(flags)
        : "memory", "cc"
    );
}
```

### 11.5 加载IDT/GDT

```c
// 加载 IDT
void lidt(void* base, uint16_t limit) {
    struct {
        uint16_t limit;
        uint32_t base;
    } __attribute__((packed)) idtr = { limit, (uint32_t)base };

    asm volatile (
        "lidt %0"
        :
        : "m"(idtr)
    );
}

// 加载 GDT
void lgdt(void* base, uint16_t limit) {
    struct {
        uint16_t limit;
        uint32_t base;
    } __attribute__((packed)) gdtr = { limit, (uint32_t)base };

    asm volatile (
        "lgdt %0"
        :
        : "m"(gdtr)
    );
}
```

### 11.6 端口I/O

```c
// 从端口读取一个字节
uint8_t inb(uint16_t port) {
    uint8_t data;
    asm volatile (
        "inb %1, %0"
        : "=a"(data)      // 输出到 al
        : "d"(port)       // 端口号在 dx
    );
    return data;
}

// 向端口写入一个字节
void outb(uint16_t port, uint8_t data) {
    asm volatile (
        "outb %0, %1"
        :
        : "a"(data),      // 数据在 al
          "d"(port)       // 端口号在 dx
    );
}

// 读取一个字（16位）
uint16_t inw(uint16_t port) {
    uint16_t data;
    asm volatile (
        "inw %1, %0"
        : "=a"(data)
        : "d"(port)
    );
    return data;
}

// 写入一个字（16位）
void outw(uint16_t port, uint16_t data) {
    asm volatile (
        "outw %0, %1"
        :
        : "a"(data),
          "d"(port)
    );
}

// 读取双字（32位）
uint32_t inl(uint16_t port) {
    uint32_t data;
    asm volatile (
        "inl %1, %0"
        : "=a"(data)
        : "d"(port)
    );
    return data;
}

// 写入双字（32位）
void outl(uint16_t port, uint32_t data) {
    asm volatile (
        "outl %0, %1"
        :
        : "a"(data),
          "d"(port)
    );
}
```

### 11.7 原子操作

```c
// 原子交换
uint32_t xchg(volatile uint32_t* addr, uint32_t newval) {
    uint32_t result;
    asm volatile (
        "lock; xchgl %0, %1"
        : "+m"(*addr), "=a"(result)
        : "1"(newval)
        : "cc"
    );
    return result;
}

// 原子比较并交换
int cmpxchg(volatile uint32_t* addr, uint32_t oldval, uint32_t newval) {
    uint32_t result;
    asm volatile (
        "lock; cmpxchgl %2, %1"
        : "=a"(result), "+m"(*addr)
        : "r"(newval), "0"(oldval)
        : "cc"
    );
    return result == oldval;
}
```

---

## 12. 多行汇编

### 12.1 字符串连接

```c
asm volatile (
    "指令1;"
    "指令2;"
    "指令3"
    : ...
);
```

C 编译器会自动连接相邻的字符串字面量。

### 12.2 使用 \n\t 提高可读性

```c
asm volatile (
    "mov %%cr3, %%eax\n\t"
    "mov %%eax, %%cr3\n\t"
    :
    :
    : "%eax"
);
```

`\n\t` 会让生成的汇编代码更易读（换行+缩进）。

---

## 13. 常见错误

### 13.1 忘记%%转义

```c
// 错误！
asm ("mov %cr3, %eax");    // %cr3 会被解释为操作数

// 正确
asm ("mov %%cr3, %%eax");
```

### 13.2 输出约束缺少=或+

```c
// 错误！
asm ("mov %%cr0, %0" : "r"(result));

// 正确
asm ("mov %%cr0, %0" : "=r"(result));
```

### 13.3 忘记声明破坏的寄存器

```c
// 错误！编译器不知道 eax 被修改
asm volatile (
    "mov %%cr3, %%eax;"
    "mov %%eax, %%cr3;"
);

// 正确
asm volatile (
    "mov %%cr3, %%eax;"
    "mov %%eax, %%cr3;"
    ::: "%eax"
);
```

### 13.4 输入输出使用同一变量但约束不对

```c
// 如果想读写同一个变量
uint32_t x = 10;

// 错误！
asm ("add $5, %0" : "=r"(x) : "r"(x));  // 输入输出可能分配不同寄存器

// 正确
asm ("add $5, %0" : "+r"(x));           // 使用 + 约束
```

---

## 14. 调试技巧

### 14.1 查看生成的汇编

```bash
gcc -S -o output.s input.c    # 生成汇编文件
gcc -save-temps input.c       # 保留所有中间文件
```

### 14.2 在汇编中添加注释

```c
asm volatile (
    "# 这是注释\n\t"
    "mov %%cr3, %%eax\n\t"
    "# 写回 cr3 触发 TLB 刷新\n\t"
    "mov %%eax, %%cr3"
    ::: "%eax"
);
```

---

## 15. 总结速查表

### 15.1 语法模板

```c
asm volatile (
    "汇编指令"
    : "=约束"(输出变量)     // 输出
    : "约束"(输入变量)      // 输入
    : "破坏列表"           // 破坏
);
```

### 15.2 常用约束

| 约束        | 含义              |
|-----------|-----------------|
| `r`       | 任意寄存器           |
| `a/b/c/d` | eax/ebx/ecx/edx |
| `m`       | 内存              |
| `i`       | 立即数             |
| `=`       | 只写输出            |
| `+`       | 读写输出            |

### 15.3 常用破坏描述

| 描述         | 含义       |
|------------|----------|
| `"%eax"`   | eax 被修改  |
| `"memory"` | 内存被修改    |
| `"cc"`     | 标志寄存器被修改 |

### 15.4 AT&T vs Intel

| AT&T               | Intel            | 操作          |
|--------------------|------------------|-------------|
| `mov %eax, %ebx`   | `mov ebx, eax`   | eax → ebx   |
| `mov $10, %eax`    | `mov eax, 10`    | 10 → eax    |
| `mov (%ebx), %eax` | `mov eax, [ebx]` | [ebx] → eax |

---

**文档版本**：1.0
**最后更新**：2025-12-10
**相关文件**：

- `student-distrib/paging.c` - flush_TLB() 实现
- `student-distrib/x86_desc.S` - GDT/IDT 相关汇编
