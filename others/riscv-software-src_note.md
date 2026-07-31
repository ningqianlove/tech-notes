# 1. INIT_PMP

## 1.1 作用

1. 系统上电或刚进入M模式时执行
2. 由于某些RISC-V实现中PMP默认可能禁止所有访问，这段代码会**抢先通过条目0打开对所有内存的完全访问权限**

## 1.2 代码实现

实现代码如下：

> https://github.com/riscv-software-src/riscv-tests
>
> riscv-tests/env/p/riscv_test.h

```c
#define INIT_PMP
  la t0,1f;
  csrw mtvec,t0;
  /* Set up a PMP to permit all accesses */                             
  li t0, (1 << (31 + (__riscv_xlen / 64) * (53 - 31))) - 1;             
  csrw pmpaddr0, t0;                                                    
  li t0, PMP_NAPOT | PMP_R | PMP_W | PMP_X;                             
  csrw pmpcfg0, t0;                                                     
  .align 2;   
1:

```

**代码释义:**

- 若 `__riscv_xlen` = 32（RV32），pmpaddr =  `0x7FFFFFFF`；若 `__riscv_xlen` = 64（RV64），pmpaddr =  `0x1FFFFFFFFFFFFF`（低 53 位全 1）。

  > [!NOTE]
  >
  > 1. NAPOT模式下匹配的物理内存区域大小
  >
  >    Size = 2^(n+3)字节，此处，RV32时n=31，表示16GB; RV64时n=53,表示72PB
  >
  > 2. RV32的物理地址空间最大为2^32字节（4GB）；RV64的物理地址空间最大为2 ^ 64字节（16EB）
  >
  > 3. 在RISC-V规范中，当PMP区域大小超出机器实现的物理地址位宽时，该区域会被视为**覆盖全部可寻址物理地址空间**（即命中0x00000000~0xffffffff）
  >
  > 4. 对于RV64，虽然它不覆盖完整的64位空间（16EB）,但绝大多数RISC-V硬件平台的物理地址位宽都小于56位

- 硬件代码解析该pmpaddr时，会由于NAPOT模式下的对齐约束与地址位宽限制共同作用导致解析出来的**基地址为0**

- 将条目0设置为NAPOT模式，并赋予读、写、执行全部权限

- `.align 2` 确保后续代码（即陷阱处理程序 `1:`）在 **4 字节边界** 上对齐，满足 `mtvec` 在直接模式下对 BASE 地址最低两位为 0 的对齐要求

- 当发生异常或中断时，CPU 跳转到标签 `1:` 处的处理程序入口

# 2. INIT_XREG

## 2.1 作用

清零所有通用寄存器

## 2.2 代码实现

> https://github.com/riscv-software-src/riscv-tests
>
> riscv-tests/env/p/riscv_test.h

```c
#define INIT_XREG                                                       
  li x1, 0;                                                             
  li x2, 0;                                                             
  li x3, 0;                                                             
  li x4, 0;                                                             
  li x5, 0;                                                             
  li x6, 0;                                                             
  li x7, 0;                                                             
  li x8, 0;                                                             
  li x9, 0;                                                             
  li x10, 0;                                                            
  li x11, 0;                                                            
  li x12, 0;                                                            
  li x13, 0;                                                            
  li x14, 0;                                                            
  li x15, 0;                                                            
  li x16, 0;                                                            
  li x17, 0;                                                            
  li x18, 0;                                                            
  li x19, 0;                                                            
  li x20, 0;                                                            
  li x21, 0;                                                            
  li x22, 0;                                                            
  li x23, 0;                                                            
  li x24, 0;                                                            
  li x25, 0;                                                            
  li x26, 0;                                                            
  li x27, 0;                                                            
  li x28, 0;                                                            
  li x29, 0;                                                            
  li x30, 0;                                                            
```



# 3. RISCV_MULTICORE_DISABLE

## 3.1 作用

1. 实现了多核系统中只启用一个核心（hart 0 ）的功能

2. 常用于嵌入式或引导程序中，确保系统在单核模式下完成初始化后再启用其他核心

## 3.2 代码实现

> https://github.com/riscv-software-src/riscv-tests
>
> riscv-tests/env/p/riscv_test.h

```C
#define RISCV_MULTICORE_DISABLE                                         
  csrr a0, mhartid;                                                     
  1: bnez a0, 1b
```

- 读取当前硬件线程hart的ID，存入寄存器a0；如果a0不为0，则跳转到标号1处，形成一个死循环，使该核心永远自旋等待

# 4. INIT_SATP

## 4.1 作用

初始化时要关闭页表地址转换，使处理器进入“物理地址模式”

1. **环境还没有设置页表**
   - 还没有建立有效的页表数据结构，也没有设置页表基址。
   - 如果提前开启地址翻译，后续的指令或数据访问会变成“虚拟地址访问”，可能会访问错误地址或产生异常。
2. **简化启动逻辑**
   - 测试启动阶段通常直接使用物理地址来加载代码和数据。
   - 关闭翻译后，所有后续内存访问都按物理地址解释，行为可预测。
3. **避免早期异常复杂性**
   - 启动时尚未初始化异常/中断处理环境，若翻译出错会导致难以处理的页表异常。

## 4.2 代码实现

```C
#define INIT_SATP                                                      
  la t0, 1f;                                                            
  csrw mtvec, t0;                                                       
  csrwi satp, 0;                                                       
  .align 2;                                                             
1:
```

# 5. DELEGATE_NO_TRAPS

5.1 作用

用于在M模式启动阶段关闭异常/中断委托

5.2 代码实现

```c
#define DELEGATE_NO_TRAPS                                               
  csrwi mie, 0;                                                         
  la t0, 1f;                                                            
  csrw mtvec, t0;                                                       
  csrwi medeleg, 0;                                                     
  csrwi mideleg, 0;                                                     
  .align 2;                                                             
1:
```

- 禁用机器中断
- 禁止异常/中断委托给 supervis

# 弱符号的使用

弱符号，比如.weak aa , 本身是弱定义，链接阶段：

- 弱存在**强符号（普通全局符号，无.weak修饰）同名aa ——》强符号直接覆盖弱符号，全程优先使用强符号的地址与内容
- 全程无强符号、只有多处弱符号——》链接器任选其中一份弱定义
- 只有一份弱符号——》正常使用该弱符号

## 两种文件场景覆盖写法

**场景1：同一汇编文件内，弱定义+下方强定义覆盖**

```assembly
; 先定义弱符号aa（默认实现）
.weak aa
aa:
    mov $0x1111, %eax
    ret

; 普通全局标签 = 强符号，同名aa，直接覆盖上方弱aa
.global aa
aa:
    mov $0x2222, %eax
    ret
```

链接结果：代码里所有引用aa都会指向0x2222版本，弱定义被丢弃

**场景2：分文件（推荐工程用法）**

**文件A:默认实现，使用弱符号（库文件通用默认逻辑）**

weak_impl.S

```assembly
    .section .text
    .weak aa        ; 声明aa为弱全局符号
aa:
    # 默认函数实现
    xor %eax, %eax
    ret
```

**文件B:业务自定义实现，普通强全局符号（覆盖默认弱符号）**

user_impl.S

```assembly
    .section .text
    .global aa      ; 强符号
aa:
    # 自己重写的逻辑，覆盖库里面的弱aa
    mov $99, %eax
    ret
```

运行后调用aa执行的一定是user_impl.S里的自定义代码

## C语言与汇编互通覆盖

这是最常用场景

很多时候弱符号写在汇编，覆盖代码写C，或是反过来：

汇编侧：

```assembly
.weak aa
aa:
    ret
```

C侧：

```c
void aa(void)
{
    // 覆盖后的自定义逻辑
}
```

C默认导出强符号，链接后C函数自动覆盖汇编弱符号。

**必须是全局符号，不能用static修饰**;不能用`__attribute__((noinline)) / inline / __attribute__((always_inline))`修饰

### 特殊场景

**只想覆盖弱符号，但不想这个函数被其他文件调用污染全局命名**

GCC提供`__attribute__((visibility("hidden")))`

```c
__attribute__((visibility("hidden"))) void aa(void)
{
    // 覆盖弱符号的实现
}
```

效果：

1. 符号依旧是**全局符号**，链接器可以匹配并覆盖 weak 符号；
2. 符号对外动态链接不可见，不会暴露给其他模块调用，兼顾覆盖 + 隔离

# 特权级间切换示例代码

M→S→M

```c
// M 模式异常处理函数（由汇编入口调用）
void m_trap_handler_c(void) {
    unsigned long cause = csr_read(mcause);
    
    // 判断是否为来自 S 模式的 ecall（cause = 9）
    if (cause == 9) {
        // 恢复返回地址到 mepc
        csr_write(mepc, (unsigned long)return_pc);

        // 设置 mstatus 使 mret 返回后进入 M 模式
        unsigned long mstatus = csr_read(mstatus);
        mstatus &= ~(3UL << 11);   // 清除 MPP
        mstatus |= (3UL << 11);    // MPP = 11 (M模式)
        csr_write(mstatus, mstatus);

        // 执行 mret，跳转到 return_pc（即 after_aa）
        __asm__ volatile ("mret");
    } else {
        // 其他异常处理（可扩展）
        // 此处简单死循环
        while(1);
    }
}
// 从 S 模式返回后继续执行的函数
void after_switch(void) {
    // 这里放原来 switch_to_s_mode 位置后面的代码
    // 例如打印信息、继续其他任务等
    // 测试时可以写一个死循环
    while(1) {
        // 做一些工作
    }
}

// M 模式启动入口（类似于 _start）
void start_m_mode(void) {
    // 1. 设置 mtvec 指向汇编入口
    extern void trap_entry(void);   // 在 trap_entry.S 中定义
    csr_write(mtvec, (unsigned long)trap_entry);

    ...
    ...
        
    // 2. 保存返回地址（aa 之后的位置）
    return_pc = (void*)after_switch;

    // 3. 切换到 S 模式
    switch_to_s_mode();

    // 4. 理论上不会执行到这里，因为 mret 会跳转
    //工业级代码（如 FreeRTOS、OpenSBI）的标准写法
    while(1);
}
```

trap_entry.S

```n
.section .text
.globl trap_entry
trap_entry:
    # 保存可能被C函数修改的寄存器（根据需要可扩展）
    # 例如保存 ra, t0-t6 等，此处仅保存 ra 示例
    addi sp, sp, -4
    sw ra, 0(sp)

    # 调用 C 处理函数
    call m_trap_handler_c

    # 如果处理函数没有执行 mret（例如其他异常），则恢复并返回
    lw ra, 0(sp)
    addi sp, sp, 4
    mret
```

