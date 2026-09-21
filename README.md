# FULLCUSTOM_IC_DESIGN
# INT4 Dot Product Accelerator for CNN
### Full-Custom IC Design of Sequential MAC and Parallel Dot Product Accelerator

## Overview

This project implements an **INT4 Dot Product Accelerator for CNN workloads** using a **Full-Custom IC Design methodology**.

The main objective is to design and compare two architectures that perform the same 4-term dot-product operation:

1. **Sequential MAC Architecture**
2. **Parallel Dot Product Accelerator Architecture**

The target operation is:

\[
Y = X_0W_0 + X_1W_1 + X_2W_2 + X_3W_3
\]

where each input \(X_i\) and weight \(W_i\) is represented as a **4-bit integer (INT4)**.

Rather than describing the architectures only at RTL or behavioral level, the arithmetic blocks were implemented using **transistor-level standard building blocks and full-custom layouts**.

The overall design flow includes:

- Transistor-level schematic design
- Pre-layout simulation
- Full-custom layout
- DRC verification
- LVS verification
- Parasitic extraction
- Post-layout simulation
- Architecture-level performance comparison

---

## Project Motivation

CNN convolution operations consist largely of repeated **Multiply-Accumulate (MAC)** and **Dot Product** operations.

For a 4-term dot product:

\[
Y = \sum_{i=0}^{3} X_iW_i
\]

the same arithmetic operation can be implemented using very different hardware architectures.

A **Sequential MAC** reuses a single multiplier and accumulator over multiple clock cycles, reducing hardware usage.

A **Parallel Dot Product Accelerator** uses multiple multipliers simultaneously and combines their outputs through an adder tree, increasing hardware parallelism.

This project investigates the architectural trade-off between:

> **Hardware reuse vs. parallel computation**

through transistor-level circuit implementation and full-custom physical design.

---

# Architecture

## Overall Architecture

The same four input/weight pairs are processed using two different architectures.

```text
Inputs
X0, X1, X2, X3
W0, W1, W2, W3
        │
        ├───────────────────────────┐
        │                           │
        ▼                           ▼
Sequential MAC              Parallel Accelerator
        │                           │
  1 × MUL4                     4 × MUL4
        │                           │
    RCA10                    2-Stage Adder Tree
        │                           │
    REG10                         Output
        │
    Feedback
```

The two architectures calculate the same dot-product result but differ significantly in hardware utilization, latency, and throughput.

---

# INT4 Arithmetic

Each multiplication uses two 4-bit operands:

```text
4-bit × 4-bit
```

which produces an:

```text
8-bit Product
```

For four unsigned INT4 multiplications:

\[
15 \times 15 = 225
\]

and therefore:

\[
4 \times 225 = 900
\]

A **10-bit datapath** is used for accumulation and adder-tree operations to prevent overflow.

---

# 4-bit Multiplier — MUL4

The fundamental arithmetic block of both architectures is the **4-bit × 4-bit unsigned multiplier**.

## Structure

The multiplier consists of:

```text
16 × AND
 8 × Full Adder
 4 × Half Adder
```

The multiplication process is divided into:

```text
4-bit Inputs
     │
     ▼
Partial Product Generation
     │
     │ 16 × AND
     ▼
Column Reduction
     │
     │ HA / FA
     ▼
Carry Propagation
     │
     ▼
8-bit Product P[7:0]
```

### Partial Product Generation

Each partial product is generated using an AND gate:

\[
PP_{ij} = A_i \cdot B_j
\]

For a 4 × 4 multiplier:

\[
4 \times 4 = 16
\]

partial products are generated.

### Column Reduction

The partial products are reduced using **Half Adders and Full Adders**.

### Final Output

```text
Input  : A[3:0], B[3:0]
Output : P[7:0]
```

---

# RCA10

A **10-bit Ripple Carry Adder (RCA10)** is used for accumulation and adder-tree operations.

## Structure

```text
FA0 → FA1 → FA2 → ... → FA9
 │     │                 │
C0    C1                C10
```

Each Full Adder transfers its carry output to the next stage.

The RCA10 consists of:

```text
10 × Full Adder
```

and produces:

```text
SUM[9:0]
```

The ripple-carry structure was selected because of its simple and regular transistor/layout structure.

---

# 8-bit to 10-bit Zero Extension

The MUL4 generates an 8-bit product while RCA10 operates on 10-bit operands.

Therefore, the multiplier output is zero-extended:

```text
MUL4 Output

P7 P6 P5 P4 P3 P2 P1 P0
│  │  │  │  │  │  │  │
▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼

RCA10 Input

0  0  P7 P6 P5 P4 P3 P2 P1 P0
```

Thus:

```text
B[9] = 0
B[8] = 0
B[7:0] = P[7:0]
```

The two MSBs are connected to **VSS (Logic 0)**.

---

# 10-bit Register — REG10

The Sequential MAC requires a register to store the accumulated result.

The register consists of:

```text
10 × D Flip-Flop
```

with a common:

```text
CLK
RESET
```

signal.

## Reset Structure

An NMOS-based reset path was added to initialize the register output.

The reset sequence initializes:

```text
ACC = 0
```

before normal MAC operation begins.

After reset is released, the DFFs store new accumulator values on the active clock edge.

---

# Sequential MAC Architecture

The Sequential MAC reuses a single multiplier and accumulator.

## Datapath

```text
Xi[3:0] ──────┐
              ▼
            MUL4
              │
Wi[3:0] ──────┘
              │
           P[7:0]
              │
       Zero Extension
              │
           [9:0]
              │
              ▼
            RCA10 ◀────────────┐
              │                │
              ▼                │
            REG10              │
              │                │
              └──── Feedback ──┘
```

The accumulator operation is:

\[
ACC_{i+1} = ACC_i + X_iW_i
\]

with:

\[
ACC_0 = 0
\]

For four input/weight pairs:

| Cycle | Input | Accumulator |
|---:|---|---|
| 1 | \(X_0,W_0\) | \(X_0W_0\) |
| 2 | \(X_1,W_1\) | \(X_0W_0+X_1W_1\) |
| 3 | \(X_2,W_2\) | \(X_0W_0+X_1W_1+X_2W_2\) |
| 4 | \(X_3,W_3\) | \(\sum_{i=0}^{3}X_iW_i\) |

Therefore:

```text
4 Products
     ↓
4 Clock Cycles
     ↓
Dot Product Result
```

The minimum clock period is constrained approximately by:

\[
T_{CLK} \geq t_{MUL4}+t_{RCA10}+t_{setup}
\]

and the four-term dot-product latency is approximately:

\[
T_{MAC} \approx 4T_{CLK}
\]

---

# Parallel Dot Product Accelerator

The parallel architecture computes all four multiplications simultaneously.

## Datapath

```text
X0,W0 ──► MUL4 ──► P0 ──┐
                          ├──► RCA10 ──► S01 ──┐
X1,W1 ──► MUL4 ──► P1 ──┘                    │
                                               ├──► RCA10 ──► Y
X2,W2 ──► MUL4 ──► P2 ──┐                    │
                          ├──► RCA10 ──► S23 ──┘
X3,W3 ──► MUL4 ──► P3 ──┘
```

The accelerator consists of:

```text
4 × MUL4
3 × RCA10
```

The four products are:

\[
P_0=X_0W_0
\]

\[
P_1=X_1W_1
\]

\[
P_2=X_2W_2
\]

\[
P_3=X_3W_3
\]

---

## 2-Stage Adder Tree

### Stage 1

Two additions are performed simultaneously:

\[
S_{01}=P_0+P_1
\]

\[
S_{23}=P_2+P_3
\]

```text
P0 ──┐
     ├── RCA10 ──► S01
P1 ──┘


P2 ──┐
     ├── RCA10 ──► S23
P3 ──┘
```

### Stage 2

The intermediate results are added:

\[
Y=S_{01}+S_{23}
\]

Therefore:

\[
Y=P_0+P_1+P_2+P_3
\]

and finally:

\[
Y=X_0W_0+X_1W_1+X_2W_2+X_3W_3
\]

The approximate combinational critical path is:

\[
T_{DP}\approx t_{MUL4}+2t_{RCA10}
\]

Unlike the Sequential MAC, the parallel accelerator does not require accumulator feedback for the four-term operation.

---

# Sequential vs Parallel Architecture

| Characteristic | Sequential MAC | Parallel Dot Product Accelerator |
|---|---|---|
| Multiplier | 1 × MUL4 | 4 × MUL4 |
| Addition | 1 × RCA10 | 3 × RCA10 |
| Register | REG10 | Not required for combinational datapath |
| Feedback | Required | Not required |
| Processing | Sequential | Parallel |
| Products evaluated simultaneously | 1 | 4 |
| 4-term operation | 4 cycles | Single combinational evaluation |
| Hardware reuse | High | Low |
| Hardware parallelism | Low | High |

The comparison demonstrates the fundamental architectural trade-off:

```text
Sequential MAC
→ Hardware reuse
→ Lower hardware requirement
→ Multiple clock cycles

Parallel Accelerator
→ Hardware duplication
→ Higher parallelism
→ Reduced computation latency
```

---

# Full-Custom IC Design Flow

The project follows a transistor-to-layout full-custom design methodology.

```text
Transistor-Level Schematic
          │
          ▼
Pre-Layout Simulation
          │
          ▼
Full-Custom Layout
          │
          ▼
DRC
          │
          ▼
LVS
          │
          ▼
PEX
          │
          ▼
Post-Layout Simulation
          │
          ▼
PPA Evaluation
```

The main verification stages are:

### DRC — Design Rule Check

Verifies that the physical layout satisfies process design rules.

### LVS — Layout Versus Schematic

Verifies that the extracted layout connectivity matches the designed schematic.

### PEX — Parasitic Extraction

Extracts parasitic resistance and capacitance from the physical layout for post-layout evaluation.

---

# Hierarchical Design

The design was constructed hierarchically from basic logic cells.

```text
Basic CMOS Gates
      │
      ├── AND
      ├── XOR
      ├── NAND
      ├── NOR
      └── INV
             │
             ▼
        HA / FA / DFF
             │
       ┌─────┴─────┐
       ▼           ▼
     MUL4        RCA10
       │           │
       └─────┬─────┘
             ▼
           REG10
             │
       ┌─────┴───────────┐
       ▼                 ▼
Sequential MAC    Parallel Accelerator
```

This hierarchical approach allows individual blocks to be independently designed, simulated, laid out, and physically verified before integration.

---

# Layout Strategy

The physical design uses hierarchical placement and multi-layer metal routing.

The routing strategy used in the project was:

| Metal | Main Usage |
|---|---|
| M1 | Cell internal connections, VDD/VSS rail, pin access |
| M2 | AND input buses and short/local signals |
| M3 | Crossing buses and vertical/inter-row connections |
| M4 | AND-to-adder partial-product routing and congestion relief |
| M5 | Reserved for complex inter-block crossings |
| M6 | Reserved for MAC/top-level global routing |
| M7 | Reserved for final congestion resolution or global nets |

The objective was not simply to use the highest available metal layer, but to use higher layers when routing congestion or long interconnects required them.

---

# Layout Optimization

One important issue encountered during multiplier layout was **routing congestion caused by input-net crossings**.

Because the two data inputs of a Half Adder are logically commutative:

\[
SUM=A\oplus B
\]

\[
CARRY=A\cdot B
\]

the input order can be exchanged without changing functionality.

Similarly, for a Full Adder:

\[
SUM=X\oplus Y\oplus C_{in}
\]

\[
CARRY=XY+(X\oplus Y)C_{in}
\]

the `X` and `Y` inputs can be exchanged while keeping `CIN` unchanged.

This property was used to optimize physical pin mapping.

```text
Before

Net A ─────╲
            ╲
             ╳──── Cell
            ╱
Net B ─────╱


After

Net A ─────────── Cell
Net B ─────────── Cell
```

The optimized mapping reduced:

- Metal crossing
- Routing length
- Routing congestion

while preserving the same logical functionality.

The modified connectivity was verified using LVS.

---

# Simulation

## Multiplier

The 4-bit multiplier was functionally verified using transient simulation.

The measured critical-path propagation delay was approximately:

```text
MUL4 Critical Path Delay ≈ 0.385 ns
```

---

## Sequential MAC

The MAC was tested using sequential multiplication and accumulation.

A representative accumulation sequence was:

```text
CLK 1 → ACC = 15
CLK 2 → ACC = 29
CLK 3 → ACC = 254
CLK 4 → ACC = 255
```

Each multiplication result is accumulated in a different clock cycle because the updated accumulator value must first be stored in the register and then returned through the feedback path.

---

## Parallel Dot Product Accelerator

The four multipliers operate simultaneously and the resulting products propagate through the two-stage RCA10 adder tree.

The measured propagation delay reported for the tested accelerator was approximately:

```text
tpd ≈ 0.8142 ns
```

The computation is performed without sequential accumulator feedback.

---

# Experimental Comparison

For the test configuration reported in the project:

| Metric | Sequential MAC | Parallel Dot Product Accelerator |
|---|---:|---:|
| Computation | Sequential | Parallel |
| Multiplications per evaluation/cycle | 1 | 4 |
| Processing of 4 products | 4 cycles | 1 combinational evaluation |
| Reported single-cycle / evaluation delay | 1.750 ns | 0.814 ns |
| Reported 4-operation completion time | 5.330 ns | 0.814 ns |

The measured values demonstrate the latency reduction obtained by increasing hardware parallelism.

The comparison should be interpreted together with the hardware cost:

```text
Sequential MAC
1 × MUL4
1 × RCA10
1 × REG10
      │
      └── Hardware reuse


Parallel Accelerator
4 × MUL4
3 × RCA10
      │
      └── Hardware parallelism
```

Therefore, the parallel architecture reduces computation latency by using additional arithmetic hardware, while the sequential architecture reuses hardware across multiple cycles.

---

# PPA Evaluation

The project considers the following architecture evaluation metrics:

## Area

Physical layout area:

\[
Area=W\times H
\]

## Delay

Propagation delay is measured using transient simulation and input/output threshold crossings.

## Power

Average power can be obtained from the supply current:

\[
P_{avg}
=
\frac{1}{T}
\int_0^T V_{DD}I_{DD}(t)\,dt
\]

## Throughput

\[
Throughput
=
\frac{Operations}{Time}
\]

Increasing parallelism increases the number of operations that can be processed simultaneously.

## Energy per Operation

\[
E_{op}
=
\frac{E_{total}}{N_{ops}}
\]

or approximately:

\[
E_{op}
\approx
P_{avg}\times T_{operation}
\]

These metrics allow the two architectures to be compared beyond propagation delay alone.

---

# Scalability

The architecture can be generalized by increasing the number of parallel processing elements.

For an N-term dot product:

```text
1 PE
→ 1 MAC operation / cycle

2 PE
→ 2 MAC operations / cycle

4 PE
→ 4 MAC operations / cycle

8 PE
→ 8 MAC operations / cycle
```

Increasing the number of processing elements reduces the number of cycles required to process a larger dot product.

However, increasing parallelism also increases:

- Multiplier count
- Adder count
- Routing complexity
- Layout area
- Power consumption

Therefore, accelerator design requires balancing:

\[
\boxed{
Area \leftrightarrow Power \leftrightarrow Performance
}
\]

---

# Key Design Trade-Off

The main architectural observation of this project is:

```text
              Hardware Resources
                     ▲
                     │
Parallel Accelerator│
                     │
                     │
Sequential MAC      │
                     └────────────────► Parallelism
```

The Sequential MAC emphasizes **resource reuse**, while the Parallel Dot Product Accelerator emphasizes **computation parallelism**.

### Sequential MAC

**Characteristics**

- Single multiplier reuse
- Single RCA10 reuse
- Register-based accumulation
- Feedback datapath
- Multiple clock cycles per dot product

### Parallel Dot Product Accelerator

**Characteristics**

- Four simultaneous multipliers
- Two-stage adder tree
- No accumulator feedback for the 4-term computation
- Single combinational evaluation
- Increased arithmetic hardware

The project therefore demonstrates at the full-custom circuit level that increasing parallelism can reduce computation latency, but the resulting area and power overhead must also be considered.

---

# Project Structure

```text
INT4-Dot-Product-Accelerator/
│
├── Basic_Cells/
│   ├── INV/
│   ├── NAND/
│   ├── NOR/
│   ├── AND/
│   ├── XOR/
│   ├── HA/
│   ├── FA/
│   └── DFF/
│
├── Multiplier/
│   └── MUL4/
│
├── Adder/
│   └── RCA10/
│
├── Register/
│   └── REG10/
│
├── Sequential_MAC/
│
├── Parallel_Dot_Product_Accelerator/
│
├── Simulation/
│
├── Layout/
│
├── DRC_LVS/
│
└── README.md
```

---

# Tools

- **Cadence Virtuoso**
  - Schematic Editor
  - Layout Editor
  - ADE
- **Spectre**
  - Transient simulation
  - Timing analysis
  - Power analysis
- **DRC / LVS**
  - Physical design verification
- **PEX**
  - Parasitic extraction and post-layout evaluation

---

# Project Summary

This project implemented an **INT4 CNN Dot Product Accelerator using a Full-Custom IC Design flow** and compared two implementations of the same arithmetic operation:

```text
Sequential MAC
vs.
Parallel Dot Product Accelerator
```

The project was developed hierarchically from transistor-level logic cells through:

```text
Basic Gates
    ↓
HA / FA / DFF
    ↓
MUL4 / RCA10 / REG10
    ↓
Sequential MAC
    +
Parallel Dot Product Accelerator
```

The Sequential MAC computes products over multiple clock cycles using accumulator feedback, while the Parallel Dot Product Accelerator computes four products simultaneously and reduces them using a two-stage adder tree.

Through schematic design, simulation, full-custom layout, and physical verification, the project demonstrates the fundamental accelerator-design trade-off between **hardware reuse and parallelism**.

> **More parallel hardware can reduce computation latency and increase throughput, while sequential hardware reuse can reduce implementation resources.**

The appropriate architecture therefore depends on the target **Area, Power, Performance, Throughput, and Energy-per-Operation requirements**.
