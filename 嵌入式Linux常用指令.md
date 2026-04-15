# 嵌入式 Linux 常用指令整理

> 说明：
>
> - 本文以 Ubuntu/Debian 常见用法为主，嵌入式 Linux、BusyBox 或精简发行版中，部分选项可能略有差异。
> - 涉及用户、网络、重启、关机等操作时，通常需要 `root` 权限，可结合 `sudo` 使用。
> - 原始内容中的口语化说明、截图引用和无关上下文已移除，保留命令知识点并补充了常用示例。

## 一、文件与目录操作

### `ls`

**作用**  
查看目录内容，列出指定路径下的文件和子目录。

**基本语法**

```bash
ls [选项] [路径]
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-a` | 显示所有文件，包括隐藏文件 |
| `-l` | 以长格式显示详细信息 |
| `-h` | 配合 `-l` 使用，以更易读的方式显示大小 |
| `-t` | 按时间排序 |
| `-A` | 显示隐藏文件，但不包含 `.` 和 `..` |
| `-R` | 递归显示子目录内容 |

**示例**

```bash
ls
ls -al
ls -lh /etc
ls -lt
ls -R /home
```

**补充说明**

- `ls -al` 是最常见的组合写法。
- 查看文件大小时，通常使用 `ls -lh` 更直观。

### `cd`

**作用**  
切换当前工作目录。

**基本语法**

```bash
cd [路径]
```

**常见路径写法**

| 写法 | 说明 |
| --- | --- |
| `cd /` | 进入根目录 |
| `cd /usr` | 进入指定绝对路径 |
| `cd ..` | 返回上一级目录 |
| `cd ~` | 进入当前用户主目录 |
| `cd -` | 切换到上一次所在目录 |
| `cd .` | 当前目录，通常用于脚本或组合命令 |

**示例**

```bash
cd /
cd /usr
cd ..
cd ~
cd -
```

**补充说明**

- 路径可分为绝对路径和相对路径。
- 如果目录名包含空格，建议使用引号，例如 `cd "my dir"`。

### `pwd`

**作用**  
显示当前工作目录的绝对路径。

**基本语法**

```bash
pwd [选项]
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-L` | 显示逻辑路径，保留符号链接 |
| `-P` | 显示物理路径，解析符号链接 |

**示例**

```bash
pwd
pwd -P
```

### `cat`

**作用**  
查看文件内容，也可用于拼接多个文件内容。

**基本语法**

```bash
cat [选项] [文件]
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-n` | 对所有输出行编号 |
| `-b` | 对非空行编号 |
| `-s` | 将连续空行压缩为一行 |
| `-A` | 显示不可见字符，如制表符和行尾符 |

**示例**

```bash
cat /etc/environment
cat -n test.txt
cat -b test.txt
cat -s test.txt
cat file1.txt file2.txt
```

**补充说明**

- 文件较长时，通常配合 `less` 或 `more` 更方便阅读。
- 如果只是快速查看几行内容，也可考虑使用 `head` 或 `tail`。

## 二、系统信息与帮助

### `uname`

**作用**  
查看内核和系统相关信息。

**基本语法**

```bash
uname [选项]
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-a` | 显示全部信息 |
| `-r` | 显示内核版本号 |
| `-s` | 显示内核名称 |
| `-m` | 显示硬件架构 |
| `-n` | 显示主机名 |
| `-o` | 显示操作系统名称 |

**示例**

```bash
uname
uname -r
uname -m
uname -a
```

### `clear`

**作用**  
清空当前终端显示内容，只保留命令提示符。

**基本语法**

```bash
clear
```

**示例**

```bash
clear
```

**补充说明**

- 在很多终端中，`Ctrl + L` 的效果与 `clear` 类似。

### `man`

**作用**  
查看命令的帮助文档和手册页。

**基本语法**

```bash
man [命令名]
```

**示例**

```bash
man ls
man ifconfig
man sudo
```

**常用操作**

| 操作 | 说明 |
| --- | --- |
| `/关键字` | 向下搜索关键字 |
| `n` | 跳转到下一个匹配项 |
| `N` | 跳转到上一个匹配项 |
| `q` | 退出帮助页 |

## 三、用户与权限管理

### `sudo`

**作用**  
以其他用户身份执行命令，默认以 `root` 身份执行。

**基本语法**

```bash
sudo [选项] 命令
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-h` | 显示帮助信息 |
| `-l` | 列出当前用户可执行的 sudo 命令 |
| `-u 用户名` | 以指定用户身份执行命令 |
| `-i` | 以目标用户的登录环境进入 shell |
| `-s` | 启动一个 shell |
| `-p 提示文本` | 自定义密码提示信息 |

**示例**

```bash
sudo apt update
sudo adduser test
sudo -l
sudo -u nobody whoami
sudo -i
```

**补充说明**

- 输入 sudo 密码时，终端通常不会显示星号，这是正常现象。
- 临时执行单条高权限命令时，优先使用 `sudo 命令`，不要长期停留在 `root` shell。

### `su`

**作用**  
切换用户身份，默认切换到 `root`。

**基本语法**

```bash
su [选项] [用户名]
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-` | 以登录 shell 方式切换，推荐使用 |
| `-c "命令"` | 切换后执行指定命令 |
| `-m` | 保留当前环境变量 |
| `-s SHELL` | 指定要使用的 shell |

**示例**

```bash
su -
su root
su -c "id" root
su - test
```

**补充说明**

- `su -` 比直接 `su` 更常用，因为它会加载目标用户的环境变量。
- 在已具备 sudo 权限的系统中，很多场景下 `sudo -i` 比 `sudo su` 更规范。

### `adduser`

**作用**  
创建新用户。该命令通常需要 `root` 权限。

**基本语法**

```bash
adduser [选项] 用户名
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `--system` | 创建系统用户 |
| `--home DIR` | 指定用户主目录 |
| `--uid ID` | 指定用户 UID |
| `--ingroup GRP` | 指定用户所属组 |
| `--shell SHELL` | 指定登录 shell |
| `--disabled-password` | 创建用户但不设置密码登录 |

**示例**

```bash
sudo adduser test
sudo adduser --home /home/dev01 dev01
sudo adduser --uid 1050 test1
sudo adduser --ingroup developers dev2
sudo adduser --system www-data-test
```

**补充说明**

- `adduser` 常见于 Debian/Ubuntu 系统。
- 在一些发行版中，也可能使用 `useradd`。

### `deluser`

**作用**  
删除用户。该命令通常需要 `root` 权限。

**基本语法**

```bash
deluser [选项] 用户名
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `--system` | 删除系统用户 |
| `--remove-home` | 同时删除用户主目录 |
| `--remove-all-files` | 删除该用户相关的所有文件 |
| `--backup` | 删除前备份用户文件 |
| `--backup-to DIR` | 指定备份目录 |

**示例**

```bash
sudo deluser test
sudo deluser --remove-home test
sudo deluser --remove-all-files test
sudo deluser --backup --backup-to /tmp/user-backup test
```

**补充说明**

- 执行 `--remove-all-files` 前要特别谨慎，避免误删共享文件。

## 四、网络配置与查看

### `ifconfig`

**作用**  
查看或临时配置网络接口信息，如 IP 地址、子网掩码、启用或关闭网卡。

**基本语法**

```bash
ifconfig [网络接口]
ifconfig [网络接口] [参数]
```

**常见参数**

| 参数 | 说明 |
| --- | --- |
| `up` | 启用网络接口 |
| `down` | 关闭网络接口 |
| `netmask` | 设置子网掩码 |
| `broadcast` | 设置广播地址 |
| `promisc` | 启用混杂模式 |

**示例**

```bash
ifconfig
ifconfig eth0
sudo ifconfig eth0 up
sudo ifconfig eth0 down
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0
sudo ifconfig ens33 192.168.31.20 netmask 255.255.255.0
```

**补充说明**

- `ifconfig` 的配置通常是临时生效，重启后可能失效。
- 在较新的 Linux 发行版中，`ifconfig` 可能默认未安装，常见替代命令是 `ip addr`、`ip link`。
- 嵌入式 Linux、开发板或旧系统中，`ifconfig` 仍然很常见。

## 五、系统控制

### `reboot`

**作用**  
重启系统。

**基本语法**

```bash
reboot
```

**示例**

```bash
sudo reboot
```

**补充说明**

- 一般需要管理员权限。
- 执行前应确认重要数据已保存。

### `poweroff`

**作用**  
关闭系统电源。

**基本语法**

```bash
poweroff
```

**示例**

```bash
sudo poweroff
```

**补充说明**

- 在服务器或开发板环境中，关机前应先确认没有正在写入的数据或关键任务。

## 六、软件安装相关

> 原始内容中的“install 命令”更接近“软件安装”这个主题。为避免混淆，这里拆分为更常见、更准确的两类：包管理安装和源码安装。

### `apt install`

**作用**  
在 Ubuntu/Debian 系系统中安装软件包。

**基本语法**

```bash
sudo apt update
sudo apt install [选项] 软件包名
sudo apt-get install 软件包名
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-y` | 自动确认安装 |
| `--reinstall` | 重新安装已安装的软件包 |
| `--no-install-recommends` | 不安装推荐依赖 |

**示例**

```bash
sudo apt update
sudo apt install git
sudo apt install -y vim
sudo apt-get install minicom
sudo apt install --reinstall openssh-client
sudo apt install --no-install-recommends build-essential
```

**补充说明**

- `apt` 更适合日常手工使用，`apt-get` 更常见于脚本或旧资料中。
- 如果是第一次安装某个软件，通常先执行 `sudo apt update` 再执行安装命令。

### `apt update`

**作用**  
更新本地软件包索引列表，但不会直接升级已安装的软件。

**基本语法**

```bash
sudo apt update
sudo apt-get update
```

**示例**

```bash
sudo apt update
sudo apt-get update
```

**补充说明**

- 该命令会访问软件源并刷新本地可安装软件列表。
- 安装新软件或升级软件前，通常先执行一次 `apt update`。

### `apt upgrade`

**作用**  
升级系统中已安装的软件包到可用的新版本。

**基本语法**

```bash
sudo apt upgrade [选项]
sudo apt-get upgrade [选项]
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-y` | 自动确认升级 |
| `--only-upgrade` | 仅升级已安装的软件包，不安装新包 |

**示例**

```bash
sudo apt upgrade
sudo apt upgrade -y
sudo apt-get upgrade
sudo apt install --only-upgrade openssh-client
```

**补充说明**

- `apt upgrade` 通常用于常规升级。
- 如果只想升级某个已安装的软件包，可使用 `apt install --only-upgrade 软件包名`。

### `apt remove`

**作用**  
卸载已安装的软件包。

**基本语法**

```bash
sudo apt remove [选项] 软件包名
sudo apt-get remove 软件包名
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-y` | 自动确认卸载 |
| `--purge` | 卸载软件并删除配置文件 |

**示例**

```bash
sudo apt remove minicom
sudo apt remove -y minicom
sudo apt remove --purge minicom
sudo apt-get remove minicom
```

**补充说明**

- 普通 `remove` 一般会保留部分配置文件。
- 如果希望清理无用依赖，可额外执行 `sudo apt autoremove`。

### `apt-get check`

**作用**  
检查软件包依赖关系是否完整，并尝试发现损坏的依赖状态。

**基本语法**

```bash
sudo apt-get check
```

**示例**

```bash
sudo apt-get check
```

**补充说明**

- 当安装或卸载软件后出现依赖异常时，可以先执行该命令排查。
- 该命令主要用于检查，不负责完整的软件升级流程。

### `make install`

**作用**  
将源码编译后的程序、库或头文件安装到系统目录中。

**基本语法**

```bash
make
sudo make install
```

**示例**

```bash
./configure
make
sudo make install
```

**补充说明**

- `make install` 常见于源码编译安装流程。
- 是否支持卸载，要看源码包是否提供 `make uninstall`。
- 在嵌入式开发中，也常见交叉编译后手动拷贝产物到目标文件系统。

### `install`

**作用**  
复制文件到目标位置，并可同时设置权限、属主、属组。它本身不是常规的软件包安装命令。

**基本语法**

```bash
install [选项] 源文件 目标路径
install -d [选项] 目录
```

**常用选项**

| 选项 | 说明 |
| --- | --- |
| `-d` | 创建目录 |
| `-m MODE` | 设置权限 |
| `-o OWNER` | 设置属主 |
| `-g GROUP` | 设置属组 |

**示例**

```bash
install app /usr/local/bin/app
install -m 755 app /usr/local/bin/app
install -d /usr/local/myapp
sudo install -o root -g root -m 644 config.conf /etc/myapp/config.conf
```

**补充说明**

- 在 Makefile 中，`install` 命令经常用于“复制并设置权限”。
- 如果语境是“安装软件包”，优先写成 `apt install`；如果语境是“源码编译后安装”，优先写成 `make install`。
