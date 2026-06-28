[TOC]

# 1. RISC-V Architectural Certification Tests

## 1.1 简介

1. ACTs是Architectural Certification Tests的缩写
2. 一系列汇编测试，证明设计符合riscv规范
3. 除此之外还需要额外的验证来测试处理器

## 1.2 ACT4框架

1. 使用ACT4框架，一款基于 Makefile 与 Python 开发的工具，用于替代已废弃的 riscof 工具
2. ACT4 框架会为被测设备（DUT）生成并编译可执行可链接格式（ELF）的自检测试用例，还可按需收集覆盖率数据，以此证明测试用例覆盖到了用于校验规范规则的覆盖点。随后用户需自行使用自有测试平台在被测设备上运行所有可执行链接格式文件。每项测试会反馈通过或失败结果，若条件允许，还会向控制台打印错误信息。

## 1.3 前期准备

1. 一个UDB配置文件指定DUT支持的扩展和参数
2. 一个rvmodel_macros.h文件定义DUT特定的操作，比如在终端打印，使用一个成功/失败的code结束用例，产生中断，描述DUT内存映射的linker脚本
3. RISC-V具备高度可配置性，例如是否允许非对齐访问、实现多少个物理内存保护（PMP）寄存器等。因此，各项测试的预期结果会随被测设备（DUT）的配置而变化。ACT4框架会根据被测设备的性能规格筛选适配的测试用例并完成编译，随后调用与被测设备配置保持一致的RISC-V Sail参考模型，计算每一项测试对应的预期结果，最终将这些结果编译生成可自检的ELF可执行文件。

# 2. Getting start

## 2.1 prerequisites

1. make 和 git

   ```bash
   # On Ubuntu/Debian
   sudo apt-get install make git
   
   # On Fedora/CentOS/RHEL
   sudo dnf install make git
   ```

   

2. mise(tool manager)

   - 测试生成器与测试框架均使用Python编写。推荐通过uv项目管理器安装并运行Python，该工具可自动统一管理Python版本、虚拟环境以及各类依赖包（不需要卸载原先的python和pip）。

   - 该框架还依赖riscv-unified-db（UDB）程序包，该程序包需要Ruby语言与Bundler工具。

   - 同时安装这两款工具（uv 和 Ruby）的推荐方式是使用 mise。mise 可自动管理工具版本并完成安装，你无需手动安装 uv、Python、Ruby 或 UDB。

     安装mise

     ```shell
     curl https://mise.jdx.dev/install.sh | sh
     ```

     安装完，验证mise可用

     ```bash
     mise --version
     ```

   如果你已配备专属Python环境，可通过pip将框架包安装至该环境，无需使用mise/uv。你需自行准备Python 3.10及以上版本，且Ruby与Bundler必须单独安装（详见UDB代码仓库）。

3. RISC-V交叉编译工具链

   > [!CAUTION]
   >
   > 从源码编译工具链可能需要数小时

   ```bash
   # Ubuntu/Debian 安装编译依赖  
   sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip \  
     python3-tomli libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison  \  
     flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build \  
     git cmake libglib2.0-dev libslirp-dev libncurses-dev  
     
   # 克隆并编译工具链  
   git clone https://github.com/riscv/riscv-gnu-toolchain  
   cd riscv-gnu-toolchain  
   ./configure --prefix=/path/to/install \  
     --with-multilib-generator="rv32e-ilp32e--;rv32i-ilp32--;rv64i-lp64--;"  
   sudo make
   ```

   编译完成后，将工具链加入 `PATH`（写入 `~/.bashrc`）

   ```bash
   export PATH="/path/to/install/bin:$PATH"  
   riscv64-unknown-elf-gcc --version  # 验证
   ```

4. RISC-V Sail参考模型（版本必须为0.10）

   ```bash
   curl --location \  
     https://github.com/riscv/sail-riscv/releases/download/0.10/sail-riscv-$(uname)-$(arch).tar.gz \  
     | sudo tar xvz --directory=/usr/local --strip-components=1  
     
   sail_riscv_sim --version  # 必须输出 "0.10"
   ```

   > [!CAUTION]
   >
   > *框架在启动时会通过* `check_ref_model_version()` *严格校验版本号，版本不匹配会直接报错退出，**只支持 `0.10`**

5. 容器运行时podman

   UDB（`riscv-unified-db`）配置验证工具运行在容器中，目前**仅支持 Podman**（Docker 支持仍在开发中）

   ```bash
   # Ubuntu/Debian  
   sudo apt-get install podman  
     
   # Fedora/CentOS/RHEL  
   sudo dnf install podman  
     
   podman --version  # 验证
   ```

   

## 2.2 克隆仓库

```bash
git clone https://github.com/riscv/riscv-arch-test -b act4  
cd riscv-arch-test
```

> [!CAUTION]
>
> *必须切换到* `act4` *分支，这是 ACT4 框架所在的分支。仓库包含两个 Git 子模块（*`external/riscv-unified-db` *和* `docs/docs-resources`*），在构建时会自动引用*

# 3. 创建DUT配置目录

在 `config/cores/<vendor>/<dut-config-name>/` 下创建你的 DUT 配置目录，例如参考 `config/cores/cvw/cvw-rv64gc/`

配置目录必须包含以下7个文件：

| 文件                | 说明                                       |
| ------------------- | ------------------------------------------ |
| `test_config.yaml`  | ACT4 框架主配置（编译器路径、Sail 路径等） |
| `<dut-name>.yaml`   | UDB 架构定义（扩展列表、参数值）           |
| `rvmodel_macros.h`  | DUT 专用汇编宏实现                         |
| `link.ld`           | 链接脚本（内存映射）                       |
| `sail.json`         | Sail 参考模型配置                          |
| `rvtest_config.h`   | C 头文件（扩展支持标志）                   |
| `rvtest_config.svh` | SystemVerilog 头文件（扩展支持标志）       |

## 3.1 test_config.yaml最小示例

```yaml
name: my-dut  
compiler_exe: riscv64-unknown-elf-gcc  
ref_model_exe: sail_riscv_sim  
udb_config: my-dut.yaml  
linker_script: link.ld  
dut_include_dir: .  
include_priv_tests: False  # 若不需要特权模式测试可设为 False
```

> [!CAUTION]
>
> `udb_config`*、*`linker_script`*、*`dut_include_dir` *中的相对路径均相对于* `test_config.yaml` *所在目录解析*

## 3.2 `link.ld`链接脚本要求

- `ENTRY` 入口点必须为 `rvtest_entry_point`
- 必须包含 `.text`、`.bss`、`.data` 三个 section（顺序不能颠倒）
- 若修改了基地址，**必须同步修改 `sail.json` 中的 `memory.regions` 基地址**，否则 Sail 模型无法正确执行代码

## 3.3 `rvmodel_macro.h`必须实现的宏

| 类别   | 宏名                                      | 是否必须            |
| ------ | ----------------------------------------- | ------------------- |
| 终止   | `RVMODEL_HALT_PASS`、`RVMODEL_HALT_FAIL`  | **必须**            |
| 打印   | `RVMODEL_IO_INIT`、`RVMODEL_IO_WRITE_STR` | 可留空              |
| 启动   | `RVMODEL_BOOT`、`RVMODEL_DATA_SECTION`    | 可留空              |
| 定时器 | `RVMODEL_MTIME_ADDRESS` 等                | 不支持 M 模式可留空 |
| 中断   | `RVMODEL_SET_MEXT_INT` 等                 | 不支持中断可留空    |

# 4. 生成并编译测试ELF

运行以下指令自动生成汇编文件，编译并创建self-checking ELFs

```bash
# 使用所有 CPU 核心并行编译  
CONFIG_FILES=config/cores/my-vendor/my-dut/test_config.yaml make --jobs $(nproc)  
  
# 指定自定义输出目录（默认为 work/）  
WORKDIR=/path/to/workdir CONFIG_FILES=/path/to/test_config.yaml make --jobs $(nproc)
```

编译产物位于 `$WORKDIR/<config_name>/elfs/`（自检 ELF）和 `$WORKDIR/<config_name>/build/`（中间产物）

## 4.1 常用make目标

| 目标           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `make`（默认） | 完整构建：生成测试 → 编译 `.sig.elf` → 运行 Sail → 编译自检 `.elf` |
| `make tests`   | 仅生成汇编源文件，**不需要编译器和 Sail**，适合初始验证环境  |
| `make elfs`    | 编译 ELF（需要 Sail 和编译器）                               |
| `make clean`   | 清除 `work/` 下所有构建产物                                  |

# 5. 运行测试并解读结果

将 `elfs/` 目录下的所有 ELF 文件加载到 DUT 上运行，每个测试会通过 `RVMODEL_IO_WRITE_STR` 宏输出：

```
RVCP-SUMMARY: Test File "add-01.S": PASSED  
RVCP-SUMMARY: Test File "add-01.S": FAILED  
```

失败时会额外打印：失败的 PC、失败的指令、哪个寄存器不匹配、期望值（来自 Sail）、实际值（来自 DUT）。

# 6. 常见问题排查

| 现象                                    | 原因                               | 解决方法                                                |
| --------------------------------------- | ---------------------------------- | ------------------------------------------------------- |
| `Sail reference model version mismatch` | `sail_riscv_sim` 版本不是 `0.10`   | 严格安装 `0.10` 版本                                    |
| UDB 配置验证报错                        | UDB YAML 格式错误或字段缺失        | 对照 `config/cores/cvw/cvw-rv64gc/cvw-rv64gc.yaml` 检查 |
| 期望值错误但实际值合理                  | `sail.json` 与 UDB YAML 参数不一致 | 确保两者参数完全对齐                                    |
| 实际值错误但期望值合理                  | DUT 本身存在 bug                   | 查看 `.elf.objdump` 文件，定位失败 PC                   |
| `podman: command not found`             | 未安装 Podman                      | 安装 Podman；若使用 Docker 则设置 `DOCKER=1`            |
| 特定扩展测试全部 FAIL                   | `rvmodel_macros.h` 缺少对应宏实现  | 检查该扩展所需宏是否已实现                              |

# 7. 关键注意事项汇总

1. **分支**：必须使用 `act4` 分支，`main` 分支为旧版 `riscof` 工具。
2. **Sail 版本锁定**：框架硬性要求 `sail_riscv_sim` 版本为 `0.10`，不可使用其他版本。 config.py:93-114
3. **容器运行时**：目前仅支持 Podman，Docker 支持仍在开发中（可通过 `DOCKER=1` 环境变量尝试）。
4. **基地址一致性**：`link.ld` 中的基地址必须与 `sail.json` 的 `memory.regions` 完全一致。 README.md:247-249
5. **`sail.json`/`rvtest_config.h`/`rvtest_config.svh` 目前需手写**：这三个文件未来会从 UDB YAML 自动生成，但当前版本需要手动维护，参考 `config/cores/cvw/cvw-rv64gc/` 目录下的示例。 README.md:251-256
6. **快速验证环境**：在没有编译器和 Sail 的情况下，可先运行 `make tests` 验证 Python 环境和配置文件是否正确，只需 `make` 和 `uv` 即可。
7. **路径问题**：`CONFIG_FILES` 和 `WORKDIR` 默认相对于仓库根目录，若使用仓库外路径需使用绝对路径。