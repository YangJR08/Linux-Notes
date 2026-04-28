# Linux 内核启动笔记（ARM32 / 以 i.MX6U 场景为主）

## 1. 启动全流程总览

从上电到用户态 init 的主线：

1. BootROM 加载并运行 U-Boot。
2. U-Boot 初始化硬件、加载内核镜像（zImage/uImage）、设备树（dtb）和可选 initramfs。
3. U-Boot 按 ARM Linux 启动约定设置寄存器并跳转到内核入口。
4. 内核早期汇编代码（`stext`）完成 CPU 检查、页表创建、打开 MMU。
5. 跳入 `start_kernel()`，完成内核子系统初始化。
6. `rest_init()` 创建关键内核线程：`kernel_init`（PID 1）和 `kthreadd`（PID 2）。
7. `kernel_init` 挂载根文件系统，执行用户空间 init（如 `/sbin/init`）。
8. 系统进入用户态，继续启动服务。

---

## 2. U-Boot 跳转内核前的关键条件（ARM32）

常见要求（经典 ARM32 启动协议）：

1. MMU 关闭。
2. 数据 Cache（D-Cache）关闭。
3. I-Cache 可开可关（通常不作为硬性要求）。
4. `r0 = 0`。
5. `r1 = machine type`（老式 ATAGS 启动方式使用）。
6. `r2 = ATAGS 地址或 DTB 地址`。

说明：

- 现代系统基本使用设备树（DTB），ATAGS 已逐步淘汰。
- 使用 DTB 时，`r1` 的 machine type 通常不再是核心依赖，但很多资料仍保留该约定描述。

---

## 3. 早期汇编阶段：从 `stext` 到打开 MMU

内核入口位于 `arch/arm/kernel/head.S` 的 `stext`。

核心动作可理解为：

1. `safe_svcmode_maskall`
- 确保 CPU 处于 SVC 模式，屏蔽中断。

2. 识别 CPU 并匹配处理器信息
- 读取 CPUID。
- 调用 `__lookup_processor_type`，匹配 `proc_info_list`。
- 若找不到匹配 CPU，内核无法继续。

3. 校验启动参数
- 调用 `__vet_atags`，检查 ATAGS/DTB 参数合法性。

4. 创建页表
- 调用 `__create_page_tables` 建立早期页表。

5. 准备并打开 MMU
- 保存后续跳转地址（如 `__mmap_switched`）。
- 调用 `__enable_mmu`，最终切换到 MMU 开启后的执行环境。

6. 进入 C 语言世界
- `__mmap_switched` 最终调用 `start_kernel()`。

---

## 4. `start_kernel()` 阶段做什么

`start_kernel()` 是内核初始化主函数（`init/main.c`）。

可以把它理解为“内核总调度入口”，主要完成：

1. 内存管理、调度器、中断、时间子系统等核心框架初始化。
2. 驱动模型和总线框架初始化。
3. 为后续创建设备、挂载文件系统、启动用户态做准备。
4. 最后调用 `rest_init()` 进入内核后期启动阶段。

---

## 5. `rest_init()` 与 3 个关键进程

`rest_init()` 的关键结果：

1. 创建 `kernel_init`（PID 1）
- 这是后续过渡到用户态 init 的核心线程。

2. 创建 `kthreadd`（PID 2）
- 这是内核线程管理者，负责内核线程创建请求。
- 注意：它不是“所有进程调度器”，真正调度由调度器子系统完成。

3. 当前上下文转入 idle（PID 0）
- idle 是每个 CPU 的空闲任务。
- 当无可运行任务时执行 idle 循环，有任务就被抢占。

---

## 6. `kernel_init` 如何进入用户空间

`kernel_init` 的关键路径：

1. 调用 `kernel_init_freeable()`
- 进行后续初始化，例如 `do_basic_setup()`（驱动/子系统后续初始化）。

2. 打开控制台
- 打开 `/dev/console` 作为标准输入。
- 复制得到标准输出和标准错误。

3. 挂载根文件系统
- 通过 `prepare_namespace()` 等路径处理 `root=` 参数并挂载根文件系统。

4. 执行用户态 init
- 优先尝试 `rdinit=` 指定程序（常见默认 `/init`）。
- 其次尝试 `init=` 指定程序。
- 最后按顺序回退：`/sbin/init`、`/etc/init`、`/bin/init`、`/bin/sh`。
- 全部失败则内核 panic（无法启动用户空间）。

---

## 7. 常用启动参数（bootargs）

示例：

```bash
console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw
```

含义：

1. `console=ttymxc0,115200`
- 指定内核日志和控制台设备及波特率。

2. `root=/dev/mmcblk1p2`
- 指定根文件系统设备。

3. `rootwait`
- 等待根设备就绪（对 eMMC/SD 启动很常见）。

4. `rw`
- 根文件系统以可读写方式挂载。

可选：

- `init=/linuxrc`：指定用户态 init 程序。
- `rdinit=/init`：指定 initramfs 中的 init 程序。

---

## 8. 易错点与纠正

1. “只要看到 PID 1 就已经是用户态”是错误的
- PID 1 最初是内核线程上下文，执行到 `run_init_process` 后才切到用户态程序。

2. “kthreadd 负责所有进程调度”是错误的
- `kthreadd` 主要负责内核线程创建与管理，调度由内核调度器负责。

3. “ATAGS 与 DTB 同时必须存在”是错误的
- 现代 ARM Linux 通常只需 DTB；ATAGS 主要是历史兼容机制。

4. “内核启动只和 zImage 有关”不完整
- 实际上还强依赖：正确的 DTB、可用根文件系统、正确 bootargs。

---
