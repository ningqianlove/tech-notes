# 背景

1. 1992年发布1.1版本

2. 飞利浦公司（Philips）研发

3. 概念：I2C总线支持任何集成电路制造工艺（NMOS,CMOS,bipolar）。两根线，串行数据（SDA）和串行时钟（SCL），在连接到总线的设备之间传递信息。每个设备通过唯一的地址（无论是微控制器、LCD驱动器、内存还是键盘接口）被识别，并可以根据设备的功能作为发射器或接收器进行操作。显然，LCD驱动器仅为接收器，而内存可以接收和传输数据。除了发射器和接收器外，设备在执行数据传输时还可以被视为主设备(master)或从设备（slave）。主设备是启动总线上数据传输并生成时钟信号以允许该传输的设备，在此时，任何被寻址的设备都被视为从设备。

4. 数据传输方向和设备之间的关系：

   1） Suppose microcontroller A wants to send information to
   microcontroller B:
   • microcontroller A (master), addresses microcontroller B
   (slave)
   • microcontroller A (master-transmitter), sends data to
   microcontroller B (slave- receiver)
   • microcontroller A terminates the transfer

   2） If microcontroller A wants to receive information from

   microcontroller B:
   • microcontroller A (master) addresses microcontroller B
   (slave)
   • microcontroller A (master- receiver) receives data from
   microcontroller B (slave- transmitter)
   • microcontroller A terminates the transfer  

# 特性

- 只需要两条总线线：一条串行数据线SDA和一条串行时钟线SCL

- 每个连接到总线的设备都可以通过唯一地址进行软件寻址，并且在所有时间内都存在简单的主/从关系；主设备可以作为主发射器或主接收器操作

- 这是一种真正的多主设备总线，包括冲突检测和仲裁，以防止在两个或更多主设备同时发起数据传输时的数据损坏

- 在标准模式下，串行、8bit基础单元的双向数据传输可以以高达100kbt/s的速度进行，在快速模式下可达400kbit/s，或在高速模式下可达3.4Mbit/s

- 片上滤波可抑制总线线路上的尖峰，以保持数据完整性

  > I2C SDA/SCL容易受外界干扰产生短暂毛刺；很多I2C外设会在芯片内部集成RC滤波，不需要外接外部电阻电容，也就是on-chip filtering

- 可以连接到同一总线的设备的数量仅受最大总线电容400pF的限制

  > I2C-SDA、SCL依靠上拉电阻把总线拉高；每一颗挂载的芯片都会自带引脚寄生电容
  >
  > - 挂载的器件越多，总线上等效电容就越大
  > - 电容太大时，电平上升沿就会变得迟缓，拉高速度变慢，时序就会出错、通信失败
  > - I2C协议规定：整条总线总寄生电容不可超过400pF,因此不是总线地址数量决定挂载上限（7位地址可挂127个设备），实际硬件上限由引脚电容决定

- I2C总线上时钟信号永远由主机负责产生；每一台主机在总线传输数据时，都会生成属于自身的时钟信号。主机输出的总线时钟，只有两种情况才会被改动：

  - 一是速度较慢的从机拉低时钟线进行时钟拉伸
  - 二是多主机竞争总线、发生总线仲裁的时候，被其他主机改写

  > [!NOTE]
  >
  > 1. 时钟只能由主机产出：I2C-SCL默认由当前通信的主机驱动；从机本身不会主动输出时钟
  >
  > 2. 每个多主机设备，通信时发出自己的时钟。I2C支持多主机架构，哪一个主机抢到总线，就由它输出SCL时钟
  >
  > 3. 仅两类场景主机时钟会被外力强行改变：
  >
  >    - 从机时钟拉伸clock-stretching
  >
  >      低速从机来不及接收字节，就主动把SCL引脚持续拉低，强制暂停主机时钟，等自己准备就绪再释放时钟线。这是从机唯一可以干预时钟的手段
  >
  >    - 多主机中拆arbitration
  >
  >      两台及以上主机同时发起信号，会对比SDA数据线电平进行竞争；输掉仲裁的主机需要立即停止输出时钟，此时时钟就由获胜的主机接管

- SCL和SDA都是双向线路。当总线空闲时，两条线路均为高电平。

- 数据有效性：SDA线上的数据必须在时钟的高电平期间保持稳定，数据线的高电平或低电平状态只能在SCL线上的时钟信号为低电平时发生变化

  ![](assets/image-20260806110815941.png)

# 启动和停止条件

在I2C总线的过程内，会出现独特的情况，这些情况被定义为开始S和停止P条件

- 启动条件：在SCL为高电平时，SDA线上的高电平到低电平的转换
- 停止条件：在SCL为高电平时，SDA线上的低电平到高电平的转换

启动和停止条件时钟由主设备生成。在启动条件之后，系统被认为是忙碌的。在停止条件之后的某段时间，系统被认为重新可用

> [!NOTE]
>
> 如果连接到总线的设备包含必要的接口硬件，则检测启动和停止条件是简单的。然而，未具备此类接口的微控制器必须在每个时钟周期至少对SDA线进行两次采用，以感知状态转换。

![image-20260806112112509](assets/image-20260806112112509.png)

# 传输数据

## 字节格式

1. 放置在SDA线上的每个字节必须为8bit。每次传输的字节数没有限制。每个字节后都必须跟随一个Ack bit
2. 

