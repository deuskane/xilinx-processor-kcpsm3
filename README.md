<!--
  README GENERATION INSTRUCTIONS (for the next regeneration run)
  ----------------------------------------------------------------
  This README follows the common Asylum IP model. Regenerate it from the
  sources, never from the previous README text alone.

  Sources of truth (in priority order):
    1. hdl/*.vhd            : entities, generics, ports, packages
    2. hdl/csr/*.hjson      : register map (regtool); *_csr.md/.h are generated
    3. <IP>.core            : VLNV (name), filesets, targets, depends, revisions
    4. mk/targets.txt       : target list shown by `make help`; mk/defs.mk
    5. sim/, syn/, esw/, boards/ : testbenches, constraints, software
  Section order (keep it, same headings in every IP):
    CI badge / Title + one-line description + VLNV / Table of Contents /
    Introduction (Key Features) / Block Diagram / Top-Level (Parameters,
    Ports, Instantiation Example) / HDL Modules / Register Map /
    Verification / Synthesis / Design Notes (optional) /
    Directory Structure / Dependencies
  Rules:
    - Language: English. Tables: Parameters = Name|Type|Default|Description,
      Ports = Name|Direction|Type|Description (grouped by interface).
    - Register Map: link to the generated hdl/csr/<X>_csr.md (plus the
      .hjson source and _csr.h header); never copy register tables here.
    - Top-Level = sbi_* wrapper if present, else the entity used by the
      `default` target, else the main entity (libraries: list packages).
    - Write "This IP has no software-visible registers." / "No dedicated
      synthesis target ..." instead of removing a section.
    - Keep still-accurate hand-written content (ISA tables, results,
      images) in "Design Notes"; drop anything not backed by the sources.
    - Block diagram: doc/<NAME>.drawio (NAME = 4th field of the VLNV),
      top entity box with generics on top, inputs left, outputs right,
      bus interfaces as bold arrows, internal blocks colour-coded
      (CSR yellow, FIFO/memory green, core logic blue, external grey).
      Update it whenever ports/generics/sub-blocks change.
    - Do not edit generated files (hdl/csr/*_csr.*) or the CI badge URL.
-->
[![no CI](https://img.shields.io/badge/CI-no%20workflow-lightgrey)](https://github.com/deuskane/asylum-project)

# xilinx-processor-kcpsm3

**Original Xilinx PicoBlaze-3 (KCPSM3 v1.30) 8-bit soft processor, kept as the golden reference model of OpenBlaze8.**

VLNV: `xilinx:processor:kcpsm:1.30`

## Table of Contents

1. [Introduction](#introduction)
2. [Block Diagram](#block-diagram)
3. [Top-Level](#top-level)
4. [Directory Structure](#directory-structure)
5. [Dependencies](#dependencies)

## Introduction

This repository is an unmodified import of the Xilinx KCPSM3 macro ("Constant (K) Coded Programmable State Machine for Spartan-3 Devices", version 1.30 of 14th June 2004, Ken Chapman, Xilinx Ltd). It is third-party code distributed under the Xilinx notice found at the top of [hdl/kcpsm3.vhd](hdl/kcpsm3.vhd); only the FuseSoC core file is specific to the Asylum project.

In the Asylum project it is **not** used in any SoC. Its only user is the OpenBlaze8 testbench: the `files_sim` fileset of `asylum:processor:OpenBlaze8` depends on `xilinx:processor:kcpsm:1.30` and [sim/tb_OpenBlaze8.vhd](../asylum-processor-OpenBlaze8/sim/tb_OpenBlaze8.vhd) instantiates `kcpsm3(low_level_definition)` as `ref`, runs the same program ROM on both cores and checks cycle by cycle that `address`, `interrupt_ack`, `read_strobe`, `write_strobe`, `port_id` and `out_port` of OpenBlaze8 match the reference.

### Key Features

- 8-bit data path, 18-bit instructions, 10-bit program address (1024 instructions)
- 16 registers `s0`..`sF` (16 x 8 dual-port distributed RAM, `RAM16X1D`)
- 64-byte scratch-pad memory (`STORE` / `FETCH`, `RAM64X1S`)
- CALL/RETURN stack of 32 x 10-bit locations, nested calls up to 31 levels (`RAM32X1S`)
- ZERO and CARRY flags, shadow copies saved on interrupt
- One interrupt input with enable flip-flop and `interrupt_ack`
- 256 IO ports through `port_id` / `in_port` / `out_port` with read / write strobes
- Two clock cycles per instruction (internal T-state)
- Flat description using Xilinx unisim primitives (`LUT1..LUT4`, `FD*`, `MUXCY`, `XORCY`, `MUXF5`, `RAM*`); LUT contents given by `INIT` attributes and repeated in `generic map` inside `synthesis translate_off`
- Simulation-only process (between `translate off` / `translate on`) that disassembles the current instruction (`kcpsm3_opcode`), shows the flags / reset status (`kcpsm3_status`) and mirrors the content of every register and scratch-pad location in variables

## Block Diagram

Diagram: [doc/kcpsm.drawio](doc/kcpsm.drawio) (open with diagrams.net or the VS Code Draw.io extension).

- The control section generates the T-state and a synchronised internal reset, and decodes `instruction`.
- The 10-bit program counter drives `address`; it is loaded from the instruction (`JUMP` / `CALL`), from the stack (`RETURN`) or forced to the interrupt vector.
- The register bank provides the `sX` / `sY` operands; `sX` is output on `out_port` and the second operand (`sY` or constant `kk`) on `port_id`.
- The ALU combines logical, shift/rotate and arithmetic units through a pipelined output multiplexer; results go back to the register bank, the scratch pad (`STORE`) or the flags.
- The interrupt logic captures `interrupt`, saves ZERO / CARRY in shadow flip-flops and generates `interrupt_ack`; `read_strobe` / `write_strobe` are decoded from `INPUT` / `OUTPUT`.

## Top-Level

Top-level entity: **`kcpsm3`** ([hdl/kcpsm3.vhd](hdl/kcpsm3.vhd)), architecture `low_level_definition`. The core has no `logical_name`, so the entity is compiled in the default library (`work`), and it uses `library unisim; use unisim.vcomponents.all;`.

### Parameters

| Name | Type | Default | Description |
|------|------|---------|-------------|
| *(none)* | | | |

### Ports

#### Clock & Reset

| Name | Direction | Type | Description |
|------|-----------|------|-------------|
| `clk` | in | std_logic | Clock (rising edge) |
| `reset` | in | std_logic | Synchronous reset, active high |

#### Instruction

| Name | Direction | Type | Description |
|------|-----------|------|-------------|
| `address` | out | std_logic_vector(9 downto 0) | Program memory address |
| `instruction` | in | std_logic_vector(17 downto 0) | Instruction read from the program memory |

#### IOs

| Name | Direction | Type | Description |
|------|-----------|------|-------------|
| `port_id` | out | std_logic_vector(7 downto 0) | IO port address |
| `write_strobe` | out | std_logic | `out_port` valid (OUTPUT instruction) |
| `out_port` | out | std_logic_vector(7 downto 0) | Output data |
| `read_strobe` | out | std_logic | Input data sampled (INPUT instruction) |
| `in_port` | in | std_logic_vector(7 downto 0) | Input data |

#### Interrupts

| Name | Direction | Type | Description |
|------|-----------|------|-------------|
| `interrupt` | in | std_logic | Interrupt request |
| `interrupt_ack` | out | std_logic | Interrupt acknowledge |

### Instantiation Example

```vhdl
library unisim;
use     unisim.vcomponents.all;

  ins_kcpsm3 : entity work.kcpsm3(low_level_definition)
    port map
    ( address       => address
     ,instruction   => instruction
     ,port_id       => port_id
     ,write_strobe  => write_strobe
     ,out_port      => out_port
     ,read_strobe   => read_strobe
     ,in_port       => in_port
     ,interrupt     => interrupt
     ,interrupt_ack => interrupt_ack
     ,reset         => reset
     ,clk           => clk
    );
```

The program memory is not part of the core. In the Asylum flow it is generated by the `asylum:utils:generators` `pbcc_gen` generator (assembler type `kcpsm3`), which produces an `OpenBlaze8_ROM` entity with the same `address` / `instruction` interface.

## Directory Structure

```
xilinx-processor-kcpsm3/
├── kcpsm3.core             # FuseSoC core (xilinx:processor:kcpsm)
├── README.md
├── doc/
│   └── kcpsm.drawio        # Block diagram
└── hdl/
    └── kcpsm3.vhd          # Xilinx KCPSM3 v1.30 (entity kcpsm3)
```

There is no Makefile, `mk/` folder, testbench or CI workflow in this repository. The core is consumed through FuseSoC by depending on `>=xilinx:processor:kcpsm:1.30`; its single target `default` (toplevel `kcpsm3`, tool GHDL) only allows a standalone elaboration, e.g. `fusesoc --cores-root <asylum-project> run --target default xilinx:processor:kcpsm:1.30`.

## Dependencies

| Core | Used by (fileset) | Purpose |
|------|-------------------|---------|
| `xilinx:primitive:unisim` (`>=11.1`) | `files_hdl` | Xilinx primitives (`LUT*`, `FD*`, `MUXCY`, `XORCY`, `MUXF5`, `RAM16X1D`, `RAM32X1S`, `RAM64X1S`, `INV`) in library `unisim` |
