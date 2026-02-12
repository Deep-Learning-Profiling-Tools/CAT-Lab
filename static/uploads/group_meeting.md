# Introduction

Memory safety bugs (buffer overflows, use-after-free, uninitialized reads) remain a leading cause of security vulnerabilities. 

On GPUs, the problem is amplified by massive parallelism and limited debugging infrastructure.

This presentation surveys memory safety analysis in three parts:

* **Part 1**: Static methods — abstract interpretation, lattices, fixpoint, widening/narrowing, and tools (CSSV, Astrée, Sparrow, IKOS)
* **Part 2**: Dynamic methods (CPU) — shadow memory, ASan/MSan encoding, quarantine
* **Part 3**: GPU sanitizers — GPUVerify, compute-sanitizer, ASan GPU, cuCatch, GPUShield, Let-Me-In, Triton-Sanitizer
  * cusafe


---

# Part 1: Static Methods

---

## 1.0 Abstract Interpretation — Core Concepts

### Core Idea

Instead of executing the program, simulate its behavior on an "**abstract domain**," using finite abstract values to approximate infinite concrete values.

---

### Concrete Values vs. Abstract Values

```python
x = 5                # Concrete value: 5
                     # Abstract value: x ∈ [5, 5]

x = input()          # Concrete value: unknown, could be any integer
                     # Abstract value: x ∈ (-∞, +∞)

x = abs(input())     # Concrete value: unknown, but definitely ≥ 0
                     # Abstract value: x ∈ [0, +∞)
```

---

### Why Approximation?

The concrete state space of a program is **infinite** (variables can take any integer value) — exhaustive enumeration is impossible.

Abstract interpretation's strategy: use a **finite, computable** abstract domain to summarize all possible concrete values.

```
Concrete world (infinite)          Abstract world (finite, computable)
x = 3 or 5 or 7           →       x ∈ [3, 7]
y = -100 ... 100           →       y ∈ [-100, 100]
z = any value              →       z ∈ (-∞, +∞) = ⊤
```

The cost: abstract values may include values that never actually occur → this is the source of **false positives**.

---

## 1.1 What Is a Lattice?

### Definition and Example: Interval Domain

A lattice is a **partially ordered set** where any two elements have a least upper bound and a greatest lower bound.

Using the interval domain as an example: the abstract value of variable x is represented by an interval `[a, b]`, meaning x's actual value is guaranteed to be between a and b.

| Operation | Symbol | Meaning | Interval Domain Example |
|-----------|--------|---------|------------------------|
| Partial order | ⊑ | "more precise than" (contained in) | `[2,5] ⊑ [0,10]` |
| Join | ⊔ | Least upper bound: merge info from two branches | `[1,3] ⊔ [5,8] = [1,8]` |
| Meet | ⊓ | Greatest lower bound: take intersection | `[1,6] ⊓ [4,9] = [4,6]` |
| Top | ⊤ | Global maximum: no information | `(-∞, +∞)` |
| Bottom | ⊥ | Global minimum: unreachable | `∅` (empty set) |

---

### Example 1: Precise Inference with Interval Domain

```python
x = 5        # x ∈ [5, 5]
y = -3       # y ∈ [-3, -3]
z = x + y    # [5,5] + [-3,-3] = [2, 2] ✓ Exact!
```

The interval domain can precisely track the results of arithmetic operations.

---

### Example 2: Precision Loss from Join

```python
if cond:
    a = 1    # a ∈ [1, 1]
else:
    a = 10   # a ∈ [10, 10]

# Join: a ∈ [1,1] ⊔ [10,10] = [1, 10]
#       Introduces spurious values 2~9

b = a * a    # [1,10] × [1,10] = [1, 100]
             # Actually b can only be 1 or 100
             # But the interval domain thinks b could be 50 → source of false positives
```

The Join operation is the root cause of precision loss: when merging branches, intervals can only grow, never shrink.

---

## 1.2 Fixpoint

### Definition

For each statement in the program, define a **transfer function** F that describes how the statement changes the abstract state.

Iterate F repeatedly until the abstract state no longer changes: **F(x) = x** — at this point x is the fixpoint.

---

### Example: Analyzing a Loop with Interval Domain

```python
x = 1
while x < 3:
    x = x + 1
```

Core question: without actually running the loop, how do we determine all possible values of x at the loop head?

**Method: repeatedly simulate the loop until the abstract state stabilizes.**

```python
# ============ Iteration 1 ============
x = 1                  # x ∈ [1, 1]
while x < 3:           # Condition filter: x ∈ [1, 1] (satisfies <3)
    x = x + 1          # x ∈ [2, 2]
                        # Back to loop head, Join: [1,1] ⊔ [2,2] = [1, 2]
                        # State changed → continue

# ============ Iteration 2 ============
                        # Loop head: x ∈ [1, 2]
while x < 3:           # Condition filter: x ∈ [1, 2] (all satisfy <3)
    x = x + 1          # x ∈ [2, 3]
                        # Back to loop head, Join: [1,1] ⊔ [2,3] = [1, 3]
                        # State changed → continue

# ============ Iteration 3 ============
                        # Loop head: x ∈ [1, 3]
while x < 3:           # Condition filter: x ∈ [1, 2] (only the <3 portion enters)
    x = x + 1          # x ∈ [2, 3]
                        # Back to loop head, Join: [1,1] ⊔ [2,3] = [1, 3]
                        # State unchanged! [1, 3] is the fixpoint ✓
```

---

### Loop Exit Conclusion

Negate the loop condition `x >= 3`, filter the fixpoint [1, 3]:

→ After the loop, x ∈ **[3, 3]** (exact!)

---

## 1.3 Widening

### Problem Review

```python
x = 1
while x < 10:
    x = x + 1
```

Naive fixpoint iteration: the upper bound increases by 1 each round, requiring 10 iterations to converge. If `x < N`, then N iterations are needed.

When N is very large or **the loop is unbounded**, the iteration **never terminates**.

---

### The Widening Approach

At the loop head, if the abstract state's bounds are **continuously growing**, jump directly to +∞.

---

### Re-analysis with Widening

```python
# ============ Iteration 1 ============
x = 1                  # x ∈ [1, 1]
while x < 10:          # Condition filter: x < 10. x ∈ [1, 1]
    x = x + 1          # x ∈ [2, 2]
                        # Back to loop head, normal Join: [1,1] ⊔ [2,2] = [1, 2]
                        # Upper bound grew (1 → 2), trigger Widening!
                        # [1, 1] ▽ [1, 2] = [1, +∞)

# ============ Iteration 2 ============
                        # Loop head: x ∈ [1, +∞)
while x < 10:          # Condition filter: x < 10. x ∈ [1, 9]
    x = x + 1          # x ∈ [2, 10]
                        # Back to loop head, Join: [1,1] ⊔ [2,10] = [1, 10]
                        # [1, 10] ⊆ [1, +∞), no expansion, Widening not triggered
                        # State remains [1, +∞), unchanged → fixpoint ✓
```

**Converged in just 2 iterations**, regardless of how large N is.

---

### Loop Exit Conclusion

At loop exit, negate condition `x >= 10`, filter [1, +∞):

→ After the loop, x ∈ **[10, +∞)**

Compared to naive iteration's result x ∈ [10, 10], Widening lost the **upper bound** information.

---

### Summary

|  | Naive Fixpoint | Widening |
|--|---------------|----------|
| Convergence speed | O(N) iterations | 2 iterations |
| x after loop | [10, 10] (exact) | [10, +∞) (some loss) |
| Unbounded loops | Does not terminate | Still terminates |

**Widening is a trade-off between termination and precision: it guarantees the analysis will finish, at the cost of potentially less precise results.**

---

## 1.4 Narrowing

### Problem Review

Widening guaranteed termination, but the result is overly broad:

```
Loop head fixpoint: x ∈ [1, +∞)
Loop exit:          x ∈ [10, +∞)    ← upper bound was lost
```

Can we **recover** the excess portion after Widening?

---

### The Narrowing Approach

After Widening produces a fixpoint, perform a few more iterations, but this time use the Narrowing operator △ instead of Widening: **only allow the interval to shrink, never expand.**

---

### Recovering Precision with Narrowing

Starting from Widening's fixpoint x ∈ [1, +∞):

```python
# ============ Narrowing Iteration 1 ============
                        # Loop head: x ∈ [1, +∞)
while x < 10:          # Condition filter: x < 10. x ∈ [1, 9]
    x = x + 1          # x ∈ [2, 10]
                        # Back to loop head, Join: [1,1] ⊔ [2,10] = [1, 10]
                        # Narrowing: [1, +∞) △ [1, 10] = [1, 10]
                        #            Upper bound narrowed from +∞ to 10 ✓

# ============ Narrowing Iteration 2 ============
                        # Loop head: x ∈ [1, 10]
while x < 10:          # Condition filter: x ∈ [1, 9]
    x = x + 1          # x ∈ [2, 10]
                        # Back to loop head, Join: [1,1] ⊔ [2,10] = [1, 10]
                        # Narrowing: [1, 10] △ [1, 10] = [1, 10]
                        #            No change → fixpoint ✓
```

---

### Loop Exit Conclusion

Negate condition `x >= 10`, filter [1, 10]:

→ After the loop, x ∈ **[10, 10]** — precision fully recovered!

---

### Complete Workflow

```
Naive iteration (does not terminate or too slow)
        ↓
Widening (fast convergence, but overly broad)
   x ∈ [1, +∞)  →  exit x ∈ [10, +∞)
        ↓
Narrowing (shrink, recover precision)
   x ∈ [1, 10]  →  exit x ∈ [10, 10] ✓
```

**Widening + Narrowing = guaranteed termination with maximum precision recovery.**

---

## 1.5 Static Analysis Tools Based on Abstract Interpretation

### Review: Limitations of Basic Abstract Interpretation

The basic methods we've introduced: interval domain + fixpoint iteration + Widening/Narrowing

Limitations:
- The interval domain is **non-relational**: cannot express relationships between variables (e.g., `len < alloc`)
- Abstract state must be computed at every program point: **does not scale** for large programs
- All paths are merged (Join): **path information is lost**

The following tools address these problems from different angles.

---

### CSSV — More Precise Abstract Domains

**Goal**: Statically detect all buffer overflows in C programs.

**Key innovation**: Replace the interval domain with the **Polyhedra domain**.

```python
alloc = 10         # Allocated a 10-byte buffer
len = input()      # String length from user input
assert len < alloc # Safety check: length must not exceed allocation size
```

Interval domain perspective (each variable independent):

```
alloc ∈ [10, 10]
len   ∈ [0, +∞)     ← Only knows len's range

Problem: The interval domain cannot remember the relationship "len < alloc"
         In subsequent code, it doesn't know there's a constraint between len and alloc
```

Polyhedra domain perspective (can express linear inequalities):

```
alloc = 10
len >= 0
len <= alloc - 1    ← Remembers the relationship between variables!

When subsequent code accesses buffer[len]:
→ Can prove len <= 9 < alloc = 10, no overflow ✓
```

Visual comparison:

```
Interval domain: one "box" per variable    Polyhedra domain: one "polyhedron" for all

  alloc                                     alloc
   ↑                                         ↑
   |  ┌──┐                                   |      /
10 |  │  │                                10 |     / (len < alloc)
   |  └──┘                                   |    /
   +--------→ len                            |   /
      ┌─────────┐                            |  /
      │ 0 ... +∞│                          +─────→ len
      └─────────┘                             0    9

   Two independent intervals                  One constraint region
   No relationship visible                    Relationship precisely preserved
```

**Cost**: Polyhedra domain has exponential complexity; only suitable for small-scale programs.

---

### Astrée — Multiple Abstract Domains Collaborating

Continuing the same example with an additional variable:

```python
alloc = 10
len = input()
offset = input()
assert offset + len < alloc   # Safety condition: offset + length must not exceed allocation
```

Analysis capability of three domains on this code:

```
Interval domain (each variable independent):
  alloc ∈ [10, 10], len ∈ [0, +∞), offset ∈ [0, +∞)
  → Three variables completely independent, no relationships known ✗

Octagon domain (±X ± Y ≤ c, can only constrain two variables):
  len - alloc ≤ -1         → len ≤ 9       ✓
  offset - alloc ≤ -1      → offset ≤ 9    ✓
  offset + len ≤ alloc?    → ✗ Involves three variables, cannot express!

Polyhedra domain (arbitrary linear inequalities):
  offset + len ≤ alloc - 1                  ✓
```

For accessing `buffer[offset + len]`:
- Interval domain: cannot determine safety
- Octagon: can prove each is within bounds individually, but cannot prove their **sum** is within bounds
- Polyhedra domain: can prove it precisely

```
        High precision, high cost
            ↑
    ┌───────────────┐
    │   Polyhedra   │  Arbitrary linear relations    O(2^n)
    ├───────────────┤
    │   Octagons    │  ±X ± Y ≤ c                   O(n³)
    ├───────────────┤
    │   Intervals   │  Each variable independent     O(n)
    └───────────────┘
            ↓
        Low precision, low cost
```

Astrée's strategy: **run multiple domains simultaneously** and let them collaborate. Use Octagon in most cases (sufficient and fast), and enable Polyhedra when necessary (precise but expensive). 

---

### IKOS & Sparrow — Sparse Analysis for Scalability

**Goal**: Scale abstract interpretation to **millions of lines** of code.

**Core problem**: The basic method computes abstract state at every program point — most computation is wasted.

**Sparrow's approach — Sparse Value-Flow**:

```
Basic method (Dense): compute at every program point

    a = 1        ← compute state of a
    b = 2        ← compute state of a, b
    c = 3        ← compute state of a, b, c
    ...          ← computing everywhere
    d = a + c    ← compute state of all variables

Sparse method: propagate only along def-use chains

    a = 1  ─────────────────→  d = a + c
    c = 3  ─────────────────→  ↑
                              Only compute where a, c are used
```

**Result**: Same precision as dense analysis, but **1500× faster**.

**IKOS's approach**: Parallel fixpoint computation + memory-efficient iteration strategies.

---

# Part 2: Dynamic Methods (CPU)

---

## 2.1 Dynamic Instrumentation Overview

### Core Idea: Insert Runtime Callbacks for Every Memory Access

Static analysis infers program behavior at compile time, while dynamic **instrumentation** takes a completely different strategy: **monitor every memory access during actual program execution**.

```c
// Original code
int arr[10];
arr[idx] = 42;

// After instrumentation (pseudocode)
int arr[10];
__sanitizer_check_write(&arr[idx], sizeof(int));  // Runtime callback
arr[idx] = 42;
```

Before each load/store operation, insert a check function that verifies whether the access is legal (out of bounds, already freed, uninitialized, etc.).

---

### Two Approaches to Instrumentation

| Approach | Representative Tools | Timing | Pros | Cons |
|----------|---------------------|--------|------|------|
| Compile-time instrumentation | ASan, MSan, TSan | Compiler inserts checks at IR level | Lower overhead, optimizable | Requires recompilation |
| Binary instrumentation | Valgrind, DynamoRIO | Runtime binary instruction | No source code/recompilation needed | Extremely high overhead (10-50×) |

---

### Shadow Memory Mechanism

Shadow Memory is the core data structure of dynamic instrumentation tools: **for every byte of program memory, maintain additional metadata (shadow state)**.

```
Application Memory                    Shadow Memory
┌──────────────────────┐            ┌──────────────────────┐
│ addr: 0x1000  [data] │  ──maps──→ │ shadow: metadata      │
│ addr: 0x1001  [data] │  ──maps──→ │ shadow: metadata      │
│        ...           │            │        ...            │
└──────────────────────┘            └──────────────────────┘
```

Typical mapping formula:

```
ShadowAddr = (AppAddr >> Scale) + Offset
```

Using ASan as an example: every **8 bytes** of application memory → corresponds to **1 byte** of shadow memory (Scale = 3), with memory overhead of approximately 12.5%.

---

#### ASan's Shadow Encoding

ASan (AddressSanitizer) uses **1 byte of shadow** to represent the accessibility state of the corresponding 8 bytes of application memory:

| Shadow Value | Meaning | Detected Issue |
|-------------|---------|----------------|
| `0x00` | All 8 bytes accessible | — |
| `0x01`~`0x07` | First k bytes accessible (at allocation boundary) | — |
| `0xFA` (-6) | Heap red zone — guard area around heap allocations<br /><br />[0, 1,2, 3,4,5,6,7,8]<br />[0, 1] [4, 5] , [7,8] | Heap buffer overflow (Heap OOB) |
| `0xF1` (-15) | Stack left red zone — guard area to the left of stack variables | Stack buffer overflow (Stack OOB) |
| `0xF2` (-14) | Stack mid red zone — guard area between adjacent stack variables | Stack buffer overflow (Stack OOB) |
| `0xF3` (-13) | Stack right red zone — guard area to the right of stack variables | Stack buffer overflow (Stack OOB) |
| `0xFD` (-3) | Freed — already-freed heap memory | Use-After-Free |

Check logic: shadow interpreted as `int8_t`, **any negative value is illegal**, with different negative values distinguishing error types.

---

#### MSan's Shadow Encoding

MSan (MemorySanitizer) uses an **independent shadow encoding scheme**, tracking initialization state at the bit level:

| Shadow Value | Meaning | Detected Issue |
|-------------|---------|----------------|
| `0x00` | Corresponding memory is initialized (every bit determined) | — |
| `0xFF` | Corresponding memory is uninitialized (every bit unknown) | Uninitialized memory read |

At allocation, shadow is `0xFF`; after a write, it becomes `0x00`; reading memory marked `0xFF` triggers an error.

```c
int x;          // shadow = 0xFF (uninitialized)
x = 42;         // shadow = 0x00 (initialized)

int y;          // shadow = 0xFF
if (y > 0) {}   // Reading uninitialized value → MSan reports error
```

---

### Quarantine

Quarantine is essentially a **FIFO queue** that delays memory deallocation:

The actual flow of `free(ptr)`:

1. Do not immediately return to the allocator
2. Push ptr to the tail of the quarantine queue
3. Mark shadow as 0xFD
4. If the queue's total size exceeds the limit → pop from the head and truly return to the allocator

**Without quarantine**:

```
free(A)    → shadow = 0xFD
malloc(B)  → allocator immediately reuses A's address → shadow = 0x00
*A = 1     → shadow is 0x00 → missed detection

Between free and the next malloc there may be only a few instructions,
the 0xFD window is extremely short, making UAF nearly impossible to catch.
```

**With quarantine**:

```
free(A)    → shadow = 0xFD, A enters quarantine (not returned to allocator)
malloc(B)  → allocator cannot reuse A's address (A is still in quarantine)
malloc(C)  → same as above
...        → during this time, any *A access is caught by 0xFD ✓
[quarantine full] → A is finally truly returned
```

Essence: quarantine **trades memory** to **extend the detection window**. Freed memory in the quarantine zone won't be reused, shadow persistently remains 0xFD, and the UAF detection window extends from "a few instructions" to "until quarantine is exhausted."

This is a trade-off between **memory overhead vs. detection coverage**.

---

The core advantage of dynamic methods: **zero false positives** — only illegal accesses actually executed by the program are reported.

---

# Part 3: GPU Sanitizers

---

## 3.1 GPU Memory Safety Challenges

### GPUVerify: Static Verification

**Problem**: Detect data races and barrier divergence in GPU kernels.

**Method**: Reduce an N-thread kernel to a predicated execution program (sequential program) of any two distinct threads, insert assertions, and convert to Z3 constraints for solving.

- **Data race**: At each shared memory access, insert logs (LOG_RD/LOG_WR) recording the read/write addresses of both threads, and assert that write addresses do not conflict.
- **Barrier divergence**: When a barrier is called, assert that the enabled predicates of both threads are identical (either both reach it or neither does). If one thread reaches the barrier while the other does not, it indicates thread divergence in control flow.

**Example**:

```c
// Data race check: two threads cannot write to the same address
assert(!(WR_exists_A1 && WR_exists_A2 && WR_elem_A1 == WR_elem_A2));

// Barrier divergence check: both threads must reach barrier simultaneously
assert(en1 == en2);
```

Limitations: Static methods have false positives; complex kernels require user-annotated loop invariants; difficult to scale to large workloads.

---

### compute-sanitizer: Dynamic Instrumentation

Performs runtime instrumentation on compiled GPU binaries — no recompilation needed, but with high overhead (~48%).

Working mechanism:

1. Operates on compiled GPU binaries (SASS instructions), inserting check code before each memory access instruction
2. Maintains an allocated **region registry** at runtime (recording which address ranges are allocated and not freed)
3. Before each memory access, checks whether the target address falls within any allocated region

Key limitation: It only knows whether an address belongs to "some" allocated region, but cannot distinguish **which** object it belongs to. When two buffers are allocated contiguously, an overflow from A into B is still a "valid address" → misses inter-object OOB.

---

### ASan (GPU Version): Compile-Time Instrumentation + Shadow Memory

Modifies code at the LLVM IR level, inserting bounds checks for every memory access. Uses Shadow Memory to record the accessibility state of each byte:

```
ShadowAddr = (AppAddr >> Scale) + Offset
```

When the shadow encodes a negative value, it indicates an illegal access (red zone, freed, etc.).

---

### cuCatch: Compile-Time Instrumentation + Driver Support (cuda 13.1)

cuCatch also performs compile-time instrumentation, but has key differences from ASan:

| | ASan | cuCatch |
|--|------|---------|
| Pointer format | Plain pointer | **Fat Pointer** (carries base, bounds, tag) |
| Shadow content | 1-byte status code (indexed by address) | base/size/tag triple (indexed by object) |
| Temporal safety | Marks freed (0xFD) + quarantine | Tag version number matching |
| Driver dependency | None | **Requires GPU driver cooperation** (writes shared memory range before launch) |
| Detection logic | shadow < 0 → illegal | Out-of-bounds or tag mismatch → illegal |

**Tag mechanism explained**: Each time `malloc` allocates, a unique tag (version number) is generated for that object, written into both the fat pointer and the shadow map. When `free` is called, only the tag in the shadow map is updated. If the old pointer is used afterward, pointer.tag ≠ shadow.tag → Use-After-Free detected.

**Why use tags instead of ASan's quarantine?**

ASan's temporal safety relies on quarantine: freed memory is marked as 0xFD and temporarily not reclaimed. But quarantine size is limited — once exhausted, old memory is reallocated, the 0xFD mark is overwritten, and subsequent UAF is completely undetectable:

```
malloc(A) → free(A) → [in quarantine, 0xFD, detectable]
                     → [quarantine full, A is reclaimed and reallocated]
                     → access old pointer → shadow is now 0x00 → missed!
```

With the tag approach, even when memory is reallocated, the new allocation gets a new tag, while the old pointer still carries the old tag:

```
malloc(A), tag=5 → free(A) → malloc(B) reuses same address, tag=9
                            → access with old pointer: ptr.tag(5) ≠ shadow.tag(9) → UAF!
```

| | ASan (quarantine) | cuCatch (tag) |
|--|-------------------|---------------|
| Immediate UAF (just freed) | Detectable (0xFD) | Detectable (tag mismatch) |
| Delayed UAF (already reallocated) | **Missed** | **Probabilistic detection** (low tag collision probability) |
| GPU memory pressure | Quarantine occupies precious GPU memory | No need to retain freed memory |

---

### GPUShield: Hardware-Assisted Bounds Checking

GPUShield (ISCA 2022, Georgia Tech) takes a completely different approach: **offloading bounds checks to custom hardware** rather than software instrumentation. Custom hardware performs bounds checks on every memory access by indexing a Bounds Table with an ID.

The custom hardware includes an **RCache** and address comparators. The **Bounds Table** is a table stored in global memory, where each entry records a buffer's (base, size). The driver assigns a unique ID to each buffer and writes it to the Bounds Table. During access, the hardware uses the ID to look up the table and compare whether the address is out of bounds. 

Total hardware overhead: only 14.2 KB (NVIDIA) / 21.3 KB (Intel) across all SMs, with minimal impact on chip area. The trade-off is that it requires modifying GPU hardware design and cannot be deployed on existing GPUs.

---

### Let-Me-In: Hardware-Assisted (Same Team's Improvement)

Let-Me-In (HPCA 2025, Georgia Tech, same team) is also a hardware-assisted approach, but improves the metadata storage method:

| | GPUShield (ISCA 2022) | Let-Me-In (HPCA 2025) |
|--|---|---|
| Metadata location | External Bounds Table (in global memory) | High bits of the pointer itself |
| Lookup method | ID embedded in pointer → table lookup → get bounds | Extract bounds directly from pointer (no table lookup needed) |
| Additional hardware | RCache + comparator | Comparator only (no cache needed) |
| Memory allocation requirement | No special requirements | Must use 2ⁿ-aligned allocation |

---

### Triton-Sanitizer: Hybrid Symbolic Execution + SMT Solving

Triton-Sanitizer (ASPLOS 2026, George Mason + Anthropic + Meta + Google + OpenAI) takes a completely different approach from all the above tools: **no GPU instrumentation — instead, it detects memory errors on the CPU side through symbolic execution + Z3 SMT solver**.

**Core mechanism**:

1. **Tile-semantic awareness**: Triton operates on memory in units of tiles (tensor blocks). Triton-Sanitizer leverages this property to perform symbolic reasoning on the address range of entire tiles, rather than checking each thread's individual access.
2. **Symbolic execution**: Interprets the Triton kernel on the CPU, expressing address computations as symbolic expressions (rather than concrete values).
3. **Z3 SMT solving**: Encodes "whether a tile's address range is out of bounds" as Z3 constraints, and the solver determines whether an out-of-bounds access is possible.
4. **Eager simulation**: For complex operations that cannot be symbolized (e.g., `tl.where`, indirect indexing), falls back to element-by-element simulation on the CPU.
