# Cortex-A7 中断工作机制笔记

## 一、整体结构

Cortex-A7（ARMv7-A）平台中的中断处理通常由 CPU 内核 + GIC（Generic Interrupt Controller）共同完成。

其中：

* 外设负责产生中断请求
* GIC 负责中断收集、优先级仲裁、目标 CPU 路由
* Cortex-A7 负责响应异常、保存现场、执行中断处理程序
* 软件负责根据中断号调用具体 ISR（Interrupt Service Routine）

因此，中断系统不是“外设直接跳到 ISR”，而是经过 GIC 和 CPU 异常机制的分层处理流程。

---

## 二、一次外部中断的完整流程

### 1. 外设产生中断请求

例如 UART 接收到数据、Timer 超时、GPIO 电平变化等，外设会拉起中断信号线。

---

### 2. GIC Distributor（分发器端）接收中断

Distributor 是 GIC 的全局控制部分，负责管理所有共享中断源。

主要处理内容：

* 全局中断使能控制（Distributor 全局开关）
* 控制每个中断的使能或关闭
* 设置每个中断的优先级
* 设置每个中断的目标处理器列表（路由到哪些 CPU 核）
* 设置每个外部中断的触发模式（电平触发或边沿触发）
* 设置每个中断属于 Group 0 或 Group 1
* 维护中断的 pending/active 状态，并从 pending 中断中仲裁出最高优先级中断

若条件满足，则将中断发送到目标核的 CPU Interface。

---

### 3. 中断使能（总开关 + ID 中断源开关）

中断使能包含两部分：

* IRQ/FIQ 总中断使能（总开关）
* ID0~ID1019 中断源使能（分开关）

可以类比“总电闸 + 各电器开关”：

* IRQ/FIQ 总开关相当于进户总闸
* 各 ID 中断使能相当于每个电器开关

如果要使用 i.MX6U 外设中断，至少要先打开 IRQ（本笔记默认使用 IRQ，不使用 FIQ）。

#### 1. IRQ 和 FIQ 总中断使能

CPSR 中相关位含义：

* I=1：禁止 IRQ，I=0：使能 IRQ
* F=1：禁止 FIQ，F=0：使能 FIQ

GNU ARM 汇编（GCC）下可直接使用 `cps` 指令：

| 指令 | 描述 |
| --- | --- |
| `cpsid i` | 禁止 IRQ 中断 |
| `cpsie i` | 使能 IRQ 中断 |
| `cpsid f` | 禁止 FIQ 中断 |
| `cpsie f` | 使能 FIQ 中断 |

示例：

```asm
@ 先关闭 IRQ，完成关键配置后再打开 IRQ
cpsid i
@ ... 配置 GIC/外设寄存器 ...
cpsie i
```

#### 2. ID0~ID1019 中断使能和禁止

GIC 使用以下寄存器控制每个中断 ID 的使能状态：

* `GICD_ISENABLERn`：使能中断
* `GICD_ICENABLERn`：禁止中断

每个 bit 对应一个中断 ID。GIC 架构上可到 ID0~ID1019（1020 个中断源）；在 Cortex-A7 的常见实现中通常只使用前 512 个 ID，因此需要：

* $512 / 32 = 16$ 个 `GICD_ISENABLERn`
* $512 / 32 = 16$ 个 `GICD_ICENABLERn`

典型映射关系：

* `GICD_ISENABLER0` 的 bit[15:0] 对应 SGI（ID0~15）
* `GICD_ISENABLER0` 的 bit[31:16] 对应 PPI（ID16~31）
* `GICD_ISENABLER1` ~ `GICD_ISENABLER15` 对应 SPI（ID32 及以上）

同理，`GICD_ICENABLER0` ~ `GICD_ICENABLER15` 用于对应 ID 的禁止。

---

### 4. 中断优先级设置

Cortex-A7 的中断优先级可分为抢占优先级和子优先级，二者都可配置。

GIC 最多支持 256 级优先级（数值越小优先级越高），i.MX6U（Cortex-A7）实际使用 32 级优先级。

#### 1. 优先级掩码配置（GICC_PMR）

`GICC_PMR` 低 8 位有效，用于设置 CPU Interface 的优先级掩码阈值（屏蔽哪些较低优先级中断上报到 CPU Core）。

优先级“有多少级”由硬件决定（例如 i.MX6U 常用 32 级），不是通过 `GICC_PMR` 来选择。

常见的优先级位宽与级数关系如下：

| bit[7:0] | 优先级数 |
| --- | --- |
| `11111111` | 256 个优先级 |
| `11111110` | 128 个优先级 |
| `11111100` | 64 个优先级 |
| `11111000` | 32 个优先级 |
| `11110000` | 16 个优先级 |

i.MX6U 使用 32 级优先级时，常见做法是将 `GICC_PMR` 设置为 `0b11111000`（或按平台手册要求设置更宽松阈值），以避免低优先级中断被过度屏蔽。

#### 2. 抢占优先级和子优先级位数（GICC_BPR）

`GICC_BPR`（Binary Point Register）低 3 位有效，用于划分抢占优先级域和子优先级域：

| Binary Point | 抢占优先级域 | 子优先级域 | 描述 |
| --- | --- | --- | --- |
| 0 | [7:1] | [0] | 7 位抢占优先级，1 位子优先级 |
| 1 | [7:2] | [1:0] | 6 位抢占优先级，2 位子优先级 |
| 2 | [7:3] | [2:0] | 5 位抢占优先级，3 位子优先级 |
| 3 | [7:4] | [3:0] | 4 位抢占优先级，4 位子优先级 |
| 4 | [7:5] | [4:0] | 3 位抢占优先级，5 位子优先级 |
| 5 | [7:6] | [5:0] | 2 位抢占优先级，6 位子优先级 |
| 6 | [7:7] | [6:0] | 1 位抢占优先级，7 位子优先级 |
| 7 | 无 | [7:0] | 0 位抢占优先级，8 位子优先级 |

为了简化中断管理，常把有效优先级位尽量用于抢占优先级。对 i.MX6U 的 32 级优先级场景，常用做法是设置 `Binary Point = 2`。

#### 3. 指定中断 ID 的优先级设置（GICD_IPRIORITYRn）

每个中断 ID 对应一个优先级寄存器项（常见写法为 `GICD_IPRIORITYRn` 数组项）。

* Cortex-A7 常见实现使用 512 个中断 ID
* 优先级数为 32 时，通常使用优先级寄存器高位字段，实际优先级值需要左移 3 位写入

示例：将 ID40 的优先级设置为 5（数值越小优先级越高）：

```asm
@ GICD_IPRIORITYRn 常见基址偏移从 0x400 开始，每个 ID 对应 1 字节
@ 32 级优先级时，将优先级左移 3 位后写入
.equ GICD_BASE,            0x00A00000
.equ GICD_IPRIORITYR_OFF,  0x400

ldr r0, =GICD_BASE
mov r1, #40
add r0, r0, #GICD_IPRIORITYR_OFF
add r0, r0, r1
mov r2, #(5 << 3)
strb r2, [r0]
```

优先级设置可归纳为三步：

1. 确认平台支持的优先级级数（如 i.MX6U 为 32 级），并配置 `GICC_PMR` 作为优先级掩码阈值
2. 配置 `GICC_BPR`，划分抢占优先级和子优先级位宽
3. 配置指定中断 ID 的 `GICD_IPRIORITYR`，设置外设中断优先级

---

### 5. GIC CPU Interface 通知 Cortex-A7 内核

每个 CPU 核都有自己的 CPU Interface。

它负责：

* 使能或关闭发送到 CPU Core 的中断请求信号
* 接收 Distributor 分发来的中断
* 中断应答（读取中断应答寄存器，获取当前 irq_id）
* 通知中断处理完成（写 EOIR）
* 设置优先级掩码（通过掩码过滤不需要上报给 CPU Core 的中断）
* 定义抢占策略（如优先级分组/二进制点）
* 当多个中断同时到来时，选择优先级最高者通知给 CPU Core
* 向 CPU 输出 IRQ 或 FIQ 信号

若 CPU 当前未屏蔽中断，则 Cortex-A7 开始进入异常流程。

---

## 三、CPU 接收到 IRQ 后发生的事情

当 Cortex-A7 接收到 IRQ：

1. 当前执行流被打断
2. 保存返回地址到 LR_irq
3. 保存当前 CPSR 到 SPSR_irq
4. CPU 切换到 IRQ mode
5. 自动屏蔽新的 IRQ（通常 CPSR.I=1）
6. PC 跳转到 IRQ 异常向量入口

这一步跳转的目标不是具体设备 ISR，而是 IRQ 总入口。

---

## 四、异常向量表的作用

ARMv7-A 使用异常向量表管理异常入口，Cortex-A7 内核有 8 个异常中断，这 8 个异常中断的中断向量表如下：

| 向量地址 | 中断类型 | 中断模式 |
| --- | --- | --- |
| 0x00 | 复位中断（Reset） | 特权模式（SVC） |
| 0x04 | 未定义指令中断（Undefined Instruction） | 未定义指令中止模式（Undef） |
| 0x08 | 软中断（Software Interrupt, SWI） | 特权模式（SVC） |
| 0x0C | 指令预取中止中断（Prefetch Abort） | 中止模式（Abort） |
| 0x10 | 数据访问中止中断（Data Abort） | 中止模式（Abort） |
| 0x14 | 未使用（Not Used） | 未使用 |
| 0x18 | IRQ 中断（IRQ Interrupt） | 外部中断模式（IRQ） |
| 0x1C | FIQ 中断（FIQ Interrupt） | 快速中断模式（FIQ） |

发生 IRQ 时，CPU 只跳到 IRQ 对应入口地址。
例如：

```asm
@ GNU ARM 汇编（GCC）风格：Cortex-A 异常向量偏移定义
@ .equ：汇编常量定义，等价于 C 语言 #define
@ 作用：定义 IRQ / FIQ 在向量表中的**硬件固定偏移地址**

.equ IRQ_VECTOR_OFFSET,  0x18    @ IRQ 中断向量偏移地址
.equ FIQ_VECTOR_OFFSET,  0x1C    @ FIQ 快速中断向量偏移地址

@ 与真实向量表代码的对应逻辑：
@ 向量表起始标号：_start
@ IRQ 向量地址 = _start + IRQ_VECTOR_OFFSET  → 对应指令：ldr pc, =IRQ_Handler
@ FIQ 向量地址 = _start + FIQ_VECTOR_OFFSET  → 对应指令：ldr pc, =FIQ_Handler
```

因此这里的“查表”，本质上是先查异常类型入口，而不是查设备 ISR。

---

## 五、IRQ 总入口之后如何找到具体中断源

IRQ 向量入口通常只是一段汇编入口代码，作用包括：

* 保存通用寄存器
* 建立栈环境
* 读取 GIC 中断号
* 分发到具体 ISR

关键步骤：

读取 GICC_IAR（Interrupt Acknowledge Register）

```asm
@ r0 = irq_id
.equ GICC_BASE,       0x00A01000
.equ GICC_IAR_OFFSET, 0x0C

ldr r1, =GICC_BASE
ldr r0, [r1, #GICC_IAR_OFFSET]
```

该寄存器返回当前最高优先级待处理中断号。

例如：

* 29 = Timer
* 33 = UART
* 52 = GPIO

拿到 irq_id 后，再进入软件维护的 ISR 表：

```asm
@ r0 = irq_id
@ irq_table: 每项 4 字节函数入口地址
ldr r2, =irq_table
ldr r3, [r2, r0, lsl #2]
blx r3
```

这才是真正找到具体中断处理函数的过程。

### 实战示例：完整 `start.s` 启动与 IRQ 处理框架

下面这段代码可作为 Cortex-A7（i.MX6U/i.MX6ULL）裸机启动与中断入口的参考模板，包含向量表、复位初始化、模式栈设置、IRQ 现场保护与 EOIR 回写。

```asm

.global _start  /* 定义全局标号 _start */

/* ------------------------------------------------------------
 * 1. 中断向量表
 * ------------------------------------------------------------ */
_start:
    ldr pc, =Reset_Handler      /* 跳转至复位中断处理函数 */
    ldr pc, =Undefined_Handler  /* 跳转至未定义指令中断处理函数 */
    ldr pc, =SVC_Handler        /* 跳转至 SVC (Supervisor) 中断处理函数 */
    ldr pc, =PrefAbort_Handler  /* 跳转至预取终止中断处理函数 */
    ldr pc, =DataAbort_Handler  /* 跳转至数据终止中断处理函数 */
    ldr pc, =NotUsed_Handler    /* 跳转至未使用中断处理函数 */
    ldr pc, =IRQ_Handler        /* 跳转至 IRQ 中断处理函数 */
    ldr pc, =FIQ_Handler        /* 跳转至 FIQ (快速中断) 处理函数 */

/* ------------------------------------------------------------
 * 2. 复位中断处理 (系统初始化核心)
 * ------------------------------------------------------------ */
Reset_Handler:
    cpsid i                     /* 禁止全局中断 (关闭 IRQ) */

    /* 关闭 I-Cache, D-Cache, MMU 和分支预测 (通过读-改-写 CP15 C1 寄存器) */
    mrc p15, 0, r0, c1, c0, 0   /* 将 CP15 的 C1 寄存器值读取到 R0 中 */
    bic r0, r0, #(0x1 << 12)    /* 清除 r0 的第 12 位，即关闭 I-Cache (指令缓存) */
    bic r0, r0, #(0x1 << 2)     /* 清除 r0 的第 2 位， 即关闭 D-Cache (数据缓存) */
    bic r0, r0, #0x2            /* 清除 r0 的第 1 位， 即关闭对齐检查 (Alignment Check) */
    bic r0, r0, #(0x1 << 11)    /* 清除 r0 的第 11 位，即关闭分支预测 (Branch Prediction) */
    bic r0, r0, #0x1            /* 清除 r0 的第 0 位， 即关闭 MMU (内存管理单元) */
    mcr p15, 0, r0, c1, c0, 0   /* 将修改后的 r0 值写回 CP15 的 C1 寄存器中 */

/* ------------------------------------------------------------
 * 3. 设置各个模式下的堆栈指针 (Stack Pointer)
 * 注意：I.MX6ULL 堆栈向下增长，必须 4 字节对齐
 * ------------------------------------------------------------ */
    /* 进入 IRQ 模式 */
    mrs r0, cpsr                /* 读取当前程序状态寄存器 CPSR */
    bic r0, r0, #0x1f           /* 清除 r0 的低 5 位 (M0~M4)，准备设置模式 */
    orr r0, r0, #0x12           /* 将 r0 的低 5 位设置为 0x12，即切换到 IRQ 模式 */
    msr cpsr, r0                /* 将修改后的模式写回 CPSR */
    ldr sp, =0x80600000         /* 设置 IRQ 模式栈首地址为 0x80600000 (大小 2MB) */

    /* 进入 SYS 模式 */
    mrs r0, cpsr                /* 读取 CPSR */
    bic r0, r0, #0x1f           /* 清除 r0 的低 5 位 (M0~M4) */
    orr r0, r0, #0x1f           /* 将 r0 的低 5 位设置为 0x1f，即切换到 SYS 模式 */
    msr cpsr, r0                /* 将修改后的模式写回 CPSR */
    ldr sp, =0x80400000         /* 设置 SYS 模式栈首地址为 0x80400000 (大小 2MB) */

    /* 进入 SVC 模式 (管理模式) */
    mrs r0, cpsr                /* 读取 CPSR */
    bic r0, r0, #0x1f           /* 清除 r0 的低 5 位 (M0~M4) */
    orr r0, r0, #0x13           /* 将 r0 的低 5 位设置为 0x13，即切换到 SVC 模式 */
    msr cpsr, r0                /* 将修改后的模式写回 CPSR */
    ldr sp, =0x80200000         /* 设置 SVC 模式栈首地址为 0x80200000 (大小 2MB) */

    cpsie i                     /* 使能全局中断 (开启 IRQ) */
    b main                      /* 跳转到 C 语言 main 函数 */

/* ------------------------------------------------------------
 * 4. IRQ 中断处理逻辑 (关键部分)
 * ------------------------------------------------------------ */
IRQ_Handler:
    push {lr}                   /* 将 IRQ 模式下的返回地址压栈 */
    push {r0-r3, r12}           /* 将常用的通用寄存器压栈保存 (符合 AAPCS 调用约定) */

    mrs r0, spsr                /* 读取保存的状态寄存器 SPSR (保存中断前的模式和标志) */
    push {r0}                   /* 将 SPSR 压栈保存 */

    /* 获取 GIC (中断控制器) 的 CPU 接口寄存器基地址 */
    mrc p15, 4, r1, c15, c0, 0  /* 读取 CP15 的 C15 寄存器 (CBAR) 到 r1，获取外设基地址 */
    add r1, r1, #0x2000         /* r1 加上 0x2000 偏移，定位到 GIC CPU 接口 (GICC) 基址 */
    ldr r0, [r1, #0x0C]         /* 读取 GICC_IAR 寄存器 (偏移 0x0C)，获取当前中断 ID */

    push {r0, r1}               /* 将中断号和 GIC 基地址压栈保存 */

    /* 切换到 SVC 模式以处理 C 语言中断函数，允许中断嵌套 */
    cps #0x13                   /* 切换到 SVC 模式 */
    push {lr}                   /* 保存 SVC 模式下的 lr (返回地址) */
    ldr r2, =system_irqhandler  /* 加载 C 语言中断分配器的函数地址 */
    blx r2                      /* 调用 system_irqhandler，r0 (中断号) 为第一个参数 */

    pop {lr}                    /* 恢复 SVC 模式下的 lr */
    cps #0x12                   /* 切回到 IRQ 模式 */
    pop {r0, r1}                /* 弹出之前保存的中断号和 GIC 基地址 */

    str r0, [r1, #0x10]         /* 将中断号写入 GICC_EOIR 寄存器，通知 GIC 中断执行完成 */

    pop {r0}                    /* 弹出 SPSR 值 */
    msr spsr_cxsf, r0           /* 恢复状态寄存器 SPSR */

    pop {r0-r3, r12}            /* 恢复通用寄存器现场 */
    pop {lr}                    /* 恢复 IRQ 模式下的 lr */
    subs pc, lr, #4             /* 将 lr 减去 4 并赋给 pc，实现异常返回 (修正流水线偏差) */

/* ------------------------------------------------------------
 * 5. 其他异常处理占位 (死循环)
 * ------------------------------------------------------------ */
Undefined_Handler:
    ldr r0, =Undefined_Handler
    bx r0

SVC_Handler:
    ldr r0, =SVC_Handler
    bx r0

PrefAbort_Handler:
    ldr r0, =PrefAbort_Handler
    bx r0

DataAbort_Handler:
    ldr r0, =DataAbort_Handler
    bx r0

NotUsed_Handler:
    ldr r0, =NotUsed_Handler
    bx r0

FIQ_Handler:
    ldr r0, =FIQ_Handler
    bx r0
```

---

## 六、中断处理完成后做什么

ISR 执行结束后，需要通知 GIC 当前中断处理完成。

写入：

```asm
@ r0 = irq_id
.equ GICC_BASE,        0x00A01000
.equ GICC_EOIR_OFFSET, 0x10

ldr r1, =GICC_BASE
str r0, [r1, #GICC_EOIR_OFFSET]
```

EOIR = End Of Interrupt Register

作用：

* 清除 active 状态
* 允许下一次中断继续进入
* 支持嵌套中断管理

若设备本身状态位未清除（如 UART pending bit），中断可能再次触发。

因此通常 ISR 中还要清外设寄存器。

---

## 七、CP15 在中断中的作用

CP15 是 ARMv7-A 的系统控制协处理器接口，负责系统级控制寄存器访问。

它不负责中断仲裁（那是 GIC 的工作），但参与中断运行环境配置。

---

### 1. 设置异常向量表基地址

通过：

VBAR（Vector Base Address Register）

决定异常向量表放在哪里：

```asm
@ 设置 VBAR（示例：向量表地址在 r0）
mcr p15, 0, r0, c12, c0, 0
isb
```

CPU 发生 IRQ 时：

```asm
@ 等效关系：IRQ 入口地址 = VBAR + 0x18
.equ IRQ_VECTOR_OFFSET, 0x18

mrc p15, 0, r1, c12, c0, 0
add r1, r1, #IRQ_VECTOR_OFFSET
```

因此 IRQ 入口位置由 CP15 控制。

---

### 2. 控制高低向量

SCTLR.V 位决定是否使用高地址向量表。

---

### 3. 系统控制

CP15 的 SCTLR 控制：

* MMU
* Cache
* Branch Prediction
* Alignment
* Exception 行为相关设置

这些会影响异常入口执行效率和地址映射。

---

### 4. 获取外设基址（某些平台）

部分 SoC 中，可通过 CP15 的 CBAR 获取 GIC 基地址信息。

---

## 八、GIC 与 CP15 分工总结

GIC 负责：

* 收集中断
* 优先级判断
* 中断路由
* 中断状态管理
* 输出 IRQ/FIQ 给 CPU

CP15 负责：

* 异常向量位置配置
* CPU 系统控制
* MMU/Cache 环境
* 部分平台基址信息

CPU 本体负责：

* 异常响应
* 模式切换
* 保存恢复现场
* 执行 ISR

软件负责：

* 根据 irq_id 查找 handler
* 驱动级中断处理

---

## 九、简化时序图

```text
外设触发中断
    ↓
GIC Distributor
    ↓
目标CPU的CPU Interface
    ↓
输出IRQ给 Cortex-A7
    ↓
CPU进入IRQ异常
    ↓
跳转 IRQ Vector
    ↓
读取 GICC_IAR 得到 irq_id
    ↓
irq_table[irq_id]()
    ↓
执行ISR
    ↓
写 GICC_EOIR
    ↓
异常返回
```

---

## 十、本质理解

Cortex-A7 中断机制本质是三层结构：

第一层：异常入口层
IRQ/FIQ 进入 CPU

第二层：中断识别层
通过 GIC 获取 irq_id

第三层：软件处理层
根据 irq_id 调用 ISR

