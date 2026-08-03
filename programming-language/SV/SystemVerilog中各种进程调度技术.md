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

## 引入

process类的应用，就是在fork...join/join_any/join_none中创建，后续控制具体的相关进程的执行的一个内建类

**注意**

1. process对象的实例不能使用new，只能调用通过内部的静态函数self()来创建
2. process对象创建必须放在fork...join/join_any/join_none之间开启的进程，否则获取全局进程将毫无意义
3. process的加入使得进程可以作为参数在task间传输，进而使得进程的控制更加灵活，而不必像之前一样通过disable LABEL的方式那样局限

```systemverilog
initial begin
    process p1, p2;
    fork
        begin
            p1 = process::self();
            #2000;
            $display("p1 done! @%t", $time);
        end
        begin
            p2 = process::self();
            #2000;
            $display("p2 done! @%t", $time);
        end
    join_none

    wait(p1 != null && p2 != null);
    #500;
    $display("500 done! @%t", $time);
    $display("p1 status is %s, p2 status is %s @%t",p1.status(), p2.status(), $time);
    #500;
    p1.kill();
    $display("p1 status is %s, p2 status is %s @%t",p1.status(), p2.status(), $time);
    p2.await();
    $display("p1 status is %s, p2 status is %s @%t",p1.status(), p2.status(), $time);
    #1000;
    $display("done! @%t", $time);
    $finish;
end
```

仿真输出：

```
500 done! @500
p1 status is WAITING, p2 status is WAITING @500
p1 status is KILLED, p2 status is WAITING @1000
p2 done! @2000
p1 status is KILLED, p2 status is FINISHED @2000
done! @3000
```

## 核心特性

它允许一个进程访问和控制另一个已启动的进程

**核心特性**

- 系统自动创建：process对象在进程启动时由系统内部自动创建，用户不能手动调用new方法创建。这一设计保证了进程对象的唯一性和生命周期管理的安全性
- 不可扩展：process类被声明为final，不能被继承或扩展。这保证了进程控制机制的一致性和可靠性
- 唯一性：每个进程对应唯一的process对象，进程终止后，其句柄可以重用，但不会创建新的对象

## 进程状态模型

五种进程状态，构成完整的进程生命周期模型：

```
typedef enum {
    FINISHED,   // 进程已正常完成
    RUNNING,    // 进程正在执行
    WAITING,    // 进程在等待某种条件
    SUSPENDED,  // 进程已被挂起
    KILLED      // 进程已被终止
} process_status;
```

- FINISHED：进程已执行完毕，所有语句都已执行，包括任何延迟或事件等待。这是进程的正常终止状态
- RUNNING:进程正在执行中，未被任何事件或条件阻塞
- WAITING状态：进程在等待某种条件，如延迟时间到达、事件触发、信号变化等
- SUSPENDED：进程被显式挂起，暂停执行，直到收到恢复指令
- KILLED状态：进程被强制终止，不再执行任何语句

## process类方法详解

### self()：获取当前进程句柄

1. self()方法返回当前进程的process句柄，是进程控制的基础入口
2. self()方法通常用于：
   - 将当前进程句柄传递给其他组件进行监控
   - 在进程内部进行状态检查
   - 实现进程自注册机制

### status()：状态查询

返回进程的当前状态，用于监控和条件判断

### await（）：进程同步等待

用于等待另一个进程完成，是一种非阻塞的同步机制

### suspend()和resume()：进程挂起与恢复

### kill()进程终止

## process类高级应用模式

略.TODO

# 棋语：资源互斥访问机制

## 基本概念

棋语是一种用于控制对共享资源访问的同步机制，其核心模型是一个“钥匙桶”。进程在继续执行前必须从桶中获取钥匙，使用完资源后将钥匙归还到桶中。这一模型确保了对共享资源的安全访问。

SV中使用semaphore实现：

```systemverilog

// 创建旗语
semaphore sem = new(3);  // 初始有3把钥匙
// 获取钥匙
sem.get(1);  // 获取1把钥匙，如果没有则阻塞
// 访问共享资源
access_shared_resource();
// 归还钥匙
sem.put(1);  // 归还1把钥匙
```

## 应用场景

### 初始钥匙=1

实现互斥锁，用于保护临界区

```systemverilog

semaphore mutex = new(1);  // 只有一把钥匙
task critical_section();
    mutex.get(1);
    // 临界区代码，同一时间只能有一个进程执行
    shared_variable = shared_variable + 1;
    mutex.put(1);
endtask
```

### 初始钥匙>1

控制并发访问数量

```systemverilog
semaphore connection_pool = new(5);  // 允许5个并发连接
task handle_connection();
    connection_pool.get(1);
    // 处理连接，最多同时有5个进程
    process_request();
    connection_pool.put(1);
endtask
```

### 初始钥匙=0

实现事件同步

```systemverilog
semaphore sync_event = new(0);  // 没有初始钥匙
initial begin
    // 等待事件
    sync_event.get(1);  // 阻塞直到有人put钥匙
    $display("Event triggered");
end
initial begin    
    #100;
    sync_event.put(1);  // 触发事件
end
```

## 高级应用模式

略. TODO

# 邮箱：进程间数据通信机制

## 基本概念

邮箱是SV提供的进程间通信机制，用于数据传输和缓存。其行为类似于FIFO，允许一个进程发送消息，另一个进程接收消息

**核心特性**

- 类型安全：可以指定邮箱存储的数据类型

  ```
  mailbox #(transaction) trans_mbx;  // 只能存储transaction类型
  mailbox generic_mbx;               // 可以存储任意类型
  ```

- 容量控制：可以创建有界或无界邮箱

  ```
  mailboxbounded_mbx=new(10);  // 有界邮箱，最多存储10个消息
  mailboxunbounded_mbx=new();  // 无界邮箱，容量无限
  ```

- 阻塞语义：满邮箱put阻塞，空邮箱get阻塞

邮箱与队列的区别：

| 特性     | 邮箱                 | 队列         |
| :------- | :------------------- | :----------- |
| 类型安全 | 类型参数化           | 类型参数化   |
| 容量限制 | 可设限               | 无限制       |
| 并发安全 | 内置线程安全         | 需要外部同步 |
| 阻塞行为 | 满阻塞put，空阻塞get | 无阻塞       |
| 主要用途 | 进程间通信           | 数据结构操作 |

## 邮箱方法

- new()：默认是无界邮箱
- put()
- get()
- try_put()和try_get():非阻塞操作
- peek()：查看消息
- num()：查询消息数量

## 应用场景

略. TODO

# 进程间同步机制

## Event同步

event类型提供轻量级同步机制

```systemverilog

event sync_event;

// 进程1：触发事件
initial begin
 #10;
  ->sync_event;  // 触发事件
 $display("Event triggered at time %0t", $time);
end

// 进程2：等待事件
initial begin
  @sync_event;  // 阻塞等待事件
 $display("Event received at time %0t", $time);
end

// 使用triggered()方法检测事件状态
initial begin
 #15;
 if (sync_event.triggered()) begin
 $display("Event was triggered in current time step");
 end
end
```

## Event vs 旗语 vs 邮箱

| 机制      | 适用场景                | 特点                   |
| :-------- | :---------------------- | :--------------------- |
| Event     | 简单同步，事件通知      | 轻量级，无数据传递     |
| Semaphore | 资源互斥，并发控制      | 支持计数，阻塞获取     |
| Mailbox   | 数据传递，生产者-消费者 | 支持数据传递，可设容量 |

```systemverilog

// 简单同步：使用Event
event data_ready;
initialbegin
  generate_data();
  ->data_ready;
end

initial begin
  @data_ready;
  process_data();
end

// 资源互斥：使用Semaphore
semaphore bus_mutex =new(1);

task access_bus();
  bus_mutex.get(1);
  // 访问总线
  bus_mutex.put(1);
endtask

// 数据传递：使用Mailbox
mailbox #(packet) data_channel =new();

initial begin
  packet pkt = generate_packet();
  data_channel.put(pkt);
end

initial begin
  packet pkt;
  data_channel.get(pkt);
  process_packet(pkt);
end
```

# UVM中的应用

略

# 总结

| 机制           | 核心用途       | 阻塞特性 | 数据传递 | 推荐场景           |
| :------------- | :------------- | :------- | :------- | :----------------- |
| fork-join      | 等待所有子进程 | 阻塞     | 无       | 并行任务同步完成   |
| fork-join_any  | 等待任一子进程 | 阻塞     | 无       | 超时、竞争响应     |
| fork-join_none | 启动后台进程   | 非阻塞   | 无       | 后台监控、并行启动 |
| process类      | 进程控制       | 非阻塞   | 无       | 精细化进程管理     |
| semaphore      | 资源互斥       | 可选阻塞 | 无       | 共享资源保护       |
| mailbox        | 数据传递       | 可选阻塞 | 是       | 生产者-消费者      |
| event          | 事件通知       | 阻塞     | 无       | 简单同步           |

## **fork-join系列**

1. 使用automatic变量避免变量共享问题
2. 始终在fork-join_any后使用disable fork清理残余进程
3. 理解fork-join_none的延迟启动特性
4. 避免深层嵌套fork结构增加调试难度

## **process类**

1. 不要尝试手动创建process对象
2. 在suspend前检查进程状态
3. 使用await而非disable进行优雅等待
4. 实现进程池模式管理大量进程

## **旗语**

1. 确保get和put操作成对出现
2. 定义全局的旗语获取顺序避免死锁
3. 使用try_get实现非阻塞检查
4. 考虑封装旗语操作保证一致性

## **邮箱**

1. 对长期运行的邮箱设置容量限制
2. 传递对象时注意引用语义
3. 使用try_put/try_get实现非阻塞操作
4. 考虑使用peek预检查消息

## **竞争条件预防**

1. 时序逻辑使用非阻塞赋值
2. 不要在同一个always块混合赋值类型
3. 使用program块在Reactive区域执行验证代码
4. 利用事件区域分离设计和验证

## **死锁预防**

1. 按固定顺序获取多个资源
2. 使用超时机制避免无限等待
3. 使用try_get尝试非阻塞获取
4. 实现资源管理器集中管理

## 9.3 性能优化要点

1. **控制进程数量：**过多的进程增加调度开销
2. **合理设置邮箱容量：**平衡内存和阻塞时间
3. **避免过度同步：**只在必要时使用同步机制
4. **使用工作池模式：**复用进程而非频繁创建销毁
5. **选择合适的同步原语：**event最轻量，mailbox功能最全