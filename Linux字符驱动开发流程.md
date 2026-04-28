# Linux 字符驱动开发流程
## 一、驱动模块加载阶段

1. **加载模块**
	 终端执行 `insmod chrdevbase.ko`，内核模块加载器读取模块文件，将其链接到内核空间。

2. **调用模块入口函数**
	 内核根据 `module_init(chrdevbase_init)` 调用 `chrdevbase_init`（驱动入口函数）。

3. **注册字符设备**
	 `chrdevbase_init` 中执行：
	 ```c
	 retvalue = register_chrdev(200, "chrdevbase", &chrdevbase_fops);
	 ```
	 - `200`：指定主设备号 200（静态分配）。
	 - `"chrdevbase"`：设备名，显示在 `/proc/devices`。
	 - `&chrdevbase_fops`：`struct file_operations` 指针，包含 `.open`、`.read`、`.write`、`.release` 等函数。

4. **内核内部动作**
	 `register_chrdev` 在内核内部完成：
	 - 检查主设备号 200 是否占用。
	 - 将主设备号 200 与 `chrdevbase_fops` 关联，并占用次设备号 0~255（老式接口行为）。
	 - VFS 记住：**主设备号 200 的字符设备统一使用这组 `file_operations`**。

5. **创建设备节点（手动或 udev）**
	 执行：`mknod /dev/chrdevbase c 200 0`。
	 - `c`：字符设备。
	 - `200`：主设备号。
	 - `0`：次设备号。
	 该设备节点是用户程序与驱动之间的“桥梁”。

---

## 二、应用程序调用阶段

### 1. `open("/dev/chrdevbase", O_RDWR)`

- 系统调用进入内核：`open` → `sys_open` → `do_sys_open` → `vfs_open`。
- VFS 查找设备节点：
	- 根据路径 `/dev/chrdevbase` 找到 `inode`。
	- 从 `inode` 中取出 **主设备号=200**、**次设备号=0**。
- 找到驱动操作表：
	- VFS 以内核内部的 `chrdevs` 数组（或现代的 `cdev_map`）为索引。
	- 用主设备号 200 找到注册时的 `file_operations`，即 `chrdevbase_fops`。
- 调用驱动 `open`：
	- 执行 `chrdevbase_fops.open`（`chrdevbase_open`）。
- 返回用户空间：
	- `open` 返回 0 成功，用户拿到 `fd`。

**关键点**：用户层 `open` 能找到驱动函数，是因为 **设备节点的主设备号** 与 **注册时的主设备号** 对应。

### 2. `read(fd, buf, count)`

- `read` → `sys_read` → `vfs_read`。
- VFS 通过 `fd` 找到 `struct file`，其中保存：
	- `f_op` 指针（指向 `chrdevbase_fops`）。
- VFS 调用 `f_op->read`，即 `chrdevbase_read`。
- 驱动内部 `copy_to_user` 将内核数据拷贝到用户缓冲区。
- 驱动应返回实际拷贝的字节数。

### 3. `write(fd, buf, count)`

- `write` → `sys_write` → `vfs_write`。
- VFS 调用 `f_op->write`，即 `chrdevbase_write`。
- 驱动 `copy_from_user` 将用户数据拷贝到内核缓冲区并打印。
- 驱动应返回成功写入的字节数。

### 4. `close(fd)`

- `close` → `sys_close` → `filp_close`。
- VFS 调用 `f_op->release`（`chrdevbase_release`）。
- 驱动清理资源，返回 0。

---

## 三、模块卸载阶段

1. 执行 `rmmod chrdevbase`，内核调用模块退出函数。
2. `module_exit(chrdevbase_exit)` 指定 `chrdevbase_exit`。
3. 执行 `unregister_chrdev(200, "chrdevbase")`。
4. 内核解除主设备号 200 与 `file_operations` 的绑定，释放次设备号范围。
5. 不再需要时可删除 `/dev/chrdevbase`。

---

## 总结：用户层如何找到驱动函数

- **驱动注册时**：`register_chrdev(200, ..., &fops)` 告诉内核“主设备号 200 对应这组 `fops`”。
- **创建设备节点时**：`mknod /dev/xxx c 200 0` 将主设备号写入设备节点 `inode`。
- **用户调用时**：VFS 读取设备节点主设备号 → 查内核表 → 获取 `fops` → 调用驱动函数。

核心链路：**用户层 open/read/write → VFS → 主设备号 → file_operations → 驱动函数**。
