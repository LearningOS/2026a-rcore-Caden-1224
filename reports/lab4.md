# Lab4 实验报告：文件系统与硬链接

## 一、实验目标

在第六章文件系统的基础上：

1. 实现 `sys_fstat`，向用户返回文件的设备号、inode 编号、类型和硬链接数；
2. 实现 `sys_linkat`，为已有文件创建硬链接；
3. 实现 `sys_unlinkat`，删除目录项并在最后一个硬链接消失时回收 inode 和数据块；
4. 在 `easy-fs` 中增加 `nlink` 元数据并保持磁盘 inode 的大小约束。

## 二、实现思路

### 1. easy-fs 的 DiskInode 增加 nlink

`DiskInode` 原本占用 128 字节（4 字节 size + 28 个直接块指针 + 间接块 + 类型）。为了在不破坏块内 inode 布局的前提下加入 `nlink: u32`，把 `INODE_DIRECT_COUNT` 从 28 减为 27，正好腾出一个 4 字节字段，`DiskInode::initialize` 时把 `nlink` 置为 1。

### 2. 目录项与 link/unlink

- `Inode::find_dirent` 返回目录项偏移和 inode id；规定 `inode_id == 0` 表示空目录项（根目录占 0 号 inode，普通文件从 1 开始）。
- `append_dirent` 优先复用已删除的空目录项，否则扩展目录大小后写入新目录项。
- `Inode::link` 检查新文件名合法且不存在，然后增加目标文件的 `nlink` 并插入新目录项；目录、已存在名字和非法名字返回错误。
- `Inode::unlink` 找到目录项后把该项写成空目录项，并把 `nlink` 减一；减到 0 时清除文件数据块并释放 inode。
- `Inode::stat` 同时返回 inode id、nlink 和是否为目录，供 `fstat` 使用。

### 3. 系统调用层

`OSInode` 实现 `File::stat`，把 `easy-fs` 的元数据转换为 `Stat`。`sys_fstat` 从 fd 表取出文件，先检查用户 `Stat` 缓冲区所有页面可写，再用 `translated_byte_buffer` 按字节复制，避免跨页和对齐问题。`sys_linkat`/`sys_unlinkat` 先用 `translated_str` 复制用户路径，再调用 `ROOT_INODE.link/unlink`。

## 三、测试内容与结果

执行官方检查器：

```sh
make -C ci-user test CHAPTER=6
```

结果：`Test passed: 31/31`，其中本章新增测例全部通过：

- `ch6_file0`：文件创建、读写；
- `ch6_file1`：fstat 返回 `FILE` 类型、nlink=1；
- `ch6_file2`：连续 link 后 nlink 为 2、4，unlink 后依次变为 3、1，且 inode 号保持不变；
- `ch6_file3`：10 次大量写入后 unlink，再次 open 失败；
- 同时通过 ch2b-ch5 的基础与实验测例。

## 四、总结

本实验理解了 easy-fs 的磁盘 inode、目录项和位图布局。关键点是 `nlink` 必须持久化在 DiskInode 中，并且不能破坏 128 字节的 inode 槽位；删除目录项时只把 inode id 置 0 作为空标记，既保持目录大小不变又实现了目录项复用。
