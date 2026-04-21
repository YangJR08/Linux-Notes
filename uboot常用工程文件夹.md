# 1. arch文件夹

arch文件夹包含了与处理器架构相关的代码，主要负责处理器的初始化，异常处理，以及与处理器相关的功能实现，如中断控制，时钟管理等。对于不同的处理器架构，arch文件夹下会有不同的子目录，如arm、x86等，每个子目录对应一个处理器架构。如下图：
![arch文件夹示意图](uboot常用工程文件夹/arch文件夹示意图.png)
对于I.MX6ULL开发板，涉及的芯片是基于ARM Cortex-A架构的，因此我们主要关注arch/arm目录下的代码如下图：
![arch/arm目录示意图](uboot常用工程文件夹/arch_arm目录示意图.png)
对于arch/arm目录中的文件，我们根据芯片型号选择对应的文件夹，和在arch/arm/cpu目录下的文件夹，来查看与处理器架构相关的文件，如图下：
![arch/arm/cpu目录示意图](uboot常用工程文件夹/arch_arm_cpu目录示意图.png)
对于i.MX6ULL开发板，我们需要关注arch/arm/cpu/armv7目录下的文件，这些文件包含了与ARMv7架构相关的代码实现，这些是cpu的运行规则；arch/arm下imx-common目录下的文件，这些文件包含了与i.MX6ULL芯片相关的代码实现，这些是芯片的运行规则，列如时钟管理，ddr初始化等。

# 2. board文件夹

board文件夹包含了与具体开发板相关的代码，主要负责开发板的初始化，外设驱动，以及与开发板相关的功能实现，如串口、网络、存储等。对于不同的开发板，board文件夹下会有不同的子目录，每个子目录对应一个开发板。如下图：
![board 文件夹示意图](uboot常用工程文件夹/board_文件夹示意图.png)
使用freescale芯片的板子都放到此文件夹中,I.MX系列以前属于freescale,只是freescale后来被NXP收购了。打开此freescale文件夹,在里面找到和mx6u(I.MX6UL/ULL)有关的文件夹,如图下：
![board_freescale文件夹示意图](uboot常用工程文件夹/board_freescale文件夹示意图.png)
如果是自己画的板子，比如用的I.MX6UL/ULL芯片，那么久可以在这个文件夹中，可以创建新的文件夹，命名为自己的板子名字，在里面编写与自己板子相关的代码实现。

# 3. configs 文件夹

U-Boot 是可配置的，但从头开始逐一配置每个项目会非常麻烦。因此，半导体或开发板厂商通常会制作好配置文件供使用。我们可以在这些现成的配置文件基础上添加自己想要的功能。
这些由半导体厂商或开发板厂商制作好的配置文件统一命名为 `xxx_defconfig`，其中 `xxx` 表示开发板名字。这些 defconfig 文件都存放在 configs 文件夹中，如下图：
![configs 文件夹示意图](uboot常用工程文件夹/configs_文件夹示意图.png)

# 4. 其它重要文件说明

## 4.1 .u-boot.xxx_cmd 文件

这类文件是编译过程中自动生成的命令文件，通常以 `.u-boot.xxx_cmd` 命名。

- 例如：
 	- `.u-boot.bin.cmd` 定义了如何生成 `u-boot.bin`，内容如：
		```
  cmd_u-boot.bin := cp u-boot-nodtb.bin u-boot.bin
  ```
		表示将 `u-boot-nodtb.bin` 拷贝并重命名为 `u-boot.bin`。
	- `.u-boot-nodtb.bin.cmd` 用于生成 `u-boot-nodtb.bin`，内容如：
		```
		cmd_u-boot-nodtb.bin := arm-linux-gnueabihf-objcopy --gap-fill=0xff ... u-boot u-boot-nodtb.bin
		```
		使用 `objcopy` 工具将 ELF 格式的 `u-boot` 转为二进制格式。
	- `.u-boot.cmd` 用于生成 `u-boot` ELF 文件，内容如：
		```
		cmd_u-boot := arm-linux-gnueabihf-ld.bfd ... -o u-boot ...
		```
		通过链接各个 `built-in.o` 文件生成最终的 `u-boot` 可执行文件。
	- `.u-boot.imx.cmd` 用于生成 `u-boot.imx`，内容如：
		```
		cmd_u-boot.imx := ./tools/mkimage -n ... -d u-boot.bin u-boot.imx
		```
		利用 `mkimage` 工具为 `u-boot.bin` 添加头部信息，生成适用于 NXP 平台的 `u-boot.imx`。

## 4.2 u-boot.xxx 文件

这些是最终生成的各类镜像或配置文件，包括：

- `u-boot`：ELF 格式的可执行文件。
- `u-boot.bin`：二进制格式的可执行镜像。
- `u-boot.imx`：为 NXP 平台添加头部信息后的镜像。
- `u-boot.lds`：链接脚本。
- `u-boot.map`：映射文件，记录各函数的链接地址。
- `u-boot.srec`：S-Record 格式镜像。
- `u-boot.sym`：符号表文件。
- `u-boot-nodtb.bin`：未包含设备树的二进制镜像。

## 4.3 .config 文件

U-Boot 的配置文件，由 `make xxx_defconfig` 命令自动生成。
以 `CONFIG_` 开头的变量用于控制功能开关，内容如：
```
CONFIG_CMD_BOOTM=y
```
Makefile 会根据这些变量决定编译哪些功能模块。例如：
```
obj-$(CONFIG_CMD_BOOTM) += bootm.o
```
若 `CONFIG_CMD_BOOTM=y`，则会编译 `bootm.o`。

## 4.4 Makefile 文件

顶层 Makefile 负责整体编译流程，并调用各子目录下的 Makefile 实现模块化管理。
这种嵌套结构有助于大型项目的维护和扩展。

## 4.5 README 文件

详细介绍 U-Boot 的编译方法、各文件夹含义及常用命令，是了解 U-Boot 项目的重要文档。


