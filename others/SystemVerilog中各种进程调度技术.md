> 《芯片验证全视角经验心得分享》微信公众号

# 引言

SV在继承verilog并行执行特性的基础上，引入更加丰富和精细的并发控制机制，这些机制构成了现代验证方法学（如UVM）的底层基础

**挑战**

- **时序协调问题：**多个并行进程需要在特定时间点进行同步，确保激励生成的顺序性和响应采集的正确性
- **资源共享问题：**多个进程可能需要访问同一资源（如总线接口、寄存器模型等），需要有效的互斥机制避免冲突
- **数据传递问题：**进程间需要高效、可靠地传递数据，实现生产者-消费者等经典模式
- **异常处理问题：**需要能够监控和控制进程的执行状态，处理超时、挂起、终止等异常情况
- **性能优化问题：**在大规模验证环境中，进程调度机制的性能直接影响仿真效率

# fork-join系列

## fork-join

### 基本语法与语义

父进程（主线程）会阻塞，直到所有由fork启动的子进程**全部完成**后，才会继续执行后续代码

```systemverilog
initial begin
    fork
        begin
            // 子进程1            
            #10;
            $display("Process 1 completed");
        end
        begin
            // 子进程2            
            #20;
            $display("Process 2 completed");
        end
    join
    $display("All processes finished at time %0t", $time);
end
```

仿真输出：

```

- 时间10：Process 1 completed
- 时间20：Process 2 completed  
- 时间20：All processes finished
```

### 应用场景

适用于需要**同步**完成多个并行任务的场景

- **并行激励生成**：同时向多个接口施加激励， 并等待所有激励完成后再进行下一阶段操作
- **多通道采集**：需要等待多个监控器完成数据采集的场景
- **初始化序列**：测试平台初始化时，并行执行多个配置任务

### 潜在问题与注意事项

fork-join的阻塞特性可能导致仿真死锁风险，如果某个子进程永远无法完成（例如等待一个永远不会触发的事件），整个仿真将被挂起。因此，使用fork-join时需要确保所有子进程都有明确的终止条件。

## fork-join_any

### 基本语法与语义

父进程会阻塞，直到由fork启动的**任意一个**子进程完成**即可**继续执行后续代码

```systemverilog
initial begin
    fork
        begin 
          #10;
          $display("Fast process completed");
        end
        begin 
          #50;
          $display("Slow process completed");
        end
    join_any
    $display("At least one process finished at time %0t", $time);
end
```

仿真输出：

```

- 时间10：Fast process completed
- 时间10：At least one process finished
- 时间50：Slow process completed（仍在后台执行）
```

### 关键特性：残余进程问题

**未完成的子进程不会自动终止，他们会继续在后台执行**，可能导致意外的行为和资源泄漏

```systemverilog
initial begin
    fork
        begin            
          #100;
          $display("Background task A");
        end
        begin            
          #10;
          $display("Foreground task B");
        end
    join_any
    $display("Main continues at time %0t", $time);
    // 注意：任务A仍在执行，可能会干扰后续逻辑
end
```

为了处理参与进程，SV提供了`disable fork`语句

```systemverilog

initial begin
    fork
        begin 
           #100;
           $display("Task A");
        end
        begin 
           #10;
           $display("Task B");
        end
    join_any
    disable fork;  // 终止所有残余子进程
    $display("All background tasks killed");
end
```

### 应用场景分析

- **超时机制实现：**用于实现带超时等待的功能

  ```systemverilog
  task wait_with_timeout(input event done_evt, input int timeout_ns);
      fork
          begin
              @done_evt;
              $display("Operation completed successfully");
          end
          begin            
              #timeout_ns;
              $display("Error: Operation timed out after %0dns", timeout_ns);
          end
      join_any
      disable fork;  // 清理残余进程
  endtask
  ```

- **竞争响应模式：**当存在多个可用资源或路径，只需要其中一个响应时

  ```systemverilog
  initial begin
      fork
          begin
              // 尝试从快速缓存获取
              if (cache.available) begin
                  result = cache.get();
                  $display("Got from cache");
              end
          end
          begin
              // 同时尝试从慢速存储获取            
              #5;
              result = storage.get();
              $display("Got from storage");
          end
      join_any
      disable fork;
  end
  ```

- **中断响应模拟：**模拟硬件中断处理行为

  ```systemverilog
  initial begin
      fork
          begin
              // 正常操作序列
              execute_normal_sequence();
          end
          begin
              // 等待中断信号
              @(posedge interrupt_signal);
              $display("Interrupt received, aborting normal sequence");
          end
      join_any
      disable fork;
      handle_interrupt();
  end
  ```

## fork-join_none

### 基本语法与语义

完全非阻塞的并发启动机制，父进程不会被阻塞，不等待任何子进程完成，后续代码立即执行。

```systemverilog
initial begin
    fork
        begin 
          #100;
          $display("Background task 1");
        end
        begin 
          #200;
          $display("Background task 2"); 
        end
    join_none
    $display("Main thread continues immediately at time %0t", $time);
end
```

输出结果：

```
- 时间0：Main thread continues immediately
- 时间100：Background task 1
- 时间200：Background task 2
```

### 子进程启动时机

一个容易忽视的细节是：fork-join_none中的子进程并非在fork语句处立即开始执行。根据IEEE 1800标准，子进程的启动会延迟到父进程执行到下一个阻塞语句或终止时。

```systemverilog
initial begin
    fork
        begin
            $display("Background at time %0t", $time);
        end
    join_none
    $display("Main at time %0t", $time);
    #0;  // 这个零延迟触发后台进程启动
    $display("Main after #0 at time %0t", $time);
end
```

输出顺序为：

```
- Main at time 0
- Background at time 0
- Main after #0 at time 0
```

> [!WARNING]
>
> 1. 这种行为在某些情况下可能导致微妙的时序问题，需要特别注意
>
> 2. 可以将#0换成wait fork
> 3. 父进程（initial块）如果没有遇到任何阻塞语句（#1，@（...）,wait(...)等），它在0仿真时间内直接走到底，当initial块终止时，它所有的子进程被一并清理

### 应用场景

- **后台监控任务：**启动长期运行的监控线程，不阻塞测试流程

  ```systemverilog
  initial begin
      // 启动后台监控
      fork
          begin
              forever begin
                  @(posedge clk);
                  check_protocol_violations();
              end
          end
      join_none
      // 主测试流程不受影响
      run_main_test_sequence();
  end
  ```

- **并行激励注入：**同时启动多个独立的激励序列

  ```systemverilog
  initial begin
      fork
          begin generate_clock_sequence(); end
          begin generate_reset_sequence(); end
          begin generate_bus_traffic(); end
      join_none
      // 主测试可以继续配置和协调
      configure_test_environment();
  end
  ```

- **回调机制实现：**在某些事件触发时启动处理任务

  ```systemverilog
  
  always @(posedge trigger_event) begin
      fork
          begin
              process_trigger();
          end
      join_none
      // 主流程继续，不等待处理完成
  end
  ```

## 变量作用域和数据安全

1. 在fork-join结构中，子进程共享父进程作用域中的变量，可能导致数据竞争

   ```systemverilog
   initial begin
       int i;
       for (i = 0; i < 5; i++) begin
           fork
               begin
                   $display("Value of i: %0d", i);
               end
           join_none
       end    
       #1;  // 等待后台进程执行
   end
   ```

   代码执行的输出为5个相同的值。原因在于所有子进程共享同一个变量i,而fork-join_none会延迟子进程启动，当子进程开始执行时，i的值已经变为了5

2. 使用automatic变量解决方案

   使用automatic关键字声明局部变量，确保每个线程拥有独立的变量副本

   ```systemverilog
   initial begin
       for (int i = 0; i < 5; i++) begin
           automatic int j = i;  // 每次循环创建独立的副本
           fork
               begin
                   $display("Value of j: %0d", j);
               end
           join_none
       end    
       #1;
   end
   ```

   输出将正确显示0、1、2、3、4



# process类：进程对象的精细控制