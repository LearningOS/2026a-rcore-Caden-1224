# Lab1 实验报告：系统调用与 sys_trace

## 一、实验目标

在第三章多道程序与分时多任务内核的基础上，实现 `sys_trace` 系统调用，使应用程序能够：

1. 读取当前任务用户地址空间中的一个字节；
2. 向当前任务用户地址空间写入一个字节；
3. 查询当前任务对指定系统调用的调用次数。

同时要求系统调用的统计包含 `sys_trace` 自身，且退出系统调用、切换任务的调用也不能漏计。

## 二、实现思路

### 1. 每个任务独立维护系统调用计数

在 `TaskControlBlock` 中增加数组：

```rust
pub syscall_counts: [usize; MAX_SYSCALL_NUM],
```

`MAX_SYSCALL_NUM` 取 411，覆盖当前内核使用的最大系统调用号 410。每个任务在创建时把数组初始化为 0，任务之间互不影响。

### 2. 在系统调用总入口统一计数

在 `syscall()` 分发之前调用 `record_current_syscall(syscall_id)`：

```rust
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    record_current_syscall(syscall_id);
    match syscall_id { ... }
}
```

这样无论系统调用是否会发生任务切换（如 `yield`、`exit`），也不管返回值是什么，都只计数一次；`sys_trace` 的查询请求本身也会被计入。

### 3. 实现 sys_trace 的三种请求

```rust
match trace_request {
    0 => unsafe { (id as *const u8).read_volatile() as isize },
    1 => {
        unsafe { (id as *mut u8).write_volatile(data as u8) };
        0
    }
    2 => current_syscall_count(id) as isize,
    _ => -1,
}
```

第三章尚未启用虚拟内存，用户地址与物理地址一致，因此可以直接按用户传入的地址访问内存。读取使用 `read_volatile`，写入只写 `data` 的最低一个字节，避免编译器优化掉有意义的访问。

## 三、实验过程与问题

- 最初把计数放在 `sys_trace` 内部，导致查询自身的计数不符合测试预期。改为在系统调用总入口计数后，测试中 `count_syscall(SYSCALL_TRACE)` 的返回值与调用次数完全一致。
- 数组下标就是系统调用号，查询一个尚未调用过的系统调用号时自然返回 0，不需要额外初始化逻辑。

## 四、测试结果

执行官方检查器：

```sh
make -C ci-user test CHAPTER=3
```

输出 `Test passed: 7/7`，所有测例通过：

- `ch3_sleep`、`ch3_sleep1`：时间获取与休眠功能正常；
- `ch3_trace`：内存读写、系统调用计数、非法地址处理均符合要求。

## 五、总结

本实验通过在 TCB 中保存系统调用计数并在系统调用入口统一计数，实现了一个简单但完整的 trace 机制。实践中体会到系统调用入口是统一做统计的最佳位置，也理解了内核直接访问用户地址时 `volatile` 语义的重要性。
