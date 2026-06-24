# Distributed SVM Benchmark Experiments

### Datasets

- Real-Sim
- Covtype
- RCV1 / CCAT

### Methods Evaluated

#### CoCoA Family (Scala + Spark)

- CoCoA+
- CoCoA
- Mini-batch Coordinate Descent (MbCD)
- Mini-batch SGD
- Local SGD

#### MOCHA (MATLAB)

- MOCHA (Federated Multitask Learning)

---

# Dataset Setup

All experiments used **4 partitions/tasks**.

| Dataset | Train | Test | Features | Split |
|----------|----------:|----------:|----------|----------|
| Real-Sim | 54,231 | 18,078 | 256 (Truncated SVD) | 75/25 Stratified |
| Covtype | 435,759 | 145,253 | 54 (Original Features) | 75/25 Stratified |
| RCV1 / CCAT | 20,242 | 677,399 | 256 (Truncated SVD) | Original Train/Test |

---

# Preprocessing

## Real-Sim

- Loaded from LIBSVM format
- Labels converted to {-1, +1}
- 75/25 stratified train/test split
- L2 normalization
- Truncated SVD (256 dimensions)
- Final feature vectors L2 normalized

## Covtype

- Loaded from LIBSVM format
- Labels converted to {-1, +1}
- 75/25 stratified train/test split
- Original 54 features retained

## RCV1 / CCAT

- Used provided train/test split
- L2 normalization
- Truncated SVD (256 dimensions)
- Final feature vectors L2 normalized

---

# Partitioning

All datasets were partitioned into **4 IID partitions**.

### CoCoA Family

- Spark local mode (`local[4]`)
- 4 Spark worker partitions

### MOCHA

- 4 MATLAB task partitions

---

# Implementation Details

## CoCoA Family

### Language

- Scala
- Apache Spark

### Main Files

```text
driver.scala
CoCoA.scala
MinibatchCD.scala
SGD.scala
```

### Methods

- CoCoA+
- CoCoA
- Mini-batch Coordinate Descent
- Mini-batch SGD
- Local SGD

---

## MOCHA

### Language

- MATLAB

### Main Files

```text
optimization_driver.m
run_mocha.m
```

### Method

- MOCHA (Federated Multitask Learning)

---

# Hyperparameters

## CoCoA Family

Used for:

- CoCoA+
- CoCoA
- Mini-batch Coordinate Descent
- Mini-batch SGD
- Local SGD

```text
lambda          = 0.001
numRounds       = 50
localIterFrac   = 0.01
numSplits       = 4
beta            = 1.0
gamma           = 1.0
seed            = 0
```

### Additional Settings

- Hinge-loss SVM objective
- Initial model: `w = 0`
- Initial dual variables: `alpha = 0`
- Communication rounds: 50
- Local work per round: 1% of local partition

---

## MOCHA

```text
lambda              = 1e-4
mocha_outer_iters   = 1
mocha_inner_iters   = 50
mocha_sdca_frac     = 0.01
w_update            = 1
```

### Additional Configuration

```text
opts.obj      = 'C'
opts.avg      = 1
opts.sys_het  = 0
```


---

# Methodology

## CoCoA+

| Property | Value |
|----------|----------|
| Method Type | Distributed primal-dual optimization |
| Local Solver | SDCA-style coordinate optimization over dual variables α |
| Infrastructure | Spark local[4] |
| Workers | 4 Spark partitions |
| Communication | Spark master-worker communication |
| Aggregation | Workers return Δvₖ = AₖΔαₖ |
| Output | Single global SVM model |

### Aggregation Flow

```text
Worker k
  ↓
Δα_k
  ↓
Δv_k = A_k Δα_k
  ↓
Master Aggregation
  ↓
Global Model Update
```

---

## CoCoA

| Property | Value |
|----------|----------|
| Method Type | Distributed primal-dual optimization |
| Local Solver | SDCA-style coordinate optimization over dual variables α |
| Infrastructure | Spark local[4] |
| Workers | 4 Spark partitions |
| Communication | Spark master-worker communication |
| Aggregation | Aggregated into global state |
| Output | Single global SVM model |

---

## Mini-batch Coordinate Descent

| Property | Value |
|----------|----------|
| Method Type | Distributed coordinate-descent baseline |
| Local Solver | Coordinate descent over dual variables |
| Infrastructure | Spark local[4] |
| Workers | 4 Spark partitions |
| Communication | Spark master-worker communication |
| Aggregation | Coordinate updates aggregated globally |
| Output | Single global SVM model |

---

## Mini-batch SGD

| Property | Value |
|----------|----------|
| Method Type | Distributed first-order optimization |
| Local Solver | Mini-batch SGD on primal weights |
| Infrastructure | Spark local[4] |
| Workers | 4 Spark partitions |
| Communication | Spark master-worker communication |
| Aggregation | Worker updates averaged into global weights |
| Output | Single global SVM model |

---

## Local SGD

| Property | Value |
|----------|----------|
| Method Type | Distributed local-update SGD |
| Local Solver | Multiple local SGD updates before synchronization |
| Infrastructure | Spark local[4] |
| Workers | 4 Spark partitions |
| Communication | Spark master-worker communication |
| Aggregation | Local models combined during synchronization |
| Output | Single global SVM model |

---

## MOCHA

| Property | Value |
|----------|----------|
| Method Type | Federated Multitask Learning |
| Local Solver | SDCA-style coordinate updates |
| Tasks | 4 |
| Infrastructure | MATLAB |
| Communication | Simulated through task partitions |
| Aggregation | Task relationship matrix Ω |
| Output | Multiple task-specific models |

### Output Models

```text
w₁
w₂
w₃
w₄
```

Unlike CoCoA-family methods, MOCHA learns multiple related task-specific models rather than a single global model.
