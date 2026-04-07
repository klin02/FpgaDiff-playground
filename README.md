# FpgaDiff Playground

这个仓库用于把 FPGA DiffTest 的常用流程串起来：

```text
XiangShan / NutShell Verilog
  -> 顶层 difftest 生成 release / fpga-host
  -> env-scripts/fpga_diff 生成 Vivado bitstream
  -> NEMU 生成 reference so
  -> workload-builder 编译 workload
  -> Bin2ddr 生成 DDR txt
  -> FPGA 上烧 bit、写 DDR、运行 fpga-host 协同仿真
```

完整可执行示例见 `docs/example.md`。DiffTest 细节可参考 `difftest/docs/`。

## 产物目录

顶层构建日志和 release 仍放在 `build/`，待运行输入放在顶层 `ready-to-run/`，待烧写的 bitstream bundle 放在顶层 `bitstream/`。这些目录都已加入 `.gitignore`：

| 路径 | 内容 |
| --- | --- |
| `build/release/` | release tarball、解包后的 release、`latest-<design>.path`、`latest-<design>.name` |
| `build/build-log/` | `verilog`、`release`、`host`、`bit`、`workload`、`nemu` 等构建阶段日志 |
| `build/run-log/` | `run_host` 的运行日志，默认文件名带时间戳 |
| `ready-to-run/<nemu-config>/` | `make nemu` 复制出的 `riscv64-nemu-interpreter-so` |
| `ready-to-run/<target>/` | workload `.bin` 和 Bin2ddr 转出的 `.txt` |
| `bitstream/<design>-<time>/` | 当前 bundle 的 `.bit/.ltx` 和使用的 release 解包目录 |

`RELEASE_SUFFIX` 默认是当前时间 `HHMMSS`，用于避免同一天同配置 release 覆盖。需要隐藏 suffix 时可以显式传空值：

```sh
make release xiangshan RELEASE_SUFFIX=
```

release 的实际路径由 `difftest/scripts/fpga/release.sh` 的日志输出解析得到，日志位于 `build/build-log/release-<design>-<time>.log`。`make release` 会把 release tarball 解到 `build/release/<release-name>`，并写入：

```text
build/release/latest-xiangshan.path
build/release/latest-xiangshan.name
```

本机继续使用 release 时读 `.path`；同步到不共享 NFS 的上位机后，用 `.name` 拼远端路径更可靠。

## 初始化

```sh
make init
```

`init` 会执行顶层 submodule init、各子模块自己的 `make init`，并执行 `make -C Bin2ddr FPGA=1`。

`link_difftest` 会让 `XiangShan/difftest` 和 `NutShell/difftest` 指向顶层 `difftest`，并已包含在 `make init` 里。它不修改 XS/NutShell 的 `.gitmodules`，所以不影响后续 pull 更新；如果内部 difftest 有本地修改，会拒绝替换。

## Verilog / Release / Host

设计不再有默认值，必须显式传入：

```sh
make verilog xiangshan
make verilog nutshell
```

香山常用流程：

```sh
make clean xiangshan
make verilog xiangshan
make release xiangshan
XS_RELEASE=$(cat build/release/latest-xiangshan.path)
make host xiangshan FPGA_HOST_HOME=$XS_RELEASE
```

这些构建阶段的日志会写到 `build/build-log/`。

NutShell 常用流程：

```sh
make clean nutshell
make verilog nutshell
make release nutshell
NUT_RELEASE=$(cat build/release/latest-nutshell.path)
make host nutshell FPGA_HOST_HOME=$NUT_RELEASE
```

`host` 只面向 release 目录运行，因此需要显式传 `FPGA_HOST_HOME=<release-root>`。无论 XiangShan 还是 NutShell，默认参数都是 `RELEASE=1 FPGA=1 DIFFTEST_PERFCNT=1`。

## Vivado Bitstream

`bit` 是唯一的顶层 Vivado 生成入口，内部会执行 `env-scripts/fpga_diff` 的 `all` 和 `bitstream`，然后把 `.bit/.ltx` 和本次使用的 release 目录收集到 `bitstream/<design>-<time>/`：

```sh
make bit xiangshan
```

Vivado 可放到远端执行，例如 `open103`：

```sh
make bit \
  xiangshan \
  REMOTE=open103 \
  REMOTE_DIR=/nfs/home/youkunlin/workspace/FpgaDiff-playground
```

`open103` 的 Vivado 环境若已配置好，不需要额外传 `REMOTE_ENV`。

Vivado 生成日志也会写到远端仓库的 `build/build-log/`，如果 `REMOTE_DIR` 在共享 NFS 上，本机可以直接查看。

默认会使用 `build/release/latest-<design>.path` 指向的 release，`CORE_DIR` 也固定为该 release 下的 `build/`。需要指定其他 release 时传：

```sh
make bit xiangshan BIT_SRC_DIR=/path/to/release
```

生成后的 bundle 形如：

```text
bitstream/xiangshan-YYYYmmdd-HHMMSS/
bitstream/xiangshan-YYYYmmdd-HHMMSS/<release-name>/
bitstream/xiangshan-YYYYmmdd-HHMMSS/*.bit
bitstream/xiangshan-YYYYmmdd-HHMMSS/*.ltx
```

## NEMU Reference

`nemu` 会在 `NEMU/` 里先执行 defconfig，再执行并行编译，最后把 reference so 复制到顶层 `ready-to-run/<NEMU_CONFIG>/`：

```sh
make nemu
```

默认配置是：

```text
riscv64-xs-ref-novec-nopmppma_defconfig
```

切换配置时传 `NEMU_CONFIG`：

```sh
make nemu NEMU_CONFIG=riscv64-nutshell-ref_defconfig
```

默认输出：

```text
ready-to-run/riscv64-xs-ref-novec-nopmppma_defconfig/riscv64-nemu-interpreter-so
```

日志会写到：

```text
build/build-log/nemu-<NEMU_CONFIG>-YYYYmmdd-HHMMSS.log
```

## Workload

`workload` 会编译程序，并把 `.bin` 和 `.txt` 都放到顶层 `ready-to-run/<target>/`：

```sh
make workload TARGET=linux/hello
```

默认输出：

```text
ready-to-run/linux-hello/linux-hello.bin
ready-to-run/linux-hello/linux-hello.txt
```

`TARGET` 会原样传给 `workload-builder`，例如 `linux/hello` 或 `am/<name>`。AM workload 默认取 `package/bin/` 下排序后的第一个 `.bin`。

## 复制到 FPGA 上位机

如果上位机 `fpga` 不共享 NFS，直接把要测试的 bundle 和顶层 `ready-to-run/` 复制到远端固定路径：

```sh
REMOTE_ROOT=/home/youkunlin/FpgaDiff-playground
BIT_TAG=xiangshan-YYYYmmdd-HHMMSS

ssh fpga "mkdir -p $REMOTE_ROOT/bitstream $REMOTE_ROOT/ready-to-run"
rsync -a --delete \
  bitstream/$BIT_TAG/ \
  fpga:$REMOTE_ROOT/bitstream/$BIT_TAG/
rsync -a --delete ready-to-run/ fpga:$REMOTE_ROOT/ready-to-run/
```

同步后远端关键路径为：

```text
/home/youkunlin/FpgaDiff-playground/bitstream/<bundle-name>/<release-name>
/home/youkunlin/FpgaDiff-playground/bitstream/<bundle-name>/*.bit
/home/youkunlin/FpgaDiff-playground/bitstream/<bundle-name>/*.ltx
/home/youkunlin/FpgaDiff-playground/ready-to-run/<NEMU_CONFIG>
/home/youkunlin/FpgaDiff-playground/ready-to-run/<target>
```

## 烧写和运行

target 名称与 `env-scripts/fpga_diff/Makefile` 保持一致：

```sh
make write_bitstream FPGA_BIT_HOME=/path/to/bitstream-dir
make write_jtag_ddr FPGA_BIT_HOME=/path/to/bitstream-dir DDR_WORKLOAD=/path/to/workload.txt
make reset_cpu FPGA_BIT_HOME=/path/to/bitstream-dir
```

这些命令支持 `REMOTE`。在 `fpga` 上位机运行时，推荐使用远端固定路径：

```sh
REMOTE_ROOT=/home/youkunlin/FpgaDiff-playground
BIT_TAG=xiangshan-YYYYmmdd-HHMMSS
BIT_ROOT=$REMOTE_ROOT/bitstream/$BIT_TAG

make write_bitstream \
  REMOTE=fpga \
  REMOTE_DIR=$REMOTE_ROOT \
  FPGA_BIT_HOME=$BIT_ROOT

make write_jtag_ddr \
  REMOTE=fpga \
  REMOTE_DIR=$REMOTE_ROOT \
  FPGA_BIT_HOME=$BIT_ROOT \
  DDR_WORKLOAD=$REMOTE_ROOT/ready-to-run/linux-hello/linux-hello.txt
```

`run_host` 也支持 `REMOTE`，默认日志写到远端 `build/run-log/run-YYYYmmdd-HHMMSS-NNNNNNNNN.log`：

```sh
XS_RELEASE_NAME=$(cat build/release/latest-xiangshan.name)
BIT_TAG=xiangshan-YYYYmmdd-HHMMSS
BIT_ROOT=/home/youkunlin/FpgaDiff-playground/bitstream/$BIT_TAG

make run_host \
  REMOTE=fpga \
  REMOTE_DIR=/home/youkunlin/FpgaDiff-playground \
  HOST_BIN=$BIT_ROOT/$XS_RELEASE_NAME/difftest/build/fpga-host \
  HOST_ARGS="--diff /home/youkunlin/FpgaDiff-playground/ready-to-run/riscv64-xs-ref-novec-nopmppma_defconfig/riscv64-nemu-interpreter-so -i /home/youkunlin/FpgaDiff-playground/ready-to-run/linux-hello/linux-hello.bin"
```
