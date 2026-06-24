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

| Item | Details |
|---|---|
| Is it mini-batch SGD? | No |
| Method type | Distributed primal-dual optimization |
| Local solver | Local SDCA / coordinate ascent over dual variables `alpha` |
| Distributed setup | Spark `local[4]` |
| Workers | 4 Spark partitions |
| Communication | Workers compute local updates and send them back through Spark RDD operations |
| Aggregation | Workers return local model updates; master updates `w ← w + scaling × ΣΔw_k` |
| Output | Single global SVM model |

In the code, CoCoA+ is run through `CoCoA.runCoCoA(..., plus=true)`. The implementation maintains both primal weights `w` and dual variables `alpha`, which is why objective value and duality gap are available.

---

## CoCoA

| Item | Details |
|---|---|
| Is it mini-batch SGD? | No |
| Method type | Distributed primal-dual optimization |
| Local solver | Local SDCA / coordinate ascent over dual variables `alpha` |
| Distributed setup | Spark `local[4]` |
| Workers | 4 Spark partitions |
| Communication | Workers compute local dual/primal updates and send them back through Spark |
| Aggregation | Master sums worker updates and applies scaling to update the global model |
| Output | Single global SVM model |

In the code, vanilla CoCoA is run through `CoCoA.runCoCoA(..., plus=false)`. It uses the same local SDCA-style solver as CoCoA+, but the aggregation scaling differs from the CoCoA+ mode.

---

## Mini-batch Coordinate Descent

| Item | Details |
|---|---|
| Is it mini-batch SGD? | No |
| Method type | Mini-batch coordinate descent / SDCA-style baseline |
| Local solver | Coordinate updates over dual variables `alpha` |
| Distributed setup | Spark `local[4]` |
| Workers | 4 Spark partitions |
| Communication | Workers compute mini-batch coordinate updates and return model/dual updates |
| Aggregation | Master aggregates coordinate updates into a single global model |
| Output | Single global SVM model |

Mini-batch CD is implemented separately from CoCoA through `MinibatchCD.runMbCD(...)`. Since it maintains dual variables, the implementation reports objective value and duality gap.

---

## Mini-batch SGD

| Item | Details |
|---|---|
| Is it mini-batch SGD? | Yes |
| Method type | Distributed first-order primal optimization |
| Local solver | Stochastic gradient updates on primal weights `w` |
| Distributed setup | Spark `local[4]` |
| Workers | 4 Spark partitions |
| Communication | Workers compute local stochastic gradient/model updates and return them through Spark |
| Aggregation | Master sums worker updates and applies a global SGD update |
| Output | Single global SVM model |

Mini-batch SGD is implemented through `SGD.runSGD(..., local=false)`. It only maintains primal weights `w`, so it reports objective value and test error, but not duality gap.

---

## Local SGD

| Item | Details |
|---|---|
| Is it mini-batch SGD? | Related, but it is the local-update variant |
| Method type | Local SGD / first-order primal optimization |
| Local solver | Multiple local SGD updates before synchronization |
| Distributed setup | Spark `local[4]` |
| Workers | 4 Spark partitions |
| Communication | Workers perform local SGD steps and return local model differences |
| Aggregation | Master aggregates local model differences into the global model |
| Output | Single global SVM model |

Local SGD is implemented through `SGD.runSGD(..., local=true)`. Compared with mini-batch SGD, workers perform local updates before synchronization. It only maintains primal weights, so no duality gap is reported.

---

## MOCHA

| Item | Details |
|---|---|
| Is it mini-batch SGD? | No |
| Method type | Federated multitask learning |
| Local solver | SDCA-style coordinate updates |
| Distributed setup | MATLAB single-machine simulation |
| Tasks | 4 task partitions |
| Communication | Simulated through MATLAB task partitions, not Spark/gRPC/TCP |
| Aggregation | Task-specific models are coupled through the task relationship matrix `Ω` |
| Output | Multiple task-specific models `w₁, w₂, w₃, w₄` |

Unlike the CoCoA-family methods, MOCHA does not learn one global model. It learns multiple related task-specific models and shares information through the learned task relationship matrix `Ω`.


### Output Models

```text
w₁
w₂
w₃
w₄
```

Unlike CoCoA-family methods, MOCHA learns multiple related task-specific models rather than a single global model.
