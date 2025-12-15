# Linux 内核/驱动开发学习路线图

> 目标岗位：Linux C/C++ 内核开发、驱动开发工程师

---

## 一、当前基础评估（ECE 391 项目）

通过这个项目，你已经掌握：

| 模块 | 掌握内容 | 掌握程度 |
|------|---------|---------|
| 架构基础 | x86 保护模式、GDT/TSS | ✓ 扎实 |
| 中断机制 | IDT、8259 PIC、异常处理 | ✓ 扎实 |
| 内存管理 | 分页原理、虚拟地址转换 | ✓ 基础 |
| 设备驱动 | 键盘、RTC、终端驱动 | ✓ 入门 |
| 文件系统 | 只读文件系统、inode | ✓ 基础 |
| 进程管理 | PCB、系统调用、用户态/内核态 | ✓ 扎实 |
| 进程调度 | Round-Robin 调度 | ✓ 基础 |

**结论**：原理基础良好，但与真实 Linux 内核开发还有差距。

---

## 二、知识差距分析

### 2.1 内存管理

| 已掌握 | 需要补充 |
|--------|---------|
| 简单分页 | 伙伴系统（Buddy System） |
| 单一地址空间 | Slab/Slub 分配器（kmalloc 原理） |
| - | vmalloc / ioremap |
| - | 页面回收与交换（kswapd, swap） |
| - | OOM Killer 机制 |
| - | NUMA 架构内存管理 |
| - | 内存屏障（mb, rmb, wmb） |
| - | 写时复制（COW） |

### 2.2 进程与调度

| 已掌握 | 需要补充 |
|--------|---------|
| 简单 Round-Robin | CFS 完全公平调度器 |
| 基础 PCB | task_struct 完整结构 |
| 简单上下文切换 | 实时调度（SCHED_FIFO, SCHED_RR） |
| - | cgroups（资源限制） |
| - | namespace（容器隔离） |
| - | 内核线程（kthread） |
| - | 进程状态机详解 |

### 2.3 同步机制（重点）

这是内核开发**最难也最重要**的部分，ECE 391 几乎没有涉及：

```
必须掌握：
├── spinlock（自旋锁）
│   ├── 原理：忙等待，不可睡眠
│   ├── 使用场景：中断上下文、临界区很短
│   └── 变体：spin_lock_irq, spin_lock_irqsave
│
├── mutex（互斥锁）
│   ├── 原理：可睡眠
│   ├── 使用场景：进程上下文、临界区可能较长
│   └── 注意：中断上下文不能用
│
├── semaphore（信号量）
│   └── 计数信号量，允许多个持有者
│
├── RCU（Read-Copy-Update）
│   ├── 原理：读不加锁，写时复制
│   ├── 使用场景：读多写少
│   └── 内核大量使用，必须掌握
│
├── 原子操作（atomic_t）
│   └── atomic_add, atomic_sub, atomic_inc
│
├── 完成量（completion）
│   └── 等待某个事件完成
│
├── 内存屏障（barrier）
│   └── mb(), rmb(), wmb(), smp_mb()
│
└── 死锁检测（lockdep）
    └── 开发时必开的调试选项
```

**面试高频题**：
- spinlock 和 mutex 的区别？什么时候用哪个？
- 中断上下文能用 mutex 吗？为什么？
- RCU 的原理是什么？适用场景？
- 如何避免死锁？

### 2.4 中断子系统

| 已掌握 | 需要补充 |
|--------|---------|
| 简单中断处理 | 上半部/下半部分离机制 |
| 8259 PIC | APIC（x86）/ GIC（ARM） |
| - | softirq（软中断） |
| - | tasklet |
| - | workqueue（工作队列） |
| - | threaded IRQ（线程化中断） |
| - | 中断亲和性（IRQ affinity） |

**上半部 vs 下半部**：
```
上半部（Top Half）：
├── 硬件中断处理函数
├── 要求：快速、不可睡眠
└── 只做必要的工作（读数据、清中断）

下半部（Bottom Half）：
├── softirq：最高优先级，网络/块设备用
├── tasklet：基于 softirq，简单易用
└── workqueue：可睡眠，最灵活
```

### 2.5 文件系统

| 已掌握 | 需要补充 |
|--------|---------|
| 只读简单 FS | VFS 虚拟文件系统层 |
| 单一文件系统 | ext4 / xfs / btrfs 原理 |
| - | Page Cache（页缓存） |
| - | inode / dentry 缓存 |
| - | 块层（Block Layer） |
| - | I/O 调度器（mq-deadline, bfq, kyber） |

### 2.6 驱动开发框架

```
Linux 驱动框架：
├── 字符设备驱动
│   ├── file_operations 结构体
│   ├── 设备号（主设备号/次设备号）
│   ├── cdev 注册
│   └── /dev 节点创建
│
├── 平台设备驱动（重点）
│   ├── platform_driver 结构体
│   ├── probe / remove 函数
│   └── 设备树匹配
│
├── 设备树（Device Tree）
│   ├── DTS 语法
│   ├── compatible 属性
│   ├── of_* API
│   └── 设备树覆盖（overlay）
│
├── 总线子系统
│   ├── I2C 驱动框架
│   ├── SPI 驱动框架
│   ├── UART 驱动框架
│   └── USB 驱动框架
│
├── 中断申请
│   ├── request_irq / request_threaded_irq
│   └── 中断处理函数编写
│
└── DMA 编程
    ├── 一致性 DMA（dma_alloc_coherent）
    └── 流式 DMA（dma_map_single）
```

### 2.7 网络子系统（可选）

```
如果做网络相关：
├── socket 层实现
├── TCP/IP 协议栈
├── sk_buff 结构
├── 网络设备驱动（NAPI）
├── Netfilter 框架
└── eBPF / XDP
```

### 2.8 调试与追踪工具

```
必须熟练：
├── printk / pr_info / pr_err / pr_debug
├── dynamic_debug（动态调试）
├── ftrace（函数追踪）
│   ├── function tracer
│   ├── function_graph tracer
│   └── 事件追踪
├── perf（性能分析）
│   ├── perf top / perf record / perf report
│   └── perf sched（调度分析）
├── kprobes / tracepoints
├── eBPF（bcc / bpftrace）
├── crash + kdump（崩溃分析）
└── KASAN / UBSAN / KMSAN（内存错误检测）
```

### 2.9 架构知识

```
x86 之外还要学：
├── ARM 架构（大量驱动岗位需要）
│   ├── ARM 异常模型
│   ├── GIC 中断控制器
│   └── ARM64 内存模型
├── RISC-V（新兴，可选）
└── SoC 基础知识
```

---

## 三、学习路线图

### 阶段一：Linux 内核基础（2-4 周）

**目标**：理解 Linux 内核整体架构

```
学习内容：
├── 内核源码结构
│   ├── arch/     - 架构相关代码
│   ├── kernel/   - 核心子系统（调度、信号等）
│   ├── mm/       - 内存管理
│   ├── fs/       - 文件系统
│   ├── drivers/  - 驱动程序
│   ├── net/      - 网络子系统
│   └── include/  - 头文件
│
├── 内核编译
│   ├── make menuconfig
│   ├── make -j$(nproc)
│   └── 内核模块编译
│
└── 内核启动流程
    ├── bootloader -> kernel
    ├── start_kernel()
    └── init 进程
```

**实践任务**：
1. 下载并编译 Linux 内核
2. 用 QEMU 运行自己编译的内核
3. 修改内核配置，观察效果

**参考资料**：
- 《Linux内核设计与实现》第1-3章
- https://kernelnewbies.org/KernelBuild

---

### 阶段二：内核模块开发（1-2 周）

**目标**：掌握内核模块编写和调试

```
学习内容：
├── 模块基本结构
│   ├── module_init / module_exit
│   ├── MODULE_LICENSE / MODULE_AUTHOR
│   └── 模块参数
│
├── 编译系统
│   ├── Kbuild / Makefile
│   └── 外部模块编译
│
└── 模块调试
    ├── printk 日志
    ├── /proc 和 /sys 接口
    └── dmesg 查看
```

**实践任务**：
1. 编写 Hello World 内核模块
2. 添加模块参数
3. 创建 /proc 文件接口

**代码示例**：
```c
// hello.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hello_init(void)
{
    pr_info("Hello, kernel!\n");
    return 0;
}

static void __exit hello_exit(void)
{
    pr_info("Goodbye, kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

---

### 阶段三：同步机制（2-3 周）

**目标**：深入理解内核同步原语

```
学习顺序：
1. 原子操作 -> 最简单
2. spinlock -> 中断上下文
3. mutex -> 进程上下文
4. RCU -> 高级技巧
```

**实践任务**：
1. 编写模块演示竞态条件
2. 用 spinlock 保护共享数据
3. 用 mutex 实现互斥访问
4. 阅读内核中 RCU 使用示例

**参考资料**：
- 《Linux内核设计与实现》第9-10章
- Documentation/locking/

---

### 阶段四：内存管理（2-3 周）

**目标**：理解 Linux 内存管理子系统

```
学习内容：
├── 物理内存管理
│   ├── 伙伴系统
│   └── /proc/buddyinfo
│
├── Slab 分配器
│   ├── kmalloc / kfree
│   ├── kmem_cache_create
│   └── /proc/slabinfo
│
├── 虚拟内存
│   ├── vmalloc / vfree
│   ├── ioremap
│   └── 页表操作
│
└── 进程地址空间
    ├── mm_struct
    ├── vm_area_struct
    └── mmap 实现
```

**实践任务**：
1. 编写模块使用 kmalloc/vmalloc
2. 创建自定义 slab 缓存
3. 分析 /proc/meminfo 各字段含义

**参考资料**：
- 《深入理解Linux内核》第8章
- 《深入Linux内核架构》第3章

---

### 阶段五：字符设备驱动（2-3 周）

**目标**：掌握字符设备驱动开发

```
学习内容：
├── 设备号
│   ├── 静态分配（register_chrdev_region）
│   └── 动态分配（alloc_chrdev_region）
│
├── cdev 结构
│   ├── cdev_init
│   ├── cdev_add
│   └── cdev_del
│
├── file_operations
│   ├── open / release
│   ├── read / write
│   ├── ioctl
│   ├── mmap
│   └── poll
│
└── 设备节点
    ├── mknod 手动创建
    └── class_create + device_create 自动创建
```

**实践任务**：
1. 实现一个虚拟字符设备
2. 支持 read/write 操作
3. 添加 ioctl 命令
4. 实现内核与用户空间数据交换

**参考资料**：
- 《Linux设备驱动程序》第3章
- drivers/char/ 目录示例

---

### 阶段六：中断与下半部（2 周）

**目标**：掌握中断处理机制

```
学习内容：
├── 中断申请
│   ├── request_irq
│   ├── request_threaded_irq
│   └── free_irq
│
├── 中断处理函数
│   ├── 上半部：快速、不可睡眠
│   └── 返回值：IRQ_HANDLED / IRQ_NONE
│
└── 下半部机制
    ├── tasklet_init / tasklet_schedule
    ├── INIT_WORK / schedule_work
    └── 选择：tasklet vs workqueue
```

**实践任务**：
1. 编写中断处理程序
2. 使用 tasklet 延迟处理
3. 使用 workqueue 延迟处理
4. 比较不同下半部机制的特点

---

### 阶段七：平台驱动与设备树（2-3 周）

**目标**：掌握现代 Linux 驱动框架

```
学习内容：
├── 平台设备模型
│   ├── platform_device
│   ├── platform_driver
│   └── probe / remove
│
├── 设备树
│   ├── DTS 语法
│   ├── compatible 匹配
│   ├── of_* API
│   └── 属性读取
│
└── 资源获取
    ├── platform_get_resource（内存、IRQ）
    ├── devm_* 系列函数
    └── 资源自动释放
```

**实践任务**：
1. 编写平台驱动框架
2. 编写设备树节点
3. 实现 GPIO LED 驱动
4. 实现 I2C 传感器驱动

**参考资料**：
- Documentation/devicetree/
- 《Linux设备驱动开发详解》

---

### 阶段八：调度器与进程（2 周）

**目标**：深入理解进程调度

```
学习内容：
├── task_struct 结构
├── 进程状态转换
├── CFS 调度器
│   ├── 虚拟运行时间
│   ├── 红黑树组织
│   └── 调度实体
├── 实时调度
└── cgroups CPU 控制
```

**实践任务**：
1. 阅读 kernel/sched/fair.c
2. 使用 perf sched 分析调度
3. 配置 cgroups CPU 限制

---

### 阶段九：文件系统（2 周）

**目标**：理解 VFS 和文件系统实现

```
学习内容：
├── VFS 核心结构
│   ├── super_block
│   ├── inode
│   ├── dentry
│   └── file
│
├── 文件操作
│   ├── file_operations
│   ├── inode_operations
│   └── address_space_operations
│
└── 页缓存
    ├── Page Cache 原理
    └── 读写流程
```

**实践任务**：
1. 实现一个简单的内存文件系统
2. 注册到 VFS
3. 支持基本文件操作

---

### 阶段十：调试技术（1-2 周）

**目标**：熟练使用内核调试工具

```
工具清单：
├── printk 调试
│   ├── 日志级别
│   └── dynamic_debug
│
├── ftrace
│   ├── echo function > current_tracer
│   ├── echo func_name > set_ftrace_filter
│   └── trace-cmd 工具
│
├── perf
│   ├── perf top
│   ├── perf record / report
│   └── 火焰图生成
│
├── GDB + QEMU
│   ├── 内核调试配置
│   └── 断点调试
│
└── crash 工具
    └── 内核崩溃分析
```

**实践任务**：
1. 使用 ftrace 追踪函数调用
2. 使用 perf 分析性能瓶颈
3. 配置 QEMU + GDB 调试环境

---

## 四、实践项目清单

按难度递进：

| 序号 | 项目 | 技能点 | 难度 |
|------|------|--------|------|
| 1 | Hello World 内核模块 | 模块基础 | ⭐ |
| 2 | 带参数的内核模块 | 模块参数 | ⭐ |
| 3 | /proc 文件系统接口 | procfs | ⭐ |
| 4 | 字符设备驱动（虚拟设备） | cdev, file_ops | ⭐⭐ |
| 5 | 使用 ioctl 的字符设备 | ioctl 设计 | ⭐⭐ |
| 6 | GPIO LED 驱动 | 平台驱动，设备树 | ⭐⭐ |
| 7 | 按键中断驱动 | 中断处理 | ⭐⭐ |
| 8 | I2C 传感器驱动 | I2C 子系统 | ⭐⭐⭐ |
| 9 | SPI 设备驱动 | SPI 子系统 | ⭐⭐⭐ |
| 10 | 简单块设备驱动 | 块层 | ⭐⭐⭐ |
| 11 | 简单内存文件系统 | VFS | ⭐⭐⭐⭐ |
| 12 | 简单网络设备驱动 | 网络子系统 | ⭐⭐⭐⭐ |
| 13 | 向内核提交补丁 | 社区流程 | ⭐⭐⭐ |

---

## 五、推荐资源

### 5.1 书籍

| 书籍 | 侧重 | 推荐度 |
|------|------|--------|
| 《Linux内核设计与实现》 | 概念入门，300页 | ⭐⭐⭐⭐⭐ 必读 |
| 《Linux设备驱动程序》(LDD3) | 驱动开发 | ⭐⭐⭐⭐⭐ 必读 |
| 《深入理解Linux内核》 | 内核细节 | ⭐⭐⭐⭐ 参考 |
| 《深入Linux内核架构》 | 最全面 | ⭐⭐⭐⭐ 参考 |
| 《Linux内核源代码情景分析》 | 源码分析 | ⭐⭐⭐ |
| 《Linux设备驱动开发详解》 | 中文，实践 | ⭐⭐⭐⭐ |

### 5.2 在线资源

```
官方文档：
├── https://www.kernel.org/doc/html/latest/
├── Documentation/ 目录

社区：
├── https://kernelnewbies.org （新手友好）
├── https://lwn.net （深度文章）
├── LKML 邮件列表

教程：
├── https://linux-kernel-labs.github.io/
├── https://bootlin.com/training/
└── https://elinux.org/
```

### 5.3 视频课程

| 课程 | 内容 | 平台 |
|------|------|------|
| 韦东山 Linux 驱动 | 驱动开发实战 | B站/官网 |
| 宋宝华 Linux 内核 | 内核原理 | 网易云课堂 |
| Bootlin 培训材料 | 嵌入式 Linux | 官网免费 |

### 5.4 开发板推荐

| 开发板 | 价格 | 适合 |
|--------|------|------|
| 树莓派 4B | ~300 元 | 入门，资料多 |
| STM32MP157 | ~400 元 | 工业级，ST 生态 |
| RK3568 | ~500 元 | 性能好，国产 |
| BeagleBone Black | ~400 元 | 经典，TI 芯片 |

---

## 六、面试准备

### 6.1 高频考点

```
内存管理：
├── 伙伴系统原理
├── slab 分配器原理
├── 虚拟地址到物理地址转换
├── 缺页异常处理流程
└── OOM 处理机制

进程调度：
├── CFS 调度器原理
├── 进程状态转换
├── 上下文切换过程
└── 实时调度策略

同步机制：
├── spinlock vs mutex（最高频）
├── RCU 原理和使用场景
├── 死锁避免
└── 原子操作实现

中断处理：
├── 中断上下文 vs 进程上下文
├── 上半部 vs 下半部
├── softirq vs tasklet vs workqueue
└── 中断处理函数注意事项

驱动开发：
├── 字符设备驱动框架
├── 平台设备驱动
├── 设备树语法和匹配
└── DMA 原理和使用
```

### 6.2 常见面试题

**基础题**：
1. Linux 内核态和用户态的区别？如何切换？
2. 系统调用的完整流程？
3. 进程和线程的区别？内核如何实现线程？
4. fork 的实现原理？写时复制是什么？
5. 虚拟内存的作用？页表结构？

**同步题**：
1. spinlock 和 mutex 的区别？各自使用场景？
2. 中断上下文能否使用 mutex？为什么？
3. 什么是 RCU？适用什么场景？
4. 如何避免死锁？
5. 原子操作的实现原理？

**内存题**：
1. kmalloc 和 vmalloc 的区别？
2. slab 分配器解决什么问题？
3. 内存泄漏如何检测？
4. 什么是内存屏障？什么时候需要？

**驱动题**：
1. 字符设备驱动的编写步骤？
2. 中断处理函数有什么限制？
3. 上半部和下半部如何划分？
4. 设备树的作用？如何匹配驱动？
5. DMA 和 PIO 的区别？

---

## 七、学习时间规划

```
基础扎实路线（3-4 个月）：
├── 第1月：内核基础 + 模块开发 + 同步机制
├── 第2月：内存管理 + 字符设备驱动
├── 第3月：中断 + 平台驱动 + 设备树
└── 第4月：实践项目 + 面试准备

快速路线（2 个月）：
├── 第1月：模块 + 字符设备 + 中断
└── 第2月：平台驱动 + 设备树 + 实践

建议：
├── 每天 2-3 小时学习
├── 理论:实践 = 3:7
└── 一定要动手写代码，光看书没用
```

---

## 八、总结

### 学习优先级

```
最重要（必须掌握）：
1. 内核模块开发
2. 同步机制（spinlock/mutex/RCU）
3. 字符设备驱动
4. 中断处理
5. 平台驱动 + 设备树

重要：
6. 内存管理原理
7. 调度器原理
8. 调试技术

加分项：
9. 网络子系统
10. 文件系统
11. 内核社区贡献
```

### 核心建议

1. **买开发板**：实践比看书重要 10 倍
2. **读源码**：从 drivers/staging/ 开始
3. **写代码**：每个知识点都要写 demo
4. **参与社区**：提交补丁是最好的证明
5. **坚持**：内核学习曲线陡峭，但一旦入门会越来越顺

---

> 文档版本：v1.0
> 创建日期：2024
> 配合项目：Linux-System-Design (ECE 391)
