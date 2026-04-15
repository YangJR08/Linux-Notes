# Linux 嵌入式开发环境搭建（整理版，含 2026 更新）

本文把原教程内容按“可执行步骤”重排，并在每个步骤中保留旧方法，同时补充 2026 年更推荐的新方法。

## 0. 适用范围与准备

- 主机系统：Ubuntu 20.04/22.04/24.04（建议 22.04 或 24.04）
- 目标方向：ARM32（arm-linux-gnueabihf）/ ARM64（aarch64-linux-gnu）交叉编译
- 典型场景：开发板文件传输、远程登录、内核或应用交叉构建

先更新系统索引：

```bash
sudo apt update
```

---

## 1. 配置 NFS 服务（用于开发板挂载主机目录）

### 1.1 旧方法（原教程思路）

1. 安装 NFS 组件：

```bash
sudo apt-get install nfs-kernel-server rpcbind
```

2. 创建共享目录（示例）：

```bash
mkdir -p ~/linux/nfs
```

3. 编辑配置：

```bash
sudo vi /etc/exports
```

4. 添加一行（按你自己的用户名路径修改）：

```bash
/home/<你的用户名>/linux/nfs *(rw,sync,no_root_squash)
```

5. 重启服务：

```bash
sudo /etc/init.d/nfs-kernel-server restart
```

### 1.2 新方法（2026 推荐）

1. 使用 systemd 管理服务，不再用 init.d：

```bash
sudo apt install -y nfs-kernel-server
sudo systemctl enable --now nfs-kernel-server
```

2. 更安全地限制客户端网段（示例只允许 192.168.1.0/24）：

```bash
sudo mkdir -p /srv/nfs/board
sudo chown -R $USER:$USER /srv/nfs/board
echo "/srv/nfs/board 192.168.1.0/24(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

3. 校验导出：

```bash
sudo exportfs -v
showmount -e localhost
```

4. 如开启防火墙，放行 NFS：

```bash
sudo ufw allow from 192.168.1.0/24 to any port nfs
```

---

## 2. 配置 SSH 服务（用于远程登录和传输）

### 2.1 旧方法（原教程思路）

```bash
sudo apt-get install openssh-server
```

默认配置通常可用，配置文件位于 /etc/ssh/sshd_config。

### 2.2 新方法（2026 推荐）

1. 安装并立即开机自启：

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

2. 检查服务与端口：

```bash
systemctl status ssh --no-pager
ss -tlnp | grep ":22"
```

3. 建议改为密钥登录（比密码更安全）：

```bash
# 在你的客户端执行
ssh-keygen -t ed25519
ssh-copy-id <用户名>@<主机IP>
```

4. 可选安全加固（按需）：

- 关闭 root 远程登录
- 禁止密码登录，仅允许密钥
- 限制允许登录用户

对应配置在 /etc/ssh/sshd_config，修改后执行：

```bash
sudo systemctl restart ssh
```

---

## 3. 安装 ARM 交叉编译工具链

### 3.1 旧方法（原教程思路，Linaro 4.9.4 手动安装）

1. 创建目录并拷贝工具链压缩包：

```bash
sudo mkdir -p /usr/local/arm
sudo cp gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf.tar.xz /usr/local/arm/ -f
```

2. 解压：

```bash
cd /usr/local/arm
sudo tar -vxf gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf.tar.xz
```

3. 配置 PATH（旧方式常写到 /etc/profile）：

```bash
export PATH=$PATH:/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin
```

### 3.2 新方法（2026 推荐）

优先使用 Ubuntu 官方仓库工具链，维护更方便。

1. ARM32（常见于 Cortex-A7/A9 等）：

```bash
sudo apt install -y gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf binutils-arm-linux-gnueabihf
```

2. ARM64（常见于 Cortex-A53/A55 等）：

```bash
sudo apt install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu binutils-aarch64-linux-gnu
```

3. 验证：

```bash
arm-linux-gnueabihf-gcc -v
aarch64-linux-gnu-gcc -v
```

4. 如果项目必须固定版本（例如某些 BSP 强依赖），再用独立工具链并放在 /opt/toolchains，推荐写入 /etc/profile.d/arm-toolchain.sh，而不是直接改 /etc/profile。

### 3.3 关于旧库依赖说明

原教程中的：

```bash
sudo apt-get install lsb-core lib32stdc++6
```

在新系统上通常不是必须，尤其使用仓库版交叉编译器时可不装。只有旧版闭源工具链报缺库时再按错误提示补装。

---

## 4. 交叉编译器快速自检

1. 创建测试文件：

```bash
cat > hello.c << 'EOF'
#include <stdio.h>
int main(void){
	puts("hello arm");
	return 0;
}
EOF
```

2. 编译 ARM32 可执行文件：

```bash
arm-linux-gnueabihf-gcc hello.c -o hello_arm
file hello_arm
```

看到 ELF 且架构为 ARM，表示工具链可用。

---

## 5. 安装 VS Code 与常用扩展

### 5.1 旧方法（原教程思路）

离线安装某个历史 .deb 包：

```bash
sudo dpkg -i code_xxx_amd64.deb
```

### 5.2 新方法（2026 推荐）

建议使用官方仓库安装，后续可随系统更新：

```bash
sudo apt install -y wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /usr/share/keyrings/microsoft.gpg > /dev/null
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update
sudo apt install -y code
```

### 5.3 嵌入式开发推荐扩展（精简版）

- C/C++（ms-vscode.cpptools）
- CMake Tools（ms-vscode.cmake-tools）
- Makefile Tools（ms-vscode.makefile-tools）
- clangd（llvm-vs-code-extensions.vscode-clangd，和 cpptools 二选一做主补全）
- Cortex-Debug（marus25.cortex-debug，做裸机调试时使用）
- DeviceTree（设备树语法高亮）
- Remote - SSH（远程开发板/远程 Linux 主机时使用）
- Chinese (Simplified)（中文界面按需安装）

不建议一次装太多美化或功能重叠插件，以免索引冲突和性能下降。

---

## 6. 一次性验收清单

- NFS 服务已启动，开发板可挂载共享目录
- SSH 可从 Windows/Linux 客户端正常登录
- 交叉编译器命令可直接调用并成功编译测试程序
- VS Code 可正常打开工程并完成代码补全/构建

如果你后续还要做 U-Boot、Linux Kernel、Buildroot/Yocto，我可以继续把这份文档扩展成“从零到可启动系统”的完整流程版。
