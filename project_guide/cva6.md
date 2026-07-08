# 环境部署

1. 初始化模块

   ```shell
   git clone https://github.com/openhwgroup/cva6.git
   cd cva6
   git submodule update --init --recursive
   ```

2. 标准安装包

   ```shell
   $ sudo apt-get install autoconf automake autotools-dev curl git libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool bc zlib1g-dev
   ```

3. 安装工具链

   进入cva6/utils/toolchain-builder目录下执行以下命令：

   ```shell
   # 1. Select an installation location for the toolchain (here: the default RISC-V tooling directory $RISCV).
   INSTALL_DIR=$RISCV
   
   # 2. Fetch the source code of the toolchain (assumes Internet access.)
   bash get-toolchain.sh
   
   # 3. Build and install the toolchain (requires write+create permissions for $INSTALL_DIR.)
   bash build-toolchain.sh $INSTALL_DIR
   ```

   > [!NOTE]
   >
   > 1. 先配置RISCV环境变量，指定安装路径
   > 2. 执行get-toolchain.sh脚本时，会下载`binutils-gdb`、`gcc`、`newlib`三个组件的源码。其中`binutils-gdb`下载源是sourceware.org，但由于网络不稳定，下载很慢，会报超时错误【修改toolchain-builder/config/global.sh文件中的BINUTILS_REPO，改为从[github镜像](https://github.com/gnutools/binutils-gdb)克隆】
   > 3. 三个组件的源码在路径toolchain-builder/src下
   > 4. 执行第3步build-toolchain.sh比较耗时，大概一个小时左右

Additionally, after getting the GCC toolchain, you can apply `gcc-cva6-tune.patch` to add support for the `-mtune=cva6` option in GCC.

```shell
cd util/toolchain-builder/src/gcc && git apply ../../gcc-cva6-tune.patch
```

4. 安装cmake

   ```shell
   sudo apt update
   sudo apt install cmake
   ```

   使用cmake --version验证

5. Install `help2man` and `device-tree-compiler` packages.

   ```shell
   sudo apt-get install help2man device-tree-compiler
   ```

6. 安装riscv-dv requirements（逐个安装requirements.txt中指定的软件）:

   ```shell
   pip3 install -r verif/sim/dv/requirements.txt
   ```

   