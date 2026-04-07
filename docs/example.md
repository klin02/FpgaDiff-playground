# FPGA DiffTest 执行示例

这个例子假设：

- `node004` 用于编译 XiangShan/NutShell Verilog、release、`fpga-host`、NEMU reference so 和 workload
- `open103` 用于 Vivado 生成 bitstream
- `node004` 和 `open103` 共享 `/nfs`
- `fpga` 是 FPGA 上位机，不共享 `/nfs`
- NFS 仓库路径是 `/nfs/home/youkunlin/workspace/FpgaDiff-playground`
- `fpga` 上位机仓库路径是 `/home/youkunlin/FpgaDiff-playground`

下面以 XiangShan 为例。NutShell 时把设计参数换成 `nutshell`。

## 1. 在 node004 编译 Verilog、release 和 host

登录 `node004`：

```sh
cd /nfs/home/youkunlin/workspace/FpgaDiff-playground

make init

make clean xiangshan
make verilog xiangshan

make release xiangshan
export XS_RELEASE=$(cat build/release/latest-xiangshan.path)
export XS_RELEASE_NAME=$(cat build/release/latest-xiangshan.name)

make host xiangshan FPGA_HOST_HOME=$XS_RELEASE
```

此时 release 位于：

```sh
echo $XS_RELEASE
```

默认形如：

```text
/nfs/home/youkunlin/workspace/FpgaDiff-playground/build/release/20260407_XSTop_FpgaDiffDefaultConfig_FullDiff-noVec_ESBIFDU_012345
```

最后的 `012345` 来自默认 `RELEASE_SUFFIX=HHMMSS`，用于避免重复。

Verilog、release、host 的日志在：

```text
build/build-log/
```

## 2. 在 open103 生成 bitstream

`open103` 的 Vivado 环境已经配置好，因此不需要额外设置 `REMOTE_ENV`：

```sh
cd /nfs/home/youkunlin/workspace/FpgaDiff-playground

make bit \
  xiangshan \
  REMOTE=open103 \
  REMOTE_DIR=/nfs/home/youkunlin/workspace/FpgaDiff-playground
```

输出会收集到：

```text
bitstream/xiangshan-YYYYmmdd-HHMMSS/
```

该目录下应包含 `.bit`、`.ltx` 和本次使用的 release 目录。默认会使用 `build/release/latest-xiangshan.path` 指向的 release。

Vivado 日志也会写到共享 NFS 上的：

```text
build/build-log/bit-kmh-YYYYmmdd-HHMMSS.log
```

## 3. 构建 NEMU、workload 并生成 DDR txt

仍在 NFS 仓库上执行：

```sh
cd /nfs/home/youkunlin/workspace/FpgaDiff-playground

export NEMU_CONFIG=riscv64-xs-ref-novec-nopmppma_defconfig
make nemu NEMU_CONFIG=$NEMU_CONFIG

make workload TARGET=linux/hello
```

输出文件：

```text
ready-to-run/$NEMU_CONFIG/riscv64-nemu-interpreter-so
ready-to-run/linux-hello/linux-hello.bin
ready-to-run/linux-hello/linux-hello.txt
```

NEMU、workload 和 Bin2ddr 的日志在：

```text
build/build-log/nemu-$NEMU_CONFIG-YYYYmmdd-HHMMSS.log
build/build-log/workload-linux-hello-YYYYmmdd-HHMMSS.log
```

## 4. 复制到 fpga 上位机

`fpga` 不共享 NFS，直接把要测试的顶层 `bitstream/<design>-<time>/` bundle 和顶层 `ready-to-run/` 复制到远端固定路径。下面的 `BIT_TAG` 替换成第 2 步生成的实际目录名：

```sh
cd /nfs/home/youkunlin/workspace/FpgaDiff-playground
export NEMU_CONFIG=riscv64-xs-ref-novec-nopmppma_defconfig
export FPGA_ROOT=/home/youkunlin/FpgaDiff-playground
export BIT_TAG=xiangshan-YYYYmmdd-HHMMSS

ssh fpga "mkdir -p $FPGA_ROOT/bitstream $FPGA_ROOT/ready-to-run"
rsync -a --delete \
  bitstream/$BIT_TAG/ \
  fpga:$FPGA_ROOT/bitstream/$BIT_TAG/
rsync -a --delete ready-to-run/ fpga:$FPGA_ROOT/ready-to-run/
```

同步后远端关键路径为：

```text
/home/youkunlin/FpgaDiff-playground/bitstream/<bundle-name>/$XS_RELEASE_NAME
/home/youkunlin/FpgaDiff-playground/bitstream/<bundle-name>/*.bit
/home/youkunlin/FpgaDiff-playground/bitstream/<bundle-name>/*.ltx
/home/youkunlin/FpgaDiff-playground/ready-to-run/$NEMU_CONFIG/riscv64-nemu-interpreter-so
/home/youkunlin/FpgaDiff-playground/ready-to-run/linux-hello/linux-hello.bin
/home/youkunlin/FpgaDiff-playground/ready-to-run/linux-hello/linux-hello.txt
```

## 5. 在 fpga 上烧写、写 DDR、运行 host

在 NFS 仓库中通过 `REMOTE=fpga` 执行：

```sh
cd /nfs/home/youkunlin/workspace/FpgaDiff-playground

export FPGA_ROOT=/home/youkunlin/FpgaDiff-playground
export BIT_TAG=xiangshan-YYYYmmdd-HHMMSS
export BIT_ROOT=$FPGA_ROOT/bitstream/$BIT_TAG
export XS_RELEASE_NAME=$(cat build/release/latest-xiangshan.name)
export NEMU_CONFIG=riscv64-xs-ref-novec-nopmppma_defconfig

make write_bitstream \
  REMOTE=fpga \
  REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$BIT_ROOT

make write_jtag_ddr \
  REMOTE=fpga \
  REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$BIT_ROOT \
  DDR_WORKLOAD=$FPGA_ROOT/ready-to-run/linux-hello/linux-hello.txt

make reset_cpu \
  REMOTE=fpga \
  REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$BIT_ROOT
```

运行 host：

```sh
make run_host \
  REMOTE=fpga \
  REMOTE_DIR=$FPGA_ROOT \
  HOST_BIN=$BIT_ROOT/$XS_RELEASE_NAME/difftest/build/fpga-host \
  HOST_ARGS="--diff $FPGA_ROOT/ready-to-run/$NEMU_CONFIG/riscv64-nemu-interpreter-so -i $FPGA_ROOT/ready-to-run/linux-hello/linux-hello.bin"
```

日志默认写到远端：

```text
/home/youkunlin/FpgaDiff-playground/build/run-log/run-YYYYmmdd-HHMMSS-NNNNNNNNN.log
```
