# GCC 常用选项笔记

## 一、命令结构（重点）

你给出的公式可以直接作为 GCC 命令的通用写法：

> gcc [不带参数的选项] [带参数的选项 + 参数] [源文件]

常见示例：

```bash
gcc -Wall -g -O2 -I ./include -L ./lib -o my_app main.c
```

说明：

- 不带参数的选项：只写选项本身即可，例如 `-Wall`、`-g`、`-c`、`-E`、`-S`。
- 带参数的选项：选项后面需要一个参数，例如 `-o my_app`、`-I ./include`、`-L ./lib`、`-l m`。

## 二、选项分类说明

| 分类 | 选项示例 | 是否需要参数 | 用途说明 |
| :--- | :--- | :--- | :--- |
| 不带参数 | `-Wall` | 否 | 打开常用警告 |
| 不带参数 | `-g` | 否 | 生成调试信息 |
| 不带参数 | `-c` | 否 | 只编译不链接，生成 `.o` |
| 不带参数 | `-E` | 否 | 只做预处理 |
| 不带参数 | `-S` | 否 | 生成汇编文件 `.s` |
| 不带参数 | `-O2` | 否 | 开启优化等级 2 |
| 带参数 | `-o my_app` | 是 | 指定输出文件名 |
| 带参数 | `-I ./include` / `-I include` | 是 | 指定头文件搜索路径 |
| 带参数 | `-L ./lib` | 是 | 指定库文件搜索路径 |
| 带参数 | `-l m` | 是 | 链接库 `libm` |
| 带参数 | `-D DEBUG` / `-DDEBUG` | 是 | 定义宏 |

## 三、基础核心选项解析

### 1. `-o`：指定输出文件名

如果不带这个选项，GCC 默认生成可执行文件 `a.out`。

- 指令：`gcc main.c -o my_app`
- 演示结果：

```bash
$ ls
main.c
$ gcc main.c -o my_app
$ ls
main.c  my_app
```

### 2. `-c`：只编译不链接

将 `.c` 文件编译成 `.o` 目标文件，适合多文件模块化编译。

- 指令：`gcc -c main.c`
- 演示结果：

```bash
$ gcc -c main.c
$ ls
main.c  main.o
```

### 3. `-g`：生成调试信息

为了使用 GDB 断点调试，编译时通常要加 `-g`。

- 指令：`gcc -g main.c -o main`
- 演示结果：

```bash
$ gcc -g main.c -o main
$ gdb ./main
```

### 4. `-O` 和 `-O2`：代码优化

- `-O` 或 `-O1`：基础优化。
- `-O2`：常用发布优化等级，综合性能较好。

- 指令：`gcc -O2 main.c -o main_fast`
- 提示：优化后调试体验可能下降，开发调试阶段一般优先 `-g`。

## 四、补充常用选项

### 1. `-Wall`：显示常用警告

这是非常重要的选项，可以尽早暴露潜在问题。

- 指令：`gcc -Wall main.c -o main`

### 2. `-I`：指定头文件路径

头文件不在默认搜索路径时使用。

- 指令：`gcc main.c -I ./include -o main`

### 3. `-l` 和 `-L`：链接库文件

- `-l` 指定库名。
- `-L` 指定库搜索目录。

- 指令：`gcc main.c -L ./lib -l m -o main`

### 4. `-D`：定义宏

可在编译时注入宏定义。

- 指令：`gcc -DDEBUG main.c -o main`

## 五、GCC 编译四步

| 阶段 | 常用选项 | 输出文件 | 说明 |
| :--- | :--- | :--- | :--- |
| 预处理 | `-E` | `.i` | 展开宏、处理头文件、去注释 |
| 编译 | `-S` | `.s` | C 代码转汇编 |
| 汇编 | `-c` | `.o` | 汇编转目标文件 |
| 链接 | 无或链接参数 | 可执行文件 | 将目标文件和库链接为程序 |

示例：

```bash
gcc -S main.c
```

## 六、快速建议

下面给出两类最常用场景：STM32 裸机编译、Linux 嵌入式交叉编译。

### 1. STM32（裸机，arm-none-eabi-gcc）

```bash
arm-none-eabi-gcc \
  -mcpu=cortex-m3 -mthumb \
  -O0 -g3 -Wall \
  -ffunction-sections -fdata-sections \
  -DSTM32F103xB -DUSE_HAL_DRIVER \
  -I Inc -I Drivers/CMSIS/Include -I Drivers/STM32F1xx_HAL_Driver/Inc \
  -T STM32F103C8Tx_FLASH.ld \
  Startup/startup_stm32f103xb.s Src/main.c Src/stm32f1xx_it.c \
  -Wl,--gc-sections -Wl,-Map=build/app.map \
  -o build/app.elf
```

参数说明：

- `-mcpu=cortex-m3`：指定目标内核。
- `-mthumb`：生成 Thumb 指令集代码。
- `-O0`：关闭优化，便于调试。
- `-g3`：生成更完整的调试信息。
- `-ffunction-sections -fdata-sections`：把函数和数据放入独立 section，配合链接器做裁剪。
- `-D...`：定义宏（芯片型号、HAL 使能等）。
- `-I ...`：头文件搜索路径。
- `-T xxx.ld`：指定链接脚本。
- `-Wl,--gc-sections`：让链接器丢弃未使用代码。
- `-Wl,-Map=...`：生成 map 文件，便于看内存占用。
- `-o build/app.elf`：指定输出 ELF 文件。

### 2. Linux 嵌入式（交叉编译，arm-linux-gnueabihf-gcc）

```bash
arm-linux-gnueabihf-gcc \
  -O2 -g -Wall -Wextra \
  -I include \
  -L lib -Wl,-rpath,'$ORIGIN/lib' \
  src/main.c src/net.c src/uart.c \
  -lpthread -lm \
  -o app
```

参数说明：

- `arm-linux-gnueabihf-gcc`：面向 ARM Linux（hard-float ABI）的交叉编译器。
- `-O2`：发布常用优化等级。
- `-g`：保留调试符号，便于远程调试。
- `-Wall -Wextra`：开启更多告警。
- `-I include`：头文件路径（也可写 `-I ./include`）。
- `-L lib`：库搜索路径。
- `-Wl,-rpath,'$ORIGIN/lib'`：写入运行时库搜索路径。
- `-lpthread -lm`：链接线程库和数学库。
- `-o app`：输出可执行文件名。
