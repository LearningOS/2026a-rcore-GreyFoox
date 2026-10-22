# rCore 2026a · Lab1 实验报告

## 一、功能总结（200 字以内）

在 ch3 中实现 `sys_trace`（系统调用号 410）的三个 request：request 0 读取用户地址处一个字节，request 1 向用户地址写入一个字节，request 2 查询当前任务对指定系统调用的调用次数；其他 request 返回 -1。为每个任务在 `TaskControlBlock` 中维护 `syscall_count: [u32; MAX_SYSCALL_NUM]`，在 `syscall()` 入口统一计数，因此 `sys_trace` 自身也会被计入。

## 二、问答题（trap.S）

### 1. 刚进入 `__restore` 时，`sp` 代表什么值？`__restore` 有哪两种使用情景？

刚进入 `__restore` 时，`sp` 指向内核栈上的一个 `TrapContext`。它有两种使用情景：

- **第一次进入用户态**：`TaskContext::goto_restore(init_app_cx(i))` 把 `ra` 设为 `__restore`，`sp` 指向 `init_app_cx` 在启动时构造好的 TrapContext；`__switch` 返回到 `__restore` 后，从这里恢复并进入 U 态。
- **trap 处理完返回**：`__alltraps` 在当前内核栈上保存了一个 TrapContext，`trap_handler` 返回后落入 `__restore`，`sp` 指向这个刚保存的 TrapContext。

### 2. L43–L48 特殊处理了哪些寄存器？这些值对进入用户态有什么意义？

这三行分别从 TrapContext 取出 `sstatus`、`sepc` 和用户 `sp`，再写回对应 CSR：

- `sstatus`：`sret` 依据它的 `SPP` 决定返回 U 态还是 S 态，`SPIE`/`SIE` 控制返回后的中断使能。
- `sepc`：`sret` 返回到的 pc，即用户程序被 trap 打断的位置。
- `sscratch`：保存用户栈指针，下一次 trap 进入 `__alltraps` 时用它换回用户栈。

### 3. L50–L56 为什么跳过 x2 和 x4？

`x2` 是 `sp`，它的值不从 TrapContext 里恢复，而是在最后 `csrrw sp, sscratch, sp` 时从 `sscratch` 换回来；如果这里也 `ld x2`，会和后面的交换冲突。`x4` 是 `tp`，用户程序不使用它，内核继续保留自己的 `tp`，所以也不恢复。

### 4. `__restore` 最后一条 `csrrw sp, sscratch, sp` 之后，`sp` 和 `sscratch` 各是什么？

`sp` 指向用户栈，`sscratch` 指回内核栈顶。紧接着的 `sret` 用 `sepc` 和 `sstatus` 回到 U 态。

### 5. `__restore` 中状态切换发生在哪条指令？为什么执行后进入用户态？

状态切换发生在 `sret`。它把 `pc` 设为 `sepc`，按 `sstatus.SPP` 设置特权级（这里 `SPP=0`，回到 U 态），并把 `SIE` 恢复为 `SPIE`；因此执行完 `sret` 就进入用户态。

### 6. `__alltraps` 的 `csrrw sp, sscratch, sp` 之后，`sp` 和 `sscratch` 各是什么？

`sp` 指向内核栈，`sscratch` 保存用户栈指针；这样内核不会继续使用用户栈。

### 7. 从 U 态进入 S 态是哪条指令？

用户程序执行 `ecall` 时，硬件把当前 pc 存入 `sepc`、cause 存入 `scause`，并跳到 `stvec` 指向的 `__alltraps`，从 U 态进入 S 态。

## 三、荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与以下各位就以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：
   《无》
2. 此外，我也参考了以下资料，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：
   《无》
3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。
   我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。
4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。
   我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。
   我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。
   我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。

## 四、实验反馈（可选）

暂无。
