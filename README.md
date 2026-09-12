# AXI4 UVM Verification Environment

A complete UVM-based verification environment for an AMBA AXI4 slave connected to a synchronous single-port SRAM.

This project is the UVM implementation of the AXI4 memory verification project. It demonstrates a scalable, reusable verification architecture using UVM 1.2, SystemVerilog, constrained-random stimulus, functional coverage, SystemVerilog Assertions (SVA), scoreboards, and factory-based component overrides.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Design Under Test](#design-under-test)
- [Verification Objectives](#verification-objectives)
- [UVM Architecture](#uvm-architecture)
- [Project Structure](#project-structure)
- [Main UVM Components](#main-uvm-components)
- [AXI4 Verification](#axi4-verification)
- [Memory Verification](#memory-verification)
- [Constrained-Random Stimulus](#constrained-random-stimulus)
- [Functional Coverage](#functional-coverage)
- [Assertions](#assertions)
- [Scoreboards and Reference Models](#scoreboards-and-reference-models)
- [Factory Override](#factory-override)
- [Simulation and Automation](#simulation-and-automation)
- [Coverage Results](#coverage-results)
- [Tools and Technologies](#tools-and-technologies)
- [How to Run](#how-to-run)
- [Related Project](#related-project)
- [Authors](#authors)

---

## Project Overview

The Design Under Test (DUT) is an AMBA AXI4 slave bridge connected to an internal synchronous single-port SRAM.

The verification environment uses a dual-agent UVM architecture:

1. An active AXI agent generates and drives AXI4 transactions.
2. A passive memory agent monitors the internal memory interface.

This separation allows protocol-level behavior and storage-level behavior to be verified independently while remaining connected through the same DUT.

The project verifies normal operation, corner cases, protocol violations, reset behavior, memory consistency, and coverage closure.

---

## Design Under Test

The verified subsystem consists of:

- `axi4.v`: AXI4 slave interface/controller.
- `axi4_memory.v`: Synchronous single-port memory.

### Main Parameters

| Parameter | Value |
|---|---|
| Data Width | 32 bits |
| External Address Width | 16 bits |
| Internal Memory Depth | 1024 words |
| Memory Size | 4 KB |
| Word Size | 4 bytes |
| AXI Protocol | AMBA AXI4 |

The memory uses synchronous read behavior, and the AXI4 controller accommodates the read latency through an internal wait/control sequence.

Transactions that cross a 4 KB boundary are rejected with an AXI `SLVERR` response.

---

## Verification Objectives

The verification environment is designed to verify:

- AXI4 write transactions.
- AXI4 read transactions.
- AW, W, B, AR, and R channel handshaking.
- Burst transfers using `AWLEN`, `AWSIZE`, `ARLEN`, and `ARSIZE`.
- Correct write response generation.
- Correct read response generation.
- 4 KB boundary protection.
- Memory address range handling.
- Synchronous read latency.
- Reset behavior.
- Write priority when read and write requests overlap.
- Functional correctness through scoreboards.
- Protocol correctness through SVA assertions.
- Verification completeness through functional and code coverage.

---

## UVM Architecture

The environment uses a dual-agent topology.

```text
                              +----------------------+
                              |       axi_test       |
                              +----------+-----------+
                                         |
                              +----------v-----------+
                              |        axi_env       |
                              +----------+-----------+
                                         |
                 +-----------------------+-----------------------+
                 |                                               |
        +--------v---------+                             +---------v---------+
        |   Active AXI     |                             |   Passive Memory  |
        |      Agent       |                             |       Agent       |
        +--------+---------+                             +---------+---------+
                 |                                               |
        +--------v---------+                             +---------v---------+
        | Sequence         |                             | Memory Monitor     |
        | Sequencer        |                             | Memory Coverage    |
        | Driver           |                             | Memory Scoreboard   |
        | Monitor          |                             +---------------------+
        +--------+---------+
                 |
        +--------v---------+
        | AXI Coverage      |
        | AXI Scoreboard    |
        +--------+---------+
                 |
                 v
        +-------------------+
        | AXI Interface     |
        | Memory Interface  |
        +---------+---------+
                  |
                  v
        +-------------------+
        |       DUT         |
        |   axi4 + SRAM     |
        +-------------------+
```

### Active AXI Agent

The active AXI agent contains:

- `axi_sequence`
- `axi_sequencer`
- `axi_driver`
- `axi_driver_delay`
- `axi_monitor`

It generates randomized AXI transactions and drives them onto the AXI interface.

### Passive Memory Agent

The passive memory agent contains:

- `mem_monitor`
- `mem_scoreboard`
- `mem_coverage`

It observes internal memory activity without driving the DUT.

---

## Project Structure

```text
AXI4-UVM-Verification/
│
├── rtl/
│   ├── axi4.v
│   └── axi4_memory.v
│
├── tb/
│   ├── parameters_pkg.sv
│   ├── axi_para_pkg.sv
│   ├── common_cfg_pkg.sv
│   │
│   ├── AXI_intrf.sv
│   ├── mem_if.sv
│   ├── AXI_properties_pkg.sv
│   │
│   ├── a_transaction.sv
│   ├── a_sequencer.sv
│   ├── a_sequence.sv
│   ├── a_driver.sv
│   ├── a_delay_driver.sv
│   ├── a_monitor.sv
│   ├── a_agent.sv
│   ├── a_scoreboard.sv
│   ├── a_coverage.sv
│   │
│   ├── mem_transaction_pkg.sv
│   ├── mem_monitor_pkg.sv
│   ├── mem_agent_pkg.sv
│   ├── mem_scoreboard_pkg.sv
│   ├── mem_coverage_pkg.sv
│   │
│   ├── a_env.sv
│   ├── a_test.sv
│   └── a_top_tb.sv
│
├── scripts/
│   ├── run.do
│   ├── files.f
│   ├── dut_files.txt
│   └── tb_files.txt
│
├── coverage/
│   └── Reports/
│
├── docs/
│   └── Project_Report.pdf
│
└── README.md
```

> Keep the exact filenames in the repository consistent with the file lists used by `run.do`.

---

## Main UVM Components

| Component | Responsibility |
|---|---|
| `a_transaction` | Defines AXI transaction fields and constraints |
| `a_sequence` | Generates randomized AXI stimulus |
| `a_sequencer` | Sends sequence items to the driver |
| `a_driver` | Drives AXI signals according to the transaction |
| `a_delay_driver` | Alternative driver used for factory override testing |
| `a_monitor` | Observes AXI transactions |
| `a_agent` | Encapsulates AXI sequencer, driver, and monitor |
| `a_scoreboard` | Compares expected and actual AXI behavior |
| `a_coverage` | Collects AXI functional coverage |
| `mem_monitor_pkg` | Monitors internal memory transactions |
| `mem_scoreboard_pkg` | Checks memory consistency |
| `mem_coverage_pkg` | Collects memory functional coverage |
| `a_env` | Instantiates and connects the verification components |
| `a_test` | Configures the environment and starts the test |
| `a_top_tb` | Top-level simulation module |

---

## AXI4 Verification

The environment verifies all five AXI4 channels.

### Write Channels

- Write Address (`AW`)
- Write Data (`W`)
- Write Response (`B`)

### Read Channels

- Read Address (`AR`)
- Read Data (`R`)

### Protocol Features

The testbench checks:

- `VALID`/`READY` handshaking.
- Address and data channel sequencing.
- Burst length and size.
- `WLAST` and `RLAST` behavior.
- Write response generation.
- Read response generation.
- Error responses for invalid transactions.
- Reset and idle behavior.

---

## Memory Verification

The internal memory is monitored through a dedicated passive UVM agent.

The memory verification path checks:

- Write address decoding.
- Write data storage.
- Read address decoding.
- Read data correctness.
- Memory enable behavior.
- Read latency behavior.
- Address range handling.

The memory scoreboard maintains an expected memory model and compares DUT memory activity against expected results.

---

## Constrained-Random Stimulus

The AXI sequence generates randomized transactions to exercise normal and corner-case behavior.

Stimulus includes:

- Read transactions.
- Write transactions.
- Single-beat transfers.
- Short bursts.
- Medium bursts.
- Maximum-length bursts.
- Low, middle, and high address ranges.
- 4 KB boundary-safe transactions.
- Boundary-crossing invalid transactions.
- Randomized reset scenarios.

The constrained-random approach improves scenario diversity and reduces dependence on manually written directed tests.

---

## Functional Coverage

Functional coverage is implemented for both AXI and memory agents.

### AXI Coverage

Coverage includes:

- Operation type.
- Write address ranges.
- Read address ranges.
- Burst lengths.
- Burst sizes.
- Handshake conditions.
- Valid and ready behavior.
- Cross-coverage between addresses and burst lengths.

### Memory Coverage

Coverage includes:

- Read and write operations.
- Address ranges.
- Memory enable conditions.
- Data transfer scenarios.
- Corner-case memory accesses.

### Coverage Closure

The project defines 57 functional coverage bins for the main AXI coverage model.

The documented regression achieved closure of all defined functional coverage bins.

---

## Assertions

SystemVerilog Assertions are included in `AXI_properties_pkg.sv`.

The assertions monitor protocol invariants such as:

- `VALID` remains asserted until handshake.
- Correct channel sequencing.
- Proper burst termination.
- Correct `WLAST` and `RLAST` timing.
- Reset behavior.
- Valid response generation.
- Illegal 4 KB boundary crossing detection.

Assertions run concurrently with the UVM environment and provide immediate protocol-level feedback during simulation.

---

## Scoreboards and Reference Models

Two scoreboard paths are used.

### AXI Scoreboard

The AXI scoreboard checks:

- Expected AXI read responses.
- Expected AXI write responses.
- Burst transaction correctness.
- Data integrity.
- Error handling.

### Memory Scoreboard

The memory scoreboard maintains a reference memory model and compares internal memory operations against expected behavior.

This dual-scoreboard architecture helps isolate whether a failure originates from:

- AXI protocol/control logic.
- Memory access/storage logic.
- Transaction generation.
- Interface synchronization.

---

## Factory Override

The project includes a UVM factory override exercise.

The base AXI driver can be replaced at runtime with `a_delay_driver` without modifying higher-level sequences or the environment structure.

This demonstrates one of UVM's most important extensibility features:

```text
Base Driver
     |
     v
UVM Factory
     |
     v
Runtime Override
     |
     v
Delay Driver
```

The factory override confirms that verification components can be substituted dynamically while preserving the same test architecture.

---

## Simulation and Automation

The simulation workflow is automated using QuestaSim/ModelSim scripts.

### `run.do` Workflow

```text
1. Create the QuestaSim work library.
2. Compile parameters and configuration packages.
3. Compile AXI interfaces and assertions.
4. Compile memory-side UVM components.
5. Compile AXI-side UVM components.
6. Compile RTL and top-level testbench.
7. Start simulation with coverage enabled.
8. Run the selected UVM test.
9. Save the UCDB coverage database.
10. Generate an txt coverage report.
```

### Main Simulation Command

```text
vsim -voptargs=+acc -coverage work.top +UVM_TESTNAME=axi_test
```

### Coverage Generation

```text
coverage save -onexit regression.ucdb
vcover report -html regression.ucdb -htmldir cov_html_report
```

---

## How to Run

### Prerequisites

- QuestaSim or ModelSim.
- SystemVerilog support.
- UVM 1.2 package.
- All RTL and testbench files included in the file lists.

### Run the Project

Open QuestaSim in the project directory and execute:

```text
do scripts/run.do
```

If your script is stored in `do_file/`, use:

```text
do do_file/run.do
```

The script compiles the complete environment, runs the UVM test, and generates coverage results.

---

## Coverage Results

Coverage reports are stored in:

```text
coverage/Reports/
```

The project evaluates:

| Metric | Target |
|---|---:|
| AXI Functional Coverage | 100% defined bins |
| Memory Functional Coverage | 100% defined bins |
| RTL Line Coverage | Analyzed |
| RTL Toggle Coverage | Analyzed |
| RTL Branch Coverage | Analyzed |
| FSM Coverage | Analyzed |
| Assertion Coverage | Analyzed |

Some RTL coverage exclusions may be justified when branches or toggle bits are unreachable under the design's fixed parameters and valid operating constraints.

---

## Tools and Technologies

- SystemVerilog IEEE 1800-2017.
- Accellera UVM 1.2.
- QuestaSim .
- Constrained-Random Verification.
- Functional Coverage.
- Code Coverage.
- SystemVerilog Assertions (SVA).
- UVM Factory.
- UVM Agents, Drivers, Monitors, Sequences, and Scoreboards.
- Virtual Interfaces.
- Mailboxes and TLM communication.

---

## Related Project

This repository is the UVM implementation of the AXI4 verification project.

The related Class-Based SystemVerilog implementation is available in a separate repository.

The two projects demonstrate the progression from a manually structured SystemVerilog class-based environment to a standardized UVM architecture.

---

## Authors

**Mostafa Mohamed Farhan**  

Digital Verification Course — Project 2

---


