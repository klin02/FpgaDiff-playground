# Testing Flow

This document describes the end-to-end FPGA DiffTest pipeline. Each stage corresponds to a top-level `make` target.

```text
  verilog ──► release ──► host
                │
                ▼
               bit       nemu    workload
                │          │         │
                ▼          ▼         ▼
           bitstream/   ready-to-run/  ready-to-run/
                │          │         │
                └──────────┼─────────┘
                           ▼
                     (sync to FPGA host)
                           │
             write_bitstream & reset_cpu
                           │
              ┌────────────┴────────────────┐
              ▼                             ▼
           run_host                    UART/manual
              │                             |
     `fpga-host` runs with            (stty,halt_soc,
    external `FPGA_DDR_LOAD_CMD`   write_jtag_ddr,reset_cpu)
```

## Stage 1: Generate Verilog

Build RTL Verilog from XiangShan or NutShell source with DiffTest modules.

```sh
make verilog xiangshan    # or: make verilog nutshell
```

Key parameters:

| Variable | Default | Description |
|----------|---------|-------------|
| `DIFFTEST_CONFIG` | `ESBIFDU` | DiffTest config letters (see `difftest/docs/hw-flow.md`) |
| `DIFFTEST_EXCLUDE` | `Vec` | Comma-separated exclude list |
| `JOBS` | `16` | Parallel compilation jobs |

Output: Verilog files under the design's `build/` directory.

Log: `build/build-log/verilog-<design>-<timestamp>.log`

## Stage 2: Create Release

Package the generated Verilog and DiffTest source into a self-contained release directory.

```sh
make release xiangshan
```

Output:

```text
build/release/<release-name>/
build/release/latest-xiangshan.path   # absolute path (for local use)
build/release/latest-xiangshan.name   # directory name only (for remote path construction)
```

The release name follows the pattern `YYYYmmdd_<TopModule>_<Config>_<Suffix>`. The default `RELEASE_SUFFIX` is `HHMMSS` to avoid overwrites within the same day. To suppress the suffix:

```sh
make release xiangshan RELEASE_SUFFIX=
```

Log: `build/build-log/release-<design>-<timestamp>.log`

## Stage 3: Build FPGA Host

Compile the `fpga-host` binary inside the release directory.

```sh
export XS_RELEASE=$(cat build/release/latest-xiangshan.path)
make host xiangshan FPGA_HOST_HOME=$XS_RELEASE
```

The host binary is built with `RELEASE=1 FPGA=1 DIFFTEST_PERFCNT=1` by default. The output binary is at:

```text
$XS_RELEASE/build/fpga-host
```

Log: `build/build-log/host-<design>-<timestamp>.log`

## Stage 4: Generate Bitstream

Run Vivado synthesis and implementation to produce a `.bit` file. This can run locally or on a remote host.

```sh
make bit xiangshan
# or remotely:
make bit xiangshan REMOTE=open103 REMOTE_DIR=/nfs/path/to/repo
```

Key parameters:

| Variable | Default | Description |
|----------|---------|-------------|
| `REMOTE` | (none) | Remote host for Vivado execution |
| `REMOTE_DIR` | (none) | Repository path on the remote host |
| `REMOTE_ENV` | `source ~/.bash_profile &&` | Environment setup command on the remote host |
| `BIT_SRC_DIR` | latest release | Override the release directory used for synthesis |

Output:

```text
bitstream/<design>-YYYYmmdd-HHMMSS/
  ├── <release-name>/          # the release used
  ├── fpga_top_debug.bit
  └── *.ltx
```

Log: `build/build-log/bit-<cpu>-<timestamp>.log`

## Stage 5: Build NEMU Reference

Compile the NEMU reference model as a shared library.

```sh
make nemu                                                  # default XS config
make nemu NEMU_CONFIG=riscv64-nutshell-ref_defconfig       # NutShell config
```

Default config: `riscv64-xs-ref-novec-nopmppma_defconfig`

Output: `ready-to-run/<NEMU_CONFIG>/riscv64-nemu-interpreter-so`

Log: `build/build-log/nemu-<config>-<timestamp>.log`

## Stage 6: Build Workload

Compile a workload and convert it to DDR initialization format.

```sh
make workload TARGET=linux/hello
```

This produces both the raw binary and the Bin2ddr `.txt`:

```text
ready-to-run/linux-hello/linux-hello.bin
ready-to-run/linux-hello/linux-hello.txt
```

See [workload.md](./workload.md) for available targets and UART/device tree configuration.

Log: `build/build-log/workload-<target>-<timestamp>.log`

## Stage 7: Sync to FPGA Host

If the FPGA host machine does not share NFS, copy the bitstream bundle and `ready-to-run/` to the remote fixed path:

```sh
FPGA_ROOT=/home/youkunlin/FpgaDiff-playground
BIT_TAG=xiangshan-YYYYmmdd-HHMMSS

ssh fpga "mkdir -p $FPGA_ROOT/bitstream $FPGA_ROOT/ready-to-run"
rsync -a --delete bitstream/$BIT_TAG/ fpga:$FPGA_ROOT/bitstream/$BIT_TAG/
rsync -a --delete ready-to-run/ fpga:$FPGA_ROOT/ready-to-run/
```

## Stage 8: FPGA Operations

All FPGA operation targets support `REMOTE=` for remote execution.

### Common Preparation: Write Bitstream

```sh
make write_bitstream \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG
```

After that, choose one of the following run modes. Both paths start with `reset_cpu`.

### Path A: Let Host Trigger `write_jtag_ddr`

Diff mode:

```sh
make reset_cpu \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG

make run_host \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG \
  WORKLOAD=$FPGA_ROOT/ready-to-run/linux-hello \
  DIFF=$FPGA_ROOT/ready-to-run/$NEMU_CONFIG/riscv64-nemu-interpreter-so
```

No-diff mode:

```sh
make reset_cpu \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG

make run_host \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG \
  WORKLOAD=$FPGA_ROOT/ready-to-run/linux-hello
```

In this path, `run_host` passes a `write_jtag_ddr` command into `fpga-host` through `FPGA_DDR_LOAD_CMD`. `WORKLOAD` is a directory; `run_host` picks the `.bin` and `.txt` inside it. Runtime log: `$FPGA_ROOT/build/run-log/run-YYYYmmdd-HHMMSS-NNNNNNNNN.log`

### Path B: UART + Manual JTAG DDR Load

Open UART in one terminal:

```sh
make reset_cpu \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG

stty -F /dev/ttyUSB0 raw 115200
```

Then, in another terminal, stop the SoC, write DDR, and release reset in order:

```sh
make -C env-scripts/fpga_diff halt_soc \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG

make write_jtag_ddr \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG \
  WORKLOAD=$FPGA_ROOT/ready-to-run/linux-hello

make reset_cpu \
  REMOTE=fpga REMOTE_DIR=$FPGA_ROOT \
  FPGA_BIT_HOME=$FPGA_ROOT/bitstream/$BIT_TAG
```

Use this path when you want serial output directly from the board without starting `fpga-host`, or when you want to isolate DDR load / reset issues from host-side DiffTest. The first `reset_cpu` is the shared post-bitstream reset; the final `reset_cpu` releases the CPU after manual DDR load.

## Next Steps

- For a concrete end-to-end walkthrough, see [example.md](./example.md).
- For workload customization (UART, device tree), see [workload.md](./workload.md).
- If something fails, see [troubleshooting.md](./troubleshooting.md).
