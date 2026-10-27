# OpenPiton on CAPI=3

This branch (`capi3-migrate`) converts OpenPiton's core files to CAPI=3 to test
`fusesoc core migrate` and CAPI=3 on a real code base. It needs the CAPI=3 branch of
FuseSoC and Edalize 0.6.8.

## Status

| | Count |
|---|---|
| Core files | 68 |
| Converted to CAPI=3 | 68 |
| Left on CAPI=2 | 0 |
| Cores that load (`fusesoc core list`) | 68 (66 before, see below) |

`fusesoc run --setup` of every target with a flow (91), compared with the EDAM
CAPI=2 produces for the same target:

| Result | Targets |
|---|---|
| Same EDAM as CAPI=2 | 75 |
| Intended difference | 0 |
| Fails under both CAPI=2 and CAPI=3 | 16 |
| Works under CAPI=2, broken under CAPI=3 | 0 |

- **Failures under both:** problems already in OpenPiton. `io_ctrl` and `oled`
  `pickle`/`verify` and `test_infrstrct` `default` have no `toplevel`;
  `noc_sd_bridge` lists a missing `bistable_domain_cross.v`, which also fails
  `system` `fpga`; `piton_common` runs `pyhp_preprocess_gen` without depending on
  `openpiton::preprocessor`, so it and the `fpga_bridge`, `fpga_bridge_rcv_32` and
  `fpga_bridge_send_32` targets find no generator; `chip_fpga_bridge_tb` depends on
  `openpiton::fpga_rcv_32`, a core named `fpga_bridge_rcv_32`, so the flattened
  VCS options there are untested.
- **Also checked:** `manycore_tb` `sim` with `--var +tool_verilator ... --tool=verilator`
  gets the same C++ sources as CAPI=2 with `--tool=verilator`.
- **Baseline:** the CAPI=2 side had the two fixes below applied too; without them
  most targets do not resolve.
- **Not tested:** only setup was run; nothing was compiled or simulated. The
  comparison folds CAPI=2's `tool_options.<tool>` into flat `flow_options`, which is
  where the flow API puts them.

## Fixed before converting

Two cores were not valid CAPI=2, so FuseSoC ignored them and the converter refused
them. Both are fixed in the CAPI=3 files:

- `piton/design/chip/tile/sparc/exu/bw_r_irf/common/rtl/exu_bw_r_irf_common.core` had
  an empty `description:`. It now reads `EXU Register file`, as its sibling cores do.
  Every core above `openpiton::sparc_exu` (tile, chip, system, …) depends on it, so
  none of them resolved before.
- `piton/verif/env/manycore/manycore.core` had an empty `tools:` in `sim`. It is gone.

## What to expect

- **Tursi needs one change.** It runs `fusesoc run --target=pickle --tool=icarus <core>`
  (and the same for `verify`). A CAPI=3 top level rejects the global `--tool`. Drop
  it: every target sets `flow: sim` with the flow option `tool: icarus`. The
  `--iverilog_options=...` after the core name still adds to the core's options, as
  it did with CAPI=2. Tursi is not changed on this branch.
- **`pickle` and `verify` run the `sim` flow with Icarus Verilog.** That is the setup
  CAPI=2 produced; `pickle` still only preprocesses (`-E`). The converter asked
  whether these targets simulate or lint; they stay on `sim`, the setup Tursi was
  written against.
- **No tool variables or flags on the targets.** The converter set
  `vars: {tool_icarus: true}` and `flags: {tool: icarus}` on every Icarus target, for
  dependencies that might read them. No core of the tree does, so they are left out;
  otherwise every build warns that `tool_icarus` is set but no core declares it.
- **Chipset follows the flow.** `chipset` added the memory controller, SD bridge and
  OLED cores on `tool_vivado`. That is `flow == 'vivado'` now, so `system`'s `fpga`
  target gets them without setting anything.
- **Verilator is a variable.** `manycore_tb` `sim` adds its C++ harness and PLI
  sources on `tool_verilator`, which CAPI=3 cannot see. The core declares a
  `tool_verilator` variable; run the Verilator build with
  `fusesoc run --target=sim --var +tool_verilator openpiton::manycore_tb --tool=verilator`.
- **VCS options are flat.** `chip_fpga_bridge_tb` `sim` had `tools.vcs.vcs_options`.
  They are in `flow_options` now, used when you pass `--tool=vcs` after the core name.
  Icarus ignores them.
- **Generated cores stay CAPI=2.** `pyhp_preprocess_gen` (`preproc.py`) writes a
  CAPI=2 core, and FuseSoC accepts only CAPI=2 from a generator. The converter asked
  to check the parameters of generated cores, which are ordered differently once a
  tree holds a CAPI=3 core. The generated core declares no parameters, so nothing
  changes.
- **`test_infrstrct`'s `default` target** has `flow: sim` (CAPI=2 had
  `default_tool: icarus`) but no top level, so it is still only useful as a
  dependency.

## How the diff was kept small

`fusesoc core migrate` re-indents every file to its own style. The conversion ran with
OpenPiton's 4-space indentation, then restored every line that changed only in
spacing: `name :` and `{file_type : xci}`, the multi-line `process_me` lists the dump
folded onto one line, the 3-space fileset blocks of five cores, trailing spaces and
a trailing whitespace-only line. Each restored file was checked to read as the same
YAML as the converter's output. The original cores have no comments; every `# FUSESOC-MIGRATE:`
comment was resolved or replaced by an explanation.

## Running it

With the CAPI=3 FuseSoC installed, from the OpenPiton root:

```sh
export PITON_ROOT=$PWD
source piton/piton_settings.bash    # DV_ROOT, and pyhp.py on PATH for the generators
fusesoc --cores-root piton run --setup --target=verify openpiton::sparc_tlu
fusesoc --cores-root piton run --build --target=pickle openpiton::chip \
    --iverilog_options="-DNO_SCAN"
```

CAPI=3 top levels reject the global `--tool`; the target's `flow:` picks the backend,
and flow options go after the core name.
