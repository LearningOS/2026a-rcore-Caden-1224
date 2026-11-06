# Lab2 实验报告：地址空间与 mmap/munmap

## 一、实验目标

在第四章地址空间的基础上：

1. 重新实现 `sys_get_time`，使之能正确处理跨页的 `TimeVal`；
2. 让 `sys_trace` 在启用虚拟内存后仍能正确读写用户内存；
3. 实现 `sys_mmap` 与 `sys_munmap`，支持按页申请/释放匿名内存，并进行参数校验和权限检查。

## 二、实现思路

### 1. sys_get_time

用户传入的 `TimeVal` 指针可能跨越两个页面。实现时先用用户页表检查整个对象覆盖的每一页都具有 `V | U | W` 权限，只要有一页不可写就直接返回 -1，避免部分写入；检查通过后用 `translated_byte_buffer` 把 `TimeVal` 按字节分段复制到用户空间。这样既不依赖用户指针对齐，也正确处理了跨页情况。

### 2. sys_trace

第四章启用页表后，用户虚拟地址不能直接解引用。新增 `PageTable::translate_user(addr, permission)`：

- 先把 `VirtAddr` 与原始地址互相转换，排除不符合 Sv39 格式的地址；
- 通过页表翻译到末级 PTE，检查 `V | U` 以及请求的 `R` 或 `W` 权限；
- 返回带页内偏移的物理地址。

`sys_trace` 的请求 0/1 分别以 `R`、`W` 权限调用该函数，成功后再读写一个字节；请求 2 仍复用 Lab1 的系统调用计数。对于内核地址、未映射地址以及只读页，均返回 -1。

### 3. sys_mmap

参数校验包括：

- `start` 必须按 `PAGE_SIZE` 对齐；
- `prot` 必须非零且只能包含低 3 位（R/W/X）；
- `start + len` 向上取整到页时检查溢出和 Sv39 地址合法性。

通过校验后，把 `prot` 左移一位映射成页表权限，并加上 `MapPermission::U`，再调用 `MemorySet::mmap`。`MemorySet::mmap` 会先检查目标区间内是否已有有效映射；然后逐页分配物理页并建立映射，如果中途物理页不足则回滚已建立的映射，最后把 `MapArea` 挂到 `areas` 中，以便 fork 时复制和进程退出时回收。

### 4. sys_munmap

`sys_munmap` 要求 `start` 页对齐且区间合法；`MemorySet::munmap` 要求存在一个起始页号和结束页号都完全一致的 Framed 用户 `MapArea`，否则返回 -1。成功时解除页表映射并从 `areas` 中删除该区域，`FrameTracker` 随之释放物理页。

## 三、测试内容与结果

官方检查器执行：

```sh
make -C ci-user test CHAPTER=4
```

结果：`Test passed: 16/16`，包括：

- `ch4_mmap0`：mmap 后读写 4KB 内存；
- `ch4_mmap1` / `ch4_mmap2`：只读/只写页的访问异常被正确触发；
- `ch4_mmap3`：未对齐、重叠、非法权限等错误参数返回 -1；
- `ch4_unmap` / `ch4_unmap2`：解除映射、重新映射及非法 munmap 的行为；
- `ch4_trace1`：trace 在虚拟内存下对普通变量、mmap 页、内核地址和未映射地址的处理；
- 同时保留第三章的 sleep/trace 测例并全部通过。

## 四、总结

本实验加深了对页表翻译、用户权限位和地址空间管理的理解。实现 mmap 时最关键的是先完整校验再写页表，并为分配失败准备回滚路径；实现 trace 时则必须区分“末级 PTE 不存在”和“PTE 存在但无效”两种情况。
