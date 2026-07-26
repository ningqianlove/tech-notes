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

3.1 作用

1. 实现了多核系统中只启用一个核心（hart 0 ）的功能

2. 常用于嵌入式或引导程序中，确保系统在单核模式下完成初始化后再启用其他核心

3.2 代码实现

> https://github.com/riscv-software-src/riscv-tests
>
> riscv-tests/env/p/riscv_test.h

```C
#define RISCV_MULTICORE_DISABLE                                         
  csrr a0, mhartid;                                                     
  1: bnez a0, 1b
```

- 读取当前硬件线程hart的ID，存入寄存器a0；如果a0不为0，则跳转到标号1处，形成一个死循环，使该核心永远自选等待