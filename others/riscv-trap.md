当异常或中断在M模式发生（或未被委托）

**硬件**自动执行以下操作：

1. mepc  ← 当前/下一条指令的PC
2. mcause ← 异常/中断原因码
3. mtval ← 异常相关信息
4. mstatus.MPIE ← mstatus.MIE
5. mstatus.MIE ← 0（关中断）
6. mstatus.MPP ← 当前特权模式
7. PC ← mtvec

> [!IMPORTANT]
>
>   mtvec 的低 2 位编码了两种模式：
>   - 00 = Direct 模式：所有 trap 跳转到 mtvec.BASE
>   - 01 = Vectored 模式：同步异常跳转到 mtvec.BASE，中断跳转到 mtvec.BASE + 4 × cause

**软件**

1. 检查是否有自定义的trap_handler
2. 检查mcause，异常 or 中断？