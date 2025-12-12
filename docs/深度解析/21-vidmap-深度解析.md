# vidmap 深度解析

## 概述

vidmap 是一个系统调用，允许用户程序直接访问显存。这对于需要高性能图形输出的用户程序很有用（如游戏、图形演示）。

### 为什么需要 vidmap？

1. **性能** - 避免每次输出都通过系统调用
2. **灵活性** - 用户程序可以自由控制显示内容
3. **直接访问** - 绕过终端驱动的抽象层

### 挑战

物理显存地址 (0xB8000) 在内核空间，用户程序无法直接访问。需要：
1. 在用户空间创建一个虚拟地址映射到显存
2. 确保该映射的权限正确（用户可访问）
3. 处理多进程/多终端的情况

## 显存基础

### 文本模式显存布局

```
物理地址 0xB8000 - 0xB8F9F (80×25×2 = 4000 字节)

┌─────────────────────────────────────────────────┐
│ 字符0  │ 属性0  │ 字符1  │ 属性1  │ ... │       │
│ (ASCII)│ (颜色) │ (ASCII)│ (颜色) │     │       │
└─────────────────────────────────────────────────┘
   偏移0     1        2        3

屏幕坐标 (x, y) 对应的显存偏移:
offset = (y * 80 + x) * 2

属性字节:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
├───┴───┴───┴───┼───┴───┴───┴───┤
│   背景色      │    前景色      │
└───────────────┴───────────────┘
bit 7: 闪烁
bits 6-4: 背景色 (0-7)
bits 3-0: 前景色 (0-15)
```

### 颜色值

| 值 | 颜色 | 值 | 颜色 |
|----|------|----|----|
| 0 | 黑色 | 8 | 深灰 |
| 1 | 蓝色 | 9 | 亮蓝 |
| 2 | 绿色 | 10 | 亮绿 |
| 3 | 青色 | 11 | 亮青 |
| 4 | 红色 | 12 | 亮红 |
| 5 | 品红 | 13 | 亮品红 |
| 6 | 棕色 | 14 | 黄色 |
| 7 | 浅灰 | 15 | 白色 |

## vidmap 实现

### 系统调用接口

```c
/*
 * vidmap - 将显存映射到用户空间
 *
 * 参数:
 *   screen_start - 指向用户指针的指针
 *                  成功时，*screen_start 指向映射的显存
 *
 * 返回:
 *   0 成功，-1 失败
 *
 * 使用示例:
 *   uint8_t* video;
 *   vidmap(&video);
 *   video[0] = 'A';      // 显示字符
 *   video[1] = 0x0F;     // 白色
 */
int32_t vidmap(uint8_t** screen_start);
```

### 地址选择

需要选择一个用户空间的虚拟地址来映射显存：

```
用户虚拟地址空间:
┌─────────────────┐ 0xFFFFFFFF
│                 │
│    内核空间      │
│                 │
├─────────────────┤ 0x08400000 (132MB)
│   用户栈        │
│                 │
├─────────────────┤ 0x08048000
│   用户程序      │
│                 │
├─────────────────┤ 0x08000000 (128MB)
│                 │
│   未映射        │
│                 │
├─────────────────┤ 0x08800000 (136MB) ← vidmap 映射到这里
│   显存映射      │  (一个 4KB 页)
├─────────────────┤
│                 │
└─────────────────┘ 0x00000000
```

选择 `0x08800000` (136MB) 的原因：
1. 在用户空间范围内 (0-4GB)
2. 不与用户程序区域 (128-132MB) 冲突
3. 容易记忆（128MB + 8MB = 136MB）

### 实现代码

```c
#define VIDEO_VIRT_ADDR  0x08800000   // 用户空间显存虚拟地址
#define VIDEO_PHYS_ADDR  0x000B8000   // 物理显存地址

/*
 * vidmap - 将显存映射到用户空间
 */
int32_t vidmap(uint8_t** screen_start) {
    // 1. 参数检查
    if (screen_start == NULL) {
        return -1;
    }

    // 检查指针是否在用户空间
    if ((uint32_t)screen_start < USER_SPACE_START ||
        (uint32_t)screen_start >= USER_SPACE_END) {
        return -1;
    }

    // 2. 设置页表映射
    setup_vidmap_page();

    // 3. 返回映射地址
    *screen_start = (uint8_t*)VIDEO_VIRT_ADDR;

    return 0;
}
```

### 页表设置

vidmap 需要使用 4KB 页（不是 4MB 页），因为显存只有 4KB：

```c
// 页目录索引 34 用于 vidmap (136MB / 4MB = 34)
#define VIDMAP_PDE_INDEX  34

// 页表（需要预先分配）
uint32_t vidmap_page_table[1024] __attribute__((aligned(4096)));

/*
 * setup_vidmap_page - 设置 vidmap 的页表映射
 */
void setup_vidmap_page(void) {
    // 计算页表项索引
    // 136MB = 0x08800000
    // PTE 索引 = (0x08800000 >> 12) & 0x3FF = 0
    int pte_index = (VIDEO_VIRT_ADDR >> 12) & 0x3FF;

    // 设置页表项
    // Present | R/W | User | 物理地址
    vidmap_page_table[pte_index] = VIDEO_PHYS_ADDR | 0x07;
    //                                               │││
    //                                               ││└─ Present
    //                                               │└── R/W
    //                                               └─── User

    // 设置页目录项
    // Present | R/W | User | 页表物理地址
    page_directory[VIDMAP_PDE_INDEX] =
        (uint32_t)vidmap_page_table | 0x07;

    // 刷新 TLB
    flush_tlb();
}

/*
 * flush_tlb - 刷新 TLB
 */
void flush_tlb(void) {
    asm volatile(
        "movl %%cr3, %%eax\n"
        "movl %%eax, %%cr3\n"
        :
        :
        : "eax"
    );
}
```

### 页表结构图

```
                    页目录
                 ┌─────────┐
              0  │ 内核0-4M │ → 4MB 页
              1  │ 内核4-8M │ → 4MB 页
                 │   ...   │
             32  │ 用户程序 │ → 4MB 页 (128-132MB)
                 │   ...   │
             34  │ vidmap  │ ─────┐
                 │   ...   │      │
                 └─────────┘      │
                                  │
                                  ▼
                           vidmap_page_table
                           ┌─────────┐
                        0  │ 显存    │ → 0xB8000 (物理)
                        1  │ 未使用  │
                           │   ...   │
                           └─────────┘

映射关系:
虚拟地址 0x08800000 → PDE[34] → vidmap_page_table[0] → 物理 0xB8000
```

## 多终端支持

### 问题

在多终端系统中，每个终端有自己的视频备份缓冲区：
- 终端 0: 视频备份在 0xB9000
- 终端 1: 视频备份在 0xBA000
- 终端 2: 视频备份在 0xBB000

只有当前显示的终端应该写入真实显存 (0xB8000)，其他终端应该写入备份缓冲区。

### 解决方案

根据进程所属终端动态设置映射：

```c
/*
 * vidmap 多终端版本
 */
int32_t vidmap(uint8_t** screen_start) {
    if (screen_start == NULL) return -1;
    if ((uint32_t)screen_start < USER_SPACE_START) return -1;

    // 获取当前进程所属终端
    pcb_t* pcb = get_current_pcb();
    int term_id = pcb->term_id;

    // 根据终端设置映射
    setup_vidmap_for_terminal(term_id);

    *screen_start = (uint8_t*)VIDEO_VIRT_ADDR;
    return 0;
}

/*
 * setup_vidmap_for_terminal - 为特定终端设置 vidmap
 */
void setup_vidmap_for_terminal(int term_id) {
    uint32_t phys_addr;

    if (term_id == curr_term) {
        // 当前显示的终端，映射到真实显存
        phys_addr = VIDEO_PHYS_ADDR;  // 0xB8000
    } else {
        // 后台终端，映射到备份缓冲区
        phys_addr = VIDEO_BACKUP_BASE + term_id * 0x1000;
        // 0xB9000, 0xBA000, 0xBB000
    }

    int pte_index = (VIDEO_VIRT_ADDR >> 12) & 0x3FF;
    vidmap_page_table[pte_index] = phys_addr | 0x07;

    flush_tlb();
}
```

### 终端切换时的处理

当用户切换终端 (Alt+F1/F2/F3) 时，需要更新 vidmap 映射：

```c
/*
 * switch_terminal - 切换终端
 */
void switch_terminal(int new_term) {
    if (new_term == curr_term) return;

    // 保存当前终端的显存
    memcpy(term[curr_term].vid_backup, (void*)VIDEO_PHYS_ADDR, 4096);

    // 恢复新终端的显存
    memcpy((void*)VIDEO_PHYS_ADDR, term[new_term].vid_backup, 4096);

    // 更新当前终端
    curr_term = new_term;

    // 更新 vidmap 映射（如果新终端的进程使用了 vidmap）
    update_vidmap_for_current_terminal();
}

/*
 * update_vidmap_for_current_terminal - 更新 vidmap 映射
 */
void update_vidmap_for_current_terminal(void) {
    // 遍历所有进程，更新使用 vidmap 的进程的映射
    int i;
    for (i = 0; i < MAX_PROCESSES; i++) {
        pcb_t* pcb = get_specific_pcb(i);
        if (pcb->vidmap_enabled) {
            // 如果进程属于当前终端，映射到真实显存
            // 否则映射到备份缓冲区
            update_process_vidmap(i);
        }
    }
}
```

## PCB 中的 vidmap 标志

```c
typedef struct pcb {
    // ... 其他字段

    uint8_t vidmap_enabled;  // 该进程是否启用了 vidmap
    int term_id;             // 进程所属终端
} pcb_t;

// 在 vidmap 中设置
int32_t vidmap(uint8_t** screen_start) {
    // ...
    pcb_t* pcb = get_current_pcb();
    pcb->vidmap_enabled = 1;
    // ...
}

// 在 halt 中清除
int32_t halt(uint8_t status) {
    pcb_t* pcb = get_current_pcb();
    pcb->vidmap_enabled = 0;
    // ...
}
```

## 用户程序使用示例

### 基本使用

```c
#include "lib.h"

int main() {
    uint8_t* video;

    // 获取显存映射
    if (vidmap(&video) != 0) {
        printf("vidmap failed!\n");
        return 1;
    }

    // 直接写入显存
    // 在 (0,0) 位置显示红色的 'H'
    video[0] = 'H';
    video[1] = 0x04;  // 红色前景，黑色背景

    // 在 (1,0) 位置显示绿色的 'i'
    video[2] = 'i';
    video[3] = 0x02;  // 绿色前景，黑色背景

    return 0;
}
```

### 清屏函数

```c
void clear_screen(uint8_t* video, uint8_t attr) {
    int i;
    for (i = 0; i < 80 * 25; i++) {
        video[i * 2] = ' ';
        video[i * 2 + 1] = attr;
    }
}
```

### 绘制矩形

```c
void draw_rect(uint8_t* video, int x, int y,
               int width, int height, char c, uint8_t attr) {
    int i, j;
    for (j = y; j < y + height && j < 25; j++) {
        for (i = x; i < x + width && i < 80; i++) {
            int offset = (j * 80 + i) * 2;
            video[offset] = c;
            video[offset + 1] = attr;
        }
    }
}
```

### 动画示例

```c
void animation_demo(uint8_t* video) {
    int x = 0;
    int direction = 1;

    while (1) {
        // 清除旧位置
        clear_screen(video, 0x00);

        // 绘制球
        int offset = (12 * 80 + x) * 2;  // 中间行
        video[offset] = 'O';
        video[offset + 1] = 0x0E;  // 黄色

        // 移动
        x += direction;
        if (x >= 79) direction = -1;
        if (x <= 0) direction = 1;

        // 延时（简单忙等待）
        volatile int delay;
        for (delay = 0; delay < 1000000; delay++);
    }
}
```

## 常见问题

### 1. vidmap 返回 -1

**可能原因**：
- 参数为 NULL
- 指针不在用户空间

**排查**：
```c
int32_t vidmap(uint8_t** screen_start) {
    printf("vidmap: screen_start = 0x%x\n", screen_start);

    if (screen_start == NULL) {
        printf("vidmap: NULL pointer\n");
        return -1;
    }

    if ((uint32_t)screen_start < 0x08000000) {
        printf("vidmap: pointer not in user space\n");
        return -1;
    }

    // ...
}
```

### 2. 写入显存后无显示

**可能原因**：
- 页表未正确设置
- 映射到了错误的物理地址
- TLB 未刷新

**排查**：
```c
void debug_vidmap_mapping(void) {
    printf("PDE[34] = 0x%x\n", page_directory[34]);

    uint32_t* pt = (uint32_t*)(page_directory[34] & 0xFFFFF000);
    printf("PTE[0] = 0x%x\n", pt[0]);

    // 检查物理地址
    uint32_t phys = pt[0] & 0xFFFFF000;
    printf("Physical address = 0x%x\n", phys);
}
```

### 3. 多终端 vidmap 显示错乱

**可能原因**：
- 终端切换时未更新映射
- 进程的 term_id 设置错误

**排查**：
```c
void debug_terminal_vidmap(void) {
    printf("curr_term = %d\n", curr_term);

    pcb_t* pcb = get_current_pcb();
    printf("Process term_id = %d\n", pcb->term_id);
    printf("vidmap_enabled = %d\n", pcb->vidmap_enabled);
}
```

### 4. Page Fault 访问 vidmap 地址

**可能原因**：
- 页目录项未设置
- 页表项未设置
- 用户权限位未设置

**排查**：
```c
// 检查页表项的所有标志位
void check_vidmap_pte(void) {
    uint32_t pte = vidmap_page_table[0];

    printf("PTE = 0x%x\n", pte);
    printf("  Present: %d\n", pte & 1);
    printf("  R/W: %d\n", (pte >> 1) & 1);
    printf("  U/S: %d\n", (pte >> 2) & 1);  // 必须为 1
    printf("  Phys: 0x%x\n", pte & 0xFFFFF000);
}
```

## 安全考虑

### 1. 地址验证

必须验证 `screen_start` 指针在用户空间：

```c
#define USER_SPACE_START  0x08000000
#define USER_SPACE_END    0x08400000

if ((uint32_t)screen_start < USER_SPACE_START ||
    (uint32_t)screen_start >= USER_SPACE_END) {
    return -1;
}
```

### 2. 进程隔离

每个进程的 vidmap 映射应该独立：
- 使用每进程的页表（如果实现）
- 或在进程切换时更新全局页表

### 3. 资源清理

进程退出时必须清除 vidmap 映射：

```c
int32_t halt(uint8_t status) {
    pcb_t* pcb = get_current_pcb();

    // 清除 vidmap
    if (pcb->vidmap_enabled) {
        clear_vidmap_page();
        pcb->vidmap_enabled = 0;
    }

    // ...
}
```

## 与 fish 程序的关系

MP3 提供的 `fish` 测试程序使用 vidmap 显示动画鱼：

```c
// fish.c 简化版
int main() {
    uint8_t* video;

    if (vidmap(&video) != 0) {
        return 1;
    }

    // 显示动画鱼
    while (1) {
        draw_fish(video, x, y);
        // 移动鱼
        // 延时
    }

    return 0;
}
```

`fish` 是测试 vidmap 实现的经典方法。

## 实现检查清单

- [ ] 定义 vidmap 虚拟地址 (如 0x08800000)
- [ ] 分配 vidmap 页表 (4KB 对齐)
- [ ] 实现 vidmap 系统调用
- [ ] 验证 screen_start 指针
- [ ] 设置页目录项 (User, R/W, Present)
- [ ] 设置页表项 (User, R/W, Present)
- [ ] 刷新 TLB
- [ ] 在 PCB 中记录 vidmap 状态
- [ ] 在 halt 中清除 vidmap
- [ ] (可选) 多终端 vidmap 支持
- [ ] 测试 fish 程序

## 总结

| 组件 | 说明 |
|-----|------|
| 虚拟地址 | 0x08800000 (136MB) |
| 物理地址 | 0xB8000 (显存) |
| 页大小 | 4KB |
| PDE 索引 | 34 |
| PTE 索引 | 0 |
| 权限 | User, R/W, Present (0x07) |

vidmap 的核心要点：
1. **4KB 页映射** - 显存只需要一个 4KB 页
2. **用户权限** - 必须设置 User 位
3. **TLB 刷新** - 修改页表后必须刷新
4. **多终端** - 根据终端决定映射到真实显存还是备份
5. **资源清理** - 进程退出时清除映射
