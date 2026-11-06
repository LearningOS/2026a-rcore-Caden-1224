# Lab5 实验报告：并发与同步

## 一、实验目标

在第八章并发与同步的基础上：

1. 支持同一进程内的多线程创建、退出与等待；
2. 实现自旋锁互斥量、阻塞互斥量、信号量与条件变量的正确并发语义；
3. 实现互斥量和信号量的死锁检测：在开启检测后，对可能造成系统不安全的加锁/下操作返回 `-0xDEAD`。

## 二、实现思路

### 1. 进程与线程结构

第八章的 `ProcessControlBlock` 保存线程共享的地址空间、文件描述符表、互斥量列表、信号量列表和条件变量列表；每个 `TaskControlBlock` 通过 `process: Weak<ProcessControlBlock>` 指回所属进程，并用 `TaskUserRes` 保存线程 id。`thread_create` 在同一个进程内创建线程，`waittid` 回收线程退出码。

### 2. 同步原语

- `MutexSpin`：持有 `UPSafeCell<bool>`，lock 时循环检查并忙等；
- `MutexBlocking`：维护 `wait_queue: VecDeque<Arc<TaskControlBlock>>`，竞争失败时把当前线程加入等待队列并 `block_current_and_run_next`，unlock 时唤醒队首；
- `Semaphore`：内部计数 `count` 与等待队列，down 在计数不足时阻塞，up 增加计数并唤醒等待者；
- `Condvar`：等待时把当前线程挂起并释放关联互斥量，signal 唤醒等待者。

### 3. 死锁检测

为每个进程增加两个 `DeadlockDetector`（分别用于 mutex 和 semaphore），其内部维护：

- `available`：每类资源当前可用数量；
- `allocation[tid][rid]`：各线程已占有的资源数量；
- `need[tid]`：线程当前正在等待的资源编号。

`Resource` 封装一个资源的编号和所属检测器。线程申请资源前先调用 `request()` 记录等待需求；若检测已开启，则运行类似银行家算法的安全性检查：反复寻找 `need <= work` 的线程，将其 `allocation` 归还到 `work`（模拟线程完成并释放资源），若最终仍有线程无法完成，则判定为不安全，撤销本次请求并返回 `-0xDEAD`。获取资源成功后 `acquire` 更新分配表，释放时 `release` 归还。

`sys_enable_deadlock_detect` 支持 0/1 开关；开启前会先对当前状态做一次安全检查，非法参数返回 -1。

## 三、测试内容与结果

执行官方检查器：

```sh
make -C ci-user test CHAPTER=8
```

结果：`Test passed: 25/25`，包括：

- `ch8b_threads`、`ch8b_threads_arg`：多线程创建、参数传递和退出码回收；
- `ch8b_mpsc_sem`：4 生产者 1 消费者的信号量同步；
- `ch8b_phil_din_mutex`：哲学家就餐；
- `ch8b_race_adder_mutex_spin`：自旋锁保护共享计数器；
- `ch8b_sync_sem`、`ch8b_test_condvar`：信号量与条件变量同步；
- `ch8_deadlock_mutex1`、`ch8_deadlock_sem1`、`ch8_deadlock_sem2`：死锁检测；
- `ch7b_pipetest` 与全部基础测例。

## 四、总结

本实验完整实践了线程、互斥量、信号量、条件变量和死锁检测。死锁检测的核心是把“资源分配图”转化为银行家算法的安全性检查；实现时必须注意在检查过程中不能同时持有进程内部结构和同步对象锁，否则容易触发 `BorrowMutError` 或真正的内核死锁。
