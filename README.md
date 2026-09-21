# INT4 Dot Product Accelerator for CNN

## Full-Custom IC Design: Sequential MAC vs Parallel Dot Product Accelerator

## 1. Project Overview

This project implements an **INT4 Dot Product Accelerator for CNN workloads** using a **Full-Custom IC Design methodology**.

The main objective is to design and compare two architectures that perform the same 4-term dot-product operation:

- Sequential MAC
- Parallel Dot Product Accelerator

The target operation is:

```text
Y = X0*W0 + X1*W1 + X2*W2 + X3*W3
```

where each input `Xi` and weight `Wi` is represented as a **4-bit integer (INT4)**.

The project includes:

- Transistor-level schematic design
- Pre-layout simulation
- Full-custom layout
- DRC verification
- LVS verification
- Parasitic extraction
- Post-layout simulation
- Architecture-level performance comparison

---

## 2. Project Motivation

CNN convolution operations consist largely of repeated **Multiply-Accumulate (MAC)** and **Dot Product** operations.

For a 4-term dot product:

```text
Y = SUM(Xi*Wi), i = 0 to 3
```

or equivalently:

```text
Y = X0*W0 + X1*W1 + X2*W2 + X3*W3
```

the same arithmetic operation can be implemented using different hardware architectures.

A **Sequential MAC** reuses a single multiplier and accumulator over multiple clock cycles.

A **Parallel Dot Product Accelerator** uses multiple multipliers simultaneously and combines their outputs through an adder tree.

The project therefore investigates the trade-off between:

```text
Hardware Reuse <-----> Parallel Computation
```

---

## 3. Overall Architecture

The same four input and weight pairs are processed using two different architectures.

```text
Inputs
X0, X1, X2, X3
W0, W1, W2, W3
        |
        +---------------------------+
        |                           |
        v                           v
Sequential MAC              Parallel Accelerator
        |                           |
     1 x MUL4                    4 x MUL4
        |                           |
      RCA10                  2-Stage Adder Tree
        |                           |
      REG10                       Output
        |
     Feedback
```

Both architectures calculate:

```text
Y = X0*W0 + X1*W1 + X2*W2 + X3*W3
```

but differ significantly in hardware utilization, latency, and throughput.

---

## 4. INT4 Arithmetic

Each multiplication uses two 4-bit unsigned operands:

```text
4-bit x 4-bit -> 8-bit product
```

The maximum 4-bit unsigned value is:

```text
15
```

Therefore, the maximum multiplication result is:

```text
15 x 15 = 225
```

For four products:

```text
Maximum Dot Product
= 4 x 225
= 900
```

The required output width can be determined from:

```text
2^9  = 512
2^10 = 1024

512 < 900 < 1024
```

Therefore, a **10-bit datapath** is used for accumulation and adder-tree operations.

---

## 5. 4-bit Multiplier - MUL4

The fundamental arithmetic block used in both architectures is a **4-bit x 4-bit unsigned multiplier**.

### 5.1 Structure

The multiplier consists of:

```text
16 x AND
 8 x Full Adder
 4 x Half Adder
```

The multiplication process is:

```text
4-bit Input A
4-bit Input B
      |
      v
Partial Product Generation
      |
      | 16 x AND
      v
Column Reduction
      |
      | HA / FA
      v
Carry Propagation
      |
      v
8-bit Product P[7:0]
```

### 5.2 Partial Product Generation

Each partial product is generated using an AND operation:

```text
PP(i,j) = A(i) AND B(j)
```

For a 4-bit x 4-bit multiplier:

```text
4 x 4 = 16 partial products
```

are generated.

### 5.3 Multiplier I/O

```text
Input A : A[3:0]
Input B : B[3:0]

Output  : P[7:0]
```

---

## 6. RCA10

A **10-bit Ripple Carry Adder (RCA10)** is used for accumulation and adder-tree operations.

### 6.1 Structure

The RCA10 consists of:

```text
10 x Full Adder
```

connected in series:

```text
FA0 -> FA1 -> FA2 -> FA3 -> ... -> FA9
 |      |      |                   |
 C1     C2     C3                  COUT
```

Each Full Adder transfers its carry output to the carry input of the next stage.

### 6.2 Operation

For each bit:

```text
SUM(i) = A(i) XOR B(i) XOR CIN(i)
```

and the carry is propagated to the next stage.

The final output is:

```text
SUM[9:0]
```

---

## 7. 8-bit to 10-bit Zero Extension

MUL4 generates an 8-bit product while RCA10 operates on 10-bit operands.

Therefore, the multiplier output is zero-extended.

```text
MUL4 Output

P7 P6 P5 P4 P3 P2 P1 P0
```

becomes:

```text
RCA10 Input

0  0  P7 P6 P5 P4 P3 P2 P1 P0
```

The connection is:

```text
B[9]   = VSS
B[8]   = VSS
B[7:0] = P[7:0]
```

Since the multiplication is unsigned, the upper two bits are connected to logic 0.

---

## 8. 10-bit Register - REG10

The Sequential MAC requires a register to store the accumulated result.

REG10 consists of:

```text
10 x D Flip-Flop
```

with common:

```text
CLK
RESET
```

signals.

The register stores:

```text
ACC[9:0]
```

and returns the stored value to RCA10 through the feedback path.

---

## 9. Register Reset

An NMOS-based reset path is used to initialize the register output.

Before normal MAC operation:

```text
ACC = 0
```

is established.

The basic sequence is:

```text
RESET asserted
      |
      v
Q initialized to 0
      |
      v
RESET released
      |
      v
Normal DFF operation
```

After reset is released, each DFF stores new accumulator data according to the clock.

---

## 10. Sequential MAC Architecture

The Sequential MAC reuses one multiplier, one RCA10, and one 10-bit register.

### 10.1 Datapath

```text
Xi[3:0] -----+
             |
             v
           MUL4
             ^
             |
Wi[3:0] -----+
             |
          P[7:0]
             |
      Zero Extension
             |
          P[9:0]
             |
             v
           RCA10 <-------------+
             |                 |
             v                 |
           REG10               |
             |                 |
             +---- Feedback ---+
```

### 10.2 Accumulation Operation

The accumulator performs:

```text
ACC_next = ACC_current + Xi*Wi
```

with:

```text
ACC_initial = 0
```

For four products:

```text
Cycle 1:
ACC = X0*W0

Cycle 2:
ACC = X0*W0 + X1*W1

Cycle 3:
ACC = X0*W0 + X1*W1 + X2*W2

Cycle 4:
ACC = X0*W0 + X1*W1 + X2*W2 + X3*W3
```

Therefore:

```text
Final ACC = Dot Product Result
```

### 10.3 Sequential Operation

| Cycle | Input | Accumulated Result |
|---:|---|---|
| 1 | X0, W0 | X0*W0 |
| 2 | X1, W1 | X0*W0 + X1*W1 |
| 3 | X2, W2 | X0*W0 + X1*W1 + X2*W2 |
| 4 | X3, W3 | X0*W0 + X1*W1 + X2*W2 + X3*W3 |

The minimum clock period is approximately constrained by:

```text
T_CLK >= t_MUL4 + t_RCA10 + t_setup
```

The four-term latency is approximately:

```text
T_MAC ~= 4 x T_CLK
```

---

## 11. Parallel Dot Product Accelerator

The Parallel Dot Product Accelerator computes all four multiplications simultaneously.

### 11.1 Hardware Structure

The accelerator consists of:

```text
4 x MUL4
3 x RCA10
```

### 11.2 Datapath

```text
X0,W0 ---> MUL4 ---> P0 ---+
                            |
                            +---> RCA10 ---> S01 ---+
                            |                       |
X1,W1 ---> MUL4 ---> P1 ---+                       |
                                                    +---> RCA10 ---> Y
X2,W2 ---> MUL4 ---> P2 ---+                       |
                            |                       |
                            +---> RCA10 ---> S23 ---+
                            |
X3,W3 ---> MUL4 ---> P3 ---+
```

All four products are generated simultaneously:

```text
P0 = X0*W0
P1 = X1*W1
P2 = X2*W2
P3 = X3*W3
```

---

## 12. 2-Stage Adder Tree

The four multiplier outputs are reduced using a two-stage RCA10 adder tree.

### 12.1 Stage 1

Two additions are performed simultaneously:

```text
S01 = P0 + P1
S23 = P2 + P3
```

Structure:

```text
P0 ---+
      +---> RCA10 ---> S01
P1 ---+


P2 ---+
      +---> RCA10 ---> S23
P3 ---+
```

### 12.2 Stage 2

The two intermediate results are added:

```text
Y = S01 + S23
```

Therefore:

```text
Y = P0 + P1 + P2 + P3
```

and:

```text
Y = X0*W0 + X1*W1 + X2*W2 + X3*W3
```

The approximate critical path is:

```text
T_DP ~= t_MUL4 + 2*t_RCA10
```

Unlike the Sequential MAC, no accumulator feedback is required for the 4-term dot-product computation.

---

## 13. Sequential MAC vs Parallel Accelerator

| Characteristic | Sequential MAC | Parallel Dot Product Accelerator |
|---|---|---|
| MUL4 | 1 | 4 |
| RCA10 | 1 | 3 |
| REG10 | 1 | Not required in combinational datapath |
| Feedback | Required | Not required |
| Processing | Sequential | Parallel |
| Simultaneous multiplications | 1 | 4 |
| 4-term operation | 4 cycles | 1 combinational evaluation |
| Hardware reuse | High | Low |
| Parallelism | Low | High |

The fundamental architectural trade-off is:

```text
Sequential MAC
    |
    +--> Hardware reuse
    +--> Smaller arithmetic hardware
    +--> Multiple clock cycles


Parallel Accelerator
    |
    +--> Hardware parallelism
    +--> More arithmetic hardware
    +--> Reduced computation latency
```

---

## 14. Full-Custom IC Design Flow

The project follows a full-custom design flow from circuit implementation to physical verification.

```text
Transistor-Level Schematic
          |
          v
Pre-Layout Simulation
          |
          v
Full-Custom Layout
          |
          v
DRC
          |
          v
LVS
          |
          v
PEX
          |
          v
Post-Layout Simulation
          |
          v
PPA Evaluation
```

### DRC

**Design Rule Check** verifies that the physical layout satisfies the process design rules.

### LVS

**Layout Versus Schematic** verifies that the extracted layout connectivity matches the schematic.

### PEX

**Parasitic Extraction** extracts parasitic resistance and capacitance for post-layout evaluation.

---

## 15. Hierarchical Full-Custom Design

The complete accelerator was designed hierarchically.

```text
Basic CMOS Gates
        |
        +--> INV
        +--> NAND
        +--> NOR
        +--> AND
        +--> XOR
        |
        v
     HA / FA
        |
        +----------------+
        |                |
        v                v
      MUL4             RCA10
        |                |
        +--------+-------+
                 |
               REG10
                 |
        +--------+---------+
        |                  |
        v                  v
Sequential MAC     Parallel Accelerator
```

Each block can be independently:

```text
Designed
   ->
Simulated
   ->
Laid Out
   ->
DRC Checked
   ->
LVS Verified
```

before top-level integration.

---

## 16. Layout Routing Strategy

The physical implementation uses hierarchical placement and multi-layer metal routing.

| Metal Layer | Main Usage |
|---|---|
| M1 | Cell internal connection, VDD/VSS rail, pin access |
| M2 | AND input bus and short/local signals |
| M3 | Crossing bus and vertical/inter-row connection |
| M4 | AND-to-Adder partial-product routing and congestion relief |
| M5 | Complex inter-block crossing reserve |
| M6 | MAC/top-level global routing reserve |
| M7 | Final congestion resolution or global-net reserve |

Higher metal layers were introduced when long-distance routing or routing congestion required additional routing resources.

---

## 17. Layout Mapping Optimization

One issue encountered during multiplier layout was routing congestion caused by input-net crossings.

For a Half Adder:

```text
SUM   = A XOR B
CARRY = A AND B
```

Because the A and B inputs are commutative:

```text
HA(A,B) = HA(B,A)
```

the input mapping can be exchanged without changing the logical function.

For a Full Adder:

```text
SUM = X XOR Y XOR CIN

CARRY = (X AND Y) OR ((X XOR Y) AND CIN)
```

X and Y can be exchanged:

```text
FA(X,Y,CIN) = FA(Y,X,CIN)
```

while `CIN` remains unchanged.

This property was used to optimize the physical pin mapping.

```text
Before

Net A ------\
             \
              X---- Cell
             /
Net B ------/


After

Net A ------------ Cell
Net B ------------ Cell
```

The optimized mapping reduces:

```text
Metal Crossing
Routing Length
Routing Congestion
```

while preserving logical functionality.

The modified connectivity was verified using LVS.

---

## 18. Simulation Results

### 18.1 MUL4

The 4-bit multiplier was verified using transient simulation.

The measured critical-path propagation delay was approximately:

```text
t_MUL4 ~= 0.385 ns
```

---

### 18.2 Sequential MAC

A representative accumulation sequence was:

```text
CLK 1 -> ACC = 15
CLK 2 -> ACC = 29
CLK 3 -> ACC = 254
CLK 4 -> ACC = 255
```

Each multiplication result is processed in a different clock cycle because the updated accumulator value must be stored and returned through the feedback path.

---

### 18.3 Parallel Dot Product Accelerator

The four MUL4 blocks operate simultaneously.

Their outputs propagate through the two-stage RCA10 adder tree.

The measured propagation delay was approximately:

```text
t_DP ~= 0.8142 ns
```

---

## 19. Experimental Comparison

The reported test results are:

| Metric | Sequential MAC | Parallel Dot Product Accelerator |
|---|---:|---:|
| Operation Structure | Sequential | Parallel |
| Multiplications per Cycle/Evaluation | 1 | 4 |
| Processing of 4 Products | 4 Cycles | 1 Evaluation |
| Reported Cycle/Evaluation Delay | 1.750 ns | 0.814 ns |
| Reported 4-Operation Completion Time | 5.330 ns | 0.814 ns |

Using the reported completion times:

```text
Execution-Time Ratio
= 5.330 / 0.8142
~= 6.55
```

Therefore, for the reported test case:

```text
Parallel Accelerator completion time
~= 1 / 6.5 of Sequential MAC completion time
```

The performance improvement is obtained by increasing hardware parallelism.

---

## 20. PPA Evaluation Metrics

The two architectures can be evaluated using:

```text
PPA + Throughput + Energy/Operation
```

### 20.1 Area

Layout area:

```text
Area = Width x Height
```

---

### 20.2 Propagation Delay

Propagation delay can be measured using the 50-percent crossing points of the input and output.

```text
t_pd = t_output(50%) - t_input(50%)
```

Measured values reported in the project include:

```text
MUL4:
t_pd ~= 0.385 ns

Parallel Dot Product Accelerator:
t_pd ~= 0.8142 ns
```

---

### 20.3 Average Power

Average power is obtained from the supply voltage and average supply current.

For constant VDD:

```text
P_avg = VDD x I_avg
```

where:

```text
I_avg = average supply current over the measurement interval
```

---

### 20.4 Throughput

Throughput is defined as:

```text
Throughput = Number of Operations / Execution Time
```

The Parallel Accelerator performs four multiplications simultaneously, whereas the Sequential MAC performs one multiplication per accumulation cycle.

---

### 20.5 Energy per Operation

Energy can be approximated as:

```text
E_total ~= P_avg x T_execution
```

Energy per operation is then:

```text
E_op = E_total / N_ops
```

Therefore:

```text
E_op ~= (P_avg x T_execution) / N_ops
```

---

## 21. Scalability

The architecture can be extended by increasing the number of processing elements.

```text
1 PE -> 1 MAC operation / cycle
2 PE -> 2 MAC operations / cycle
4 PE -> 4 MAC operations / cycle
8 PE -> 8 MAC operations / cycle
```

For `N` operations and `P` processing elements, the ideal number of cycles is approximately:

```text
Cycles = ceil(N / P)
```

For example, for an 8-term dot product:

```text
1 PE -> 8 cycles
2 PE -> 4 cycles
4 PE -> 2 cycles
8 PE -> 1 cycle
```

Increasing parallelism can improve throughput but also increases:

- Multiplier count
- Adder count
- Routing complexity
- Layout area
- Power consumption

Therefore, architecture selection requires balancing:

```text
Area <-----> Power <-----> Performance
```

---

## 22. Key Design Trade-Off

The two architectures represent different optimization directions.

### Sequential MAC

```text
MUL4
 |
 v
RCA10
 |
 v
REG10
 |
 +------ Feedback ------+
```

Main characteristics:

- Single multiplier reuse
- Single RCA10 reuse
- Register-based accumulation
- Feedback datapath
- Four cycles for a 4-term dot product

### Parallel Dot Product Accelerator

```text
MUL4 ---+
MUL4 ---+
        +--> 2-Stage Adder Tree --> Y
MUL4 ---+
MUL4 ---+
```

Main characteristics:

- Four simultaneous multipliers
- Three RCA10 blocks
- Two-stage adder tree
- No sequential accumulator feedback
- Single combinational evaluation for the 4-term dot product

The fundamental design trade-off is:

```text
Hardware Reuse <-----> Hardware Parallelism
```

The final architecture should be evaluated based on:

```text
Area
Power
Delay
Throughput
Energy per Operation
```

---

## 23. Project Structure

```text
INT4-Dot-Product-Accelerator/
|
+-- Basic_Cells/
|   +-- INV/
|   +-- NAND/
|   +-- NOR/
|   +-- AND/
|   +-- XOR/
|   +-- HA/
|   +-- FA/
|   +-- DFF/
|
+-- Multiplier/
|   +-- MUL4/
|
+-- Adder/
|   +-- RCA10/
|
+-- Register/
|   +-- REG10/
|
+-- Sequential_MAC/
|
+-- Parallel_Dot_Product_Accelerator/
|
+-- Simulation/
|
+-- Layout/
|
+-- DRC_LVS/
|
+-- README.md
```

---

## 24. Tools

- Cadence Virtuoso
  - Schematic Editor
  - Layout Editor
  - ADE
- Spectre
  - Transient Simulation
  - Timing Analysis
  - Power Analysis
- DRC
- LVS
- PEX

---

## 25. Conclusion

This project implemented an **INT4 CNN Dot Product Accelerator using a Full-Custom IC Design flow** and compared two architectures:

```text
Sequential MAC
        vs
Parallel Dot Product Accelerator
```

The design hierarchy is:

```text
Basic Gates
    |
    v
HA / FA / DFF
    |
    v
MUL4 / RCA10 / REG10
    |
    +----------------------+
    |                      |
    v                      v
Sequential MAC      Parallel Accelerator
```

The Sequential MAC computes products over multiple clock cycles using accumulator feedback.

The Parallel Dot Product Accelerator computes four products simultaneously and combines them using a two-stage adder tree.

The project demonstrates the fundamental hardware-architecture trade-off:

```text
Sequential Architecture
-> Hardware Reuse
-> Multiple Clock Cycles

Parallel Architecture
-> Hardware Parallelism
-> Reduced Computation Latency
```

The final architecture should therefore be selected based on the target requirements for:

```text
Area
Power
Performance
Throughput
Energy per Operation
```
