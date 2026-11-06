# Lab3 实验报告：进程管理与调度

## 一、实验目标

在第五章进程管理的基础上完成：

1. 实现 `sys_spawn`：从指定 ELF 创建子进程并返回其 PID，不替换当前进程；
2. 实现 `sys_waitpid` 对子进程退出状态的回收（等待任意子进程和等待指定 PID）；
3. 实现 `sys_set_priority`，并为进程增加按优先级调度的 stride 调度算法；
4. 保留前两章的系统调用、地址空间与 mmap/munmap 功能。

## 二、实现思路

### 1. spawn

`TaskControlBlock` 增加 `spawn` 方法：用 `MemorySet::from_elf` 为新进程建立独立地址空间、内核栈和 TrapContext，然后把新 TCB 的父进程设为当前进程，并把新 TCB 放入父进程的 `children` 列表。与 `fork + exec` 不同，`spawn` 不会复制父进程的地址空间，也不会改变父进程的执行流：

```rust
let child = parent.spawn(data);
add_task(child);
pid as isize
```

### 2. 优先级与 stride 调度

`TaskControlBlockInner` 增加：

```rust
pub stride: usize,  // 累计调度步长
pub prio: usize,    // 优先级，至少为 2
```

任务创建和 fork 时 `stride` 初始为 0、`prio` 初始为 16。调度器每次从就绪队列选择 `stride` 最小的任务，并在取出后给它增加 `BIG_STRIDE / prio`；`sys_set_priority` 只接受大于 1 的优先级，成功时返回传入值，否则返回 -1。

### 3. wait 与退出

进程退出时把自身状态设为 Zombie、记录退出码，并把子进程迁移到 `INITPROC` 下。`sys_waitpid` 在当前进程的子进程中查找目标：不存在返回 -1；存在但未退出返回 -2；找到 Zombie 子进程时将其从 `children` 移除，把退出码写回用户地址并返回 PID。`wait` 和 `waitpid` 的用户库封装在遇到 -2 时会主动 `yield`，因此不需要内核阻塞等待。

## 三、测试内容与结果

执行官方检查器：

```sh
make -C ci-user test CHAPTER=5
```

结果：`Test passed: 15/15`，包括 `ch5_spawn0`（40 个子进程）、`ch5_spawn1`（wait/waitpid 退出码）、`ch5_setprio`、以及全部 ch4 地址空间测例。`ch5_usertest` 还会运行 `ch5_stride`，6 个不同优先级的进程输出计数比值稳定在约 4.0×10^6，说明 stride 调度按优先级公平分配了 CPU 时间。

## 四、总结

本实验掌握了进程创建、父子关系和僵尸进程回收的完整流程。stride 调度通过优先级换算步长，实现简单且能保证长期公平性；`spawn` 与 `fork` 的差异也体现了“创建新程序”和“创建当前进程副本”在地址空间处理上的本质不同。
