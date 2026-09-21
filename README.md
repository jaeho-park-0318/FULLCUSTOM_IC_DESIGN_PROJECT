# FULLCUSTOM_IC_DESIGN_PROJECT
## Project Motivation

CNN convolution operations consist largely of repeated **Multiply-Accumulate (MAC)** and **Dot Product** operations.

For a 4-term dot product:

$$
Y = \sum_{i=0}^{3} X_i W_i
$$

the same arithmetic operation can be implemented using very different hardware architectures.

A **Sequential MAC** reuses a single multiplier and accumulator over multiple clock cycles, reducing hardware usage.

A **Parallel Dot Product Accelerator** uses multiple multipliers simultaneously and combines their outputs through an adder tree, increasing hardware parallelism.

This project investigates the architectural trade-off between:

> **Hardware reuse vs. parallel computation**

through transistor-level circuit implementation and full-custom physical design.

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

For unsigned INT4 operands, the maximum multiplication result is:

$$
15 \times 15 = 225
$$

For four products:

$$
Y_{\max} = 4 \times 225 = 900
$$

Since:

$$
2^9 = 512 < 900 < 1024 = 2^{10}
$$

a **10-bit datapath** is used for accumulation and adder-tree operations.

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

### Partial Product Generation

Each partial product is generated using an AND gate:

$$
PP_{ij} = A_i \cdot B_j
$$

For a 4 × 4 multiplier:

$$
4 \times 4 = 16
$$

partial products are generated.

The final output is:

```text
Input  : A[3:0], B[3:0]
Output : P[7:0]
```

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
           P[9:0]
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

$$
ACC_{i+1} = ACC_i + X_i W_i
$$

with the initial condition:

$$
ACC_0 = 0
$$

For four input/weight pairs:

| Cycle | Input | Accumulator |
|---:|---|---|
| 1 | `X0, W0` | $X_0W_0$ |
| 2 | `X1, W1` | $X_0W_0 + X_1W_1$ |
| 3 | `X2, W2` | $X_0W_0 + X_1W_1 + X_2W_2$ |
| 4 | `X3, W3` | $X_0W_0 + X_1W_1 + X_2W_2 + X_3W_3$ |

After four cycles:

$$
Y = \sum_{i=0}^{3} X_i W_i
$$

The minimum clock period is constrained approximately by:

$$
T_{CLK} \ge t_{MUL4} + t_{RCA10} + t_{setup}
$$

Therefore, the latency of a four-term Sequential MAC is approximately:

$$
T_{MAC} \approx 4T_{CLK}
$$

---

# Parallel Dot Product Accelerator

The Parallel Dot Product Accelerator computes all four multiplications simultaneously.

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

The four products are calculated simultaneously:

$$
P_0 = X_0W_0
$$

$$
P_1 = X_1W_1
$$

$$
P_2 = X_2W_2
$$

$$
P_3 = X_3W_3
$$

---

## 2-Stage Adder Tree

### Stage 1

The first two RCA10 blocks operate in parallel:

$$
S_{01} = P_0 + P_1
$$

$$
S_{23} = P_2 + P_3
$$

```text
P0 ──┐
     ├── RCA10 ──► S01
P1 ──┘

P2 ──┐
     ├── RCA10 ──► S23
P3 ──┘
```

### Stage 2

The final RCA10 combines the two intermediate results:

$$
Y = S_{01} + S_{23}
$$

Therefore:

$$
Y = P_0 + P_1 + P_2 + P_3
$$

and:

$$
\boxed{
Y = X_0W_0 + X_1W_1 + X_2W_2 + X_3W_3
}
$$

The approximate combinational critical-path delay is:

$$
T_{DP} \approx t_{MUL4} + 2t_{RCA10}
$$

Unlike the Sequential MAC, the Parallel Dot Product Accelerator does not require accumulator feedback for the four-term operation.

---

# Layout Optimization

One important issue encountered during multiplier layout was **routing congestion caused by input-net crossings**.

Because the two inputs of a Half Adder are commutative:

$$
SUM = A \oplus B
$$

$$
CARRY = A \cdot B
$$

the input order can be exchanged without changing functionality:

$$
HA(A,B) = HA(B,A)
$$

For a Full Adder:

$$
SUM = X \oplus Y \oplus C_{in}
$$

$$
CARRY = X \cdot Y + (X \oplus Y)\cdot C_{in}
$$

the `X` and `Y` inputs can be exchanged:

$$
FA(X,Y,C_{in}) = FA(Y,X,C_{in})
$$

while `CIN` remains unchanged.

This property was used to optimize physical pin mapping while preserving logical equivalence.

---

# PPA Evaluation

The architecture is evaluated using **Area, Delay, Power, Throughput, and Energy per Operation**.

## Area

Physical layout area is calculated as:

$$
Area = W \times H
$$

where `W` and `H` represent the width and height of the layout.

---

## Delay

Propagation delay is measured between the 50% voltage crossing points of the input and output signals:

$$
t_{pd} = t_{out,50\%} - t_{in,50\%}
$$

The measured MUL4 critical-path delay was approximately:

$$
t_{MUL4} \approx 0.385\;ns
$$

The measured Parallel Dot Product Accelerator delay was approximately:

$$
t_{DP} \approx 0.8142\;ns
$$

---

## Power

Average power can be calculated from the supply voltage and current:

$$
P_{avg}
=
\frac{1}{T}
\int_{0}^{T}
V_{DD} I_{DD}(t)\,dt
$$

For a constant supply voltage:

$$
P_{avg}
=
V_{DD}
\left(
\frac{1}{T}
\int_{0}^{T} I_{DD}(t)\,dt
\right)
$$

---

## Throughput

Throughput is defined as:

$$
Throughput =
\frac{\text{Number of Operations}}
{\text{Execution Time}}
$$

For the 4-term dot-product operation, the Sequential MAC processes one multiplication at a time, while the Parallel Accelerator evaluates four multiplications simultaneously.

---

## Energy per Operation

Energy per operation can be expressed as:

$$
E_{op}
=
\frac{E_{total}}
{N_{ops}}
$$

where:

$$
E_{total}
=
\int_{0}^{T}
V_{DD} I_{DD}(t)\,dt
$$

An approximate expression using average power is:

$$
E_{op}
\approx
\frac{P_{avg}T_{execution}}
{N_{ops}}
$$

---

# Scalability

The architecture can be generalized by increasing the number of parallel processing elements.

For an `N`-term dot product:

$$
Y = \sum_{i=0}^{N-1} X_iW_i
$$

Increasing the number of processing elements increases the number of simultaneous MAC operations.

```text
1 PE → 1 MAC operation / cycle
2 PE → 2 MAC operations / cycle
4 PE → 4 MAC operations / cycle
8 PE → 8 MAC operations / cycle
```

Ideally, for `N` terms and `P` processing elements, the required number of processing cycles is approximately:

$$
N_{cycle}
=
\left\lceil
\frac{N}{P}
\right\rceil
$$

Increasing parallelism can therefore improve throughput, but also increases:

- Multiplier count
- Adder count
- Routing complexity
- Layout area
- Power consumption

The architecture must therefore balance:

$$
\boxed{
Area \leftrightarrow Power \leftrightarrow Performance
}
$$

---

# Experimental Comparison

For the test configuration used in this project:

| Metric | Sequential MAC | Parallel Dot Product Accelerator |
|---|---:|---:|
| Architecture | Sequential | Parallel |
| Multiplications per cycle/evaluation | 1 | 4 |
| Processing of 4 products | 4 cycles | 1 evaluation |
| Reported delay | 1.750 ns / cycle | 0.814 ns / evaluation |
| Reported 4-operation completion time | 5.330 ns | 0.814 ns |

Using the reported completion times, the measured execution-time ratio is:

$$
\frac{5.330}{0.8142} \approx 6.55
$$

Thus, for the tested case, the parallel implementation completed the four-product operation in approximately:

$$
\boxed{6.5\times}
$$

less execution time than the reported Sequential MAC result.

This improvement is obtained by increasing hardware parallelism:

```text
Sequential MAC
1 × MUL4
1 × RCA10
1 × REG10
       │
       └── Hardware Reuse


Parallel Dot Product Accelerator
4 × MUL4
3 × RCA10
       │
       └── Hardware Parallelism
```

---

# Key Design Trade-Off

The two architectures represent different optimization directions.

### Sequential MAC

```text
Single MUL4
     │
     ▼
   RCA10
     │
     ▼
   REG10
     │
     └──── Feedback
```

The architecture emphasizes **hardware reuse** and performs the dot product over multiple clock cycles.

### Parallel Dot Product Accelerator

```text
MUL4 ──┐
MUL4 ──┼──► 2-Stage Adder Tree ──► Y
MUL4 ──┤
MUL4 ──┘
```

The architecture emphasizes **hardware parallelism** and evaluates the complete four-term dot product without sequential accumulator feedback.

The fundamental design trade-off can be expressed as:

$$
\boxed{
\text{Hardware Reuse}
\quad\longleftrightarrow\quad
\text{Parallelism}
}
$$

and must ultimately be evaluated using:

$$
\boxed{
Area,\ Power,\ Delay,\ Throughput,\ Energy/Op
}
$$
