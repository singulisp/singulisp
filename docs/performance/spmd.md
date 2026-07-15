# SPMD

SPMD (Single Program, Multiple Data) is a data-parallel processing model for CPUs. It runs one program over multiple data elements simultaneously, and the compiler automatically vectorizes the program into SIMD instructions. This provides a CPU programming model similar to GPU compute shaders.

## `spmd` Syntax

An `spmd` block specifies an iteration variable and an element count, then executes its body for every element.

```lisp
(spmd [i count]
  (let x (Array/get data i))
  (Array/set result i (* x x)))
```

`i` is the index of each iteration, and `count` is the total number of elements. The compiler analyzes the block and, when possible, vectorizes it into SIMD instructions through strip mining.

### Basic Example

```lisp
(defn square-all [(data : [Array f32 1024])] -> [Array f32 1024]
  (let mut result data)
  (spmd [i 1024]
    (let x (Array/get data i))
    (spmd/scatter! result i (* x x)))
  result)
```

## `#[spmd :width N]` — Vectorization Width

Explicitly specifies the vectorization width: the number of elements processed at once.

```lisp
(defn process [(data : [Array f32 1024])] -> [Array f32 1024]
  (let mut result data)
  #[spmd :width 8]
  (spmd [i 1024]
    (let x (Array/get data i))
    (spmd/scatter! result i (* x 2.0f)))
  result)
```

This example processes eight elements at a time with SIMD. When no width is specified, the compiler selects an optimal width based on the target architecture.

## `#[spmd :require]` — Require Vectorization

Makes vectorization failure a compile-time error. Use it in performance-critical code to prevent unintended scalar fallback.

```lisp
(defn critical-path [(data : [Array f32 4096])] -> [Array f32 4096]
  (let mut result data)
  #[spmd :require]
  (spmd [i 4096]
    (spmd/scatter! result i (f32/sqrt (Array/get data i))))
  result)
```

## Scatter and `atomic-*` Update Surface

These operations represent noncontiguous memory access and intentional read-modify-write collisions within an SPMD block.

### Scatter Writes

```lisp
(spmd [i count]
  (let idx (compute-index i))
  (spmd/scatter! output idx (Array/get input i)))
```

Use `spmd/scatter!` when lanes write to different indices. Unlike a normal `Array/set`, it explicitly represents a noncontiguous access pattern.

### Atomic Operations

```lisp
(spmd [i count]
  (let bin (compute-bin (Array/get data i)))
  (spmd/atomic-add! histogram bin 1))
```

| Function | Description |
|---|---|
| `(spmd/atomic-add! buf idx val)` | Additive update (`buf[idx] += val`) |
| `(spmd/atomic-min! buf idx val)` | Minimum update (`buf[idx] = min(buf[idx], val)`) |
| `(spmd/atomic-max! buf idx val)` | Maximum update (`buf[idx] = max(buf[idx], val)`) |

Despite `atomic` in their names, the current MIR backend lowers these operations to ordinary load, compare, and store operations. It does not generate LLVM `atomicrmw` or `cmpxchg`. This is an update surface for scalar fallback or a single CPU loop; it does not guarantee atomicity across OS threads or lock-free behavior.

## Reduce

Aggregates values from an SPMD block into a single result.

```lisp
(spmd [i count]
  (let x (Array/get data i))
  (spmd/reduce + 0.0f (* x x)))
```

Specify `+` directly as an operator symbol, not as a keyword. The operator combines the initial value—`0.0f` in this example—with the value from each lane. Supported operators are `+`, `*`, `min`, `max`, `&&`, `||`, `&`, `|`, and `^`.

> **Note**: Bitwise operator symbols such as `^` (bitwise XOR), `&`, and `|` are accepted in this reduce/scan operator position, but not in ordinary expression position. In an ordinary expression, use a named form such as `(bit-xor a b)`. An invalid operator/type combination—for example, `&&` in an integer reduction—reports **E-SPMD-0074**.

## Uniformity Analysis

The compiler statically classifies each variable in an SPMD block as **uniform**, meaning it has the same value in every lane, or **varying**, meaning its value differs by lane.

- **uniform**: Loop counts, constants, and computation results shared by every lane
- **varying**: The iteration variable `i` and results that depend on `i`

Uniform variables remain in scalar registers, while varying variables are expanded into SIMD registers. This analysis avoids unnecessary vectorization and enables efficient code generation.

## Strip Mining

The compiler transforms an SPMD block as follows:

1. Divide the loop into strips of the SIMD width, such as eight elements.
2. Expand varying variables into SIMD registers.
3. Convert scalar operations into SIMD operations such as SimdLoad, SimdStore, and SimdBinOp.
4. Generate masked code for the remainder when the element count is not divisible by the SIMD width.

## Error Codes

Compile-time errors related to SPMD:

| Code | Description |
|---|---|
| E-SPMD-0004 | `set!` on a scalar variable inside an SPMD block; only indexed memory is allowed |
| E-SPMD-0006 | Injectivity of a write address cannot be proven because of indirect indexing |
| E-SPMD-0007 | Input/output aliasing: conflicting reads and writes to different indices in the same buffer |
| E-SPMD-0074 | Invalid reduce/scan operator and type combination, such as `&&` with an integer type |
| E-SPMD-0031 | Too few arguments to `spmd/where` |
| E-SPMD-0032 | Too few arguments to `spmd/scan-exclusive` |
| E-SPMD-0033 | Invalid `spmd/scan-exclusive` operator |
| E-SPMD-0036 | Too few arguments to `spmd/atomic-cas!` |
| E-SPMD-0037 | Too few arguments to `spmd/atomic-xchg!` |
| E-SPMD-0038 | Too few arguments to `spmd/shuffle` |
| E-SPMD-0039 | Unknown `spmd/shuffle` kind |
| E-SPMD-0040 | Too few arguments to `spmd/compact` |
| E-SPMD-0041 | Too few arguments to `spmd/transpose-load` |
| E-SPMD-0042 | Too few arguments to `spmd/transpose-store` |
| E-SPMD-0043 | Too few arguments to `spmd/for-active` |
| E-SPMD-0044 | Too few arguments to `spmd/for-unique` |

## `spmd/scan` and `spmd/scan-exclusive` — Prefix Scans

`spmd/scan` performs an inclusive prefix scan. It accumulates lane values from left to right with the operator and returns a result that includes the current lane's value. `spmd/scan-exclusive` performs an exclusive scan and omits the current lane's value, returning the accumulation through the preceding lane.

```lisp
; inclusive scan
(spmd [i count]
  (spmd/scan + 0 (Array/get data i)))

; exclusive scan (aggregate through the preceding lane)
(spmd [i count]
  (spmd/scan-exclusive + 0 (Array/get data i)))
```

As with `spmd/reduce`, supported operators include `+`, `*`, `min`, and `max`. The initial value becomes the output of the first lane in an exclusive scan.

## `spmd/where` — Conditional Execution

Provides conditional execution without an else branch. It desugars internally into `if` plus mask conversion. In lanes where the condition is false, side effects such as writes are masked and do not execute.

```lisp
(spmd [i count]
  (spmd/where (> (Array/get data i) 0.0)
    (Array/set result i (* (Array/get data i) 2.0))))
```

Vectorized execution uses a mask register for conditional writes. In scalar mode, the operation is equivalent to an ordinary `if` expression.

## `spmd/lane-index` and `spmd/lane-count` — Lane Information

These functions return the current absolute iteration index and the execution width.

```lisp
(spmd [i count]
  (let lane (spmd/lane-index))   ; absolute iteration index (scalar: IV, vector: base..base+W-1)
  (let width (spmd/lane-count))) ; lane width (scalar: 1, vector: W)
```

- `spmd/lane-index`: In scalar mode, returns the iteration variable itself. Under vectorization, returns an `iota` starting at the current IV: `[base, base+1, ..., base+W-1]`.
- `spmd/lane-count`: In scalar mode, returns the constant `1`. Under vectorization, returns the SIMD width `W` as a constant.

## Advanced SPMD Surface

The following surface is available with vector semantics that preserve SIMD lanes and masks:

- `spmd/shuffle`
- `spmd/compact`
- `spmd/transpose-load` / `spmd/transpose-store`
- `spmd/for-active` / `spmd/for-unique`
- `spmd/any` / `spmd/all` / `spmd/none`
- `spmd/extract` / `spmd/broadcast-from`
- `spmd/unmasked`
- `spmd/histogram`

`spmd/assume-uniform` returns its input value unchanged.

## `spmd/atomic-cas!` / `spmd/atomic-xchg!` — Conditional Update and Exchange

In addition to `spmd/atomic-add!` and related operations, this surface represents compare-then-update and unconditional exchange.

```lisp
(spmd [i count]
  (spmd/atomic-cas! buf idx expected desired)  ; compare and exchange; returns the old value
  (spmd/atomic-xchg! buf idx value))            ; exchange; returns the old value
```

- `spmd/atomic-cas!`: Updates `buf[idx]` to `desired` only if its current value equals `expected`. Returns the value before the update.
- `spmd/atomic-xchg!`: Unconditionally updates `buf[idx]` to `value`. Returns the value before the update.

In the current backend, these are also ordinary load, compare, and store operations. They do not guarantee atomicity across OS threads, lock-free behavior, or a synchronization order. Do not use them as synchronization primitives for concurrent data structures.

## Bit-Manipulation Intrinsics

These bit-manipulation functions are available inside and outside SPMD and map to the LLVM intrinsics `llvm.ctpop`, `llvm.ctlz`, and `llvm.cttz`.

```lisp
(i32/popcount x)   ; population count (number of one bits)
(i32/clz x)        ; count leading zero bits
(i32/ctz x)        ; count trailing zero bits
; i64 variants are analogous: (i64/popcount x), (i64/clz x), (i64/ctz x)
```

When these functions are used in an SPMD block with a varying variable, the vector intrinsic is used. For an input of zero, they return the bit width: `clz(0) == 32` and `ctz(0) == 32` for `i32`.

## Prefetching

Prefetch operations specify cache hints explicitly. They fetch a cache line before the data is needed to hide memory latency.

```lisp
(prefetch/read addr)    ; read prefetch
(prefetch/write addr)   ; write prefetch
```

`prefetch/read` issues a read prefetch to the L1 cache with `llvm.prefetch` locality 3. `prefetch/write` issues a write-intent prefetch with locality 3. Within an SPMD block, a prefetch is issued for each lane's address.

## `#[spmd :min-trip N]` — Minimum Trip Count

Provides the vectorization pass with a profitability hint for the minimum iteration count.

```lisp
#[spmd :min-trip 64]
(spmd [i count]
  ...)
```

`N` is not a runtime precondition. When a candidate SIMD width exceeds `N`, that vectorization is skipped. Scalar-loop semantics remain defined even when `count < N`.

## `#[spmd :targets [4 8]]` — Multi-Target Width Dispatch

Compiles the block at multiple SIMD widths and selects the best width at runtime based on CPU capabilities. This is analogous to ISPC's `--target=sse4-i32x4,avx2-i32x8`.

```lisp
#[spmd :targets [4 8]]
(spmd [i count]
  (Array/set result i (* (Array/get data i) 2.0)))
```

The compiler sorts and deduplicates the candidate widths, generates a function clone for each width, and selects the widest supported clone based on runtime CPU feature detection. Selection does not depend on the input-list order. If no SIMD candidate is usable, execution falls back to a separately generated whole-function scalar clone.

The implementation divides the target function into three forms:

- A wrapper with the original function name
- A whole-function scalar fallback clone
- A width clone for each `:targets` candidate

The wrapper calls `__singulisp_runtime_max_simd_bits()` exactly once and branches from the clone requiring the widest SIMD support to the narrowest. In each width clone, `spmd_width` is fixed statically, and LLVM attaches clone-specific `target-features`. Consequently, running loops contain no dynamic AoS/SoA-style check and no branch deciding whether that execution uses width four or eight.

## Tail Masking

When the element count is not divisible by the SIMD width, masked SIMD instructions process the remainder instead of a scalar epilogue. The compiler applies this optimization automatically.

With a scalar epilogue, each remainder iteration processes one element, so the epilogue can account for a large share of the runtime for short arrays. Tail masking activates only the valid lanes in the final strip and executes the same SIMD instructions. It uses facilities such as AVX-512 mask registers and ARM SVE predicate registers.

## ARM NEON / SVE Width Selection

Multi-target dispatch selects a fixed-width clone from the runtime target-feature class.

| Target | Detection Condition | Default SIMD Width |
|---|---|---|
| x86 SSE2 baseline | `+sse2` | 128 bits (`f32x4`) |
| x86 AVX2 | `+avx2` | 256 bits (`f32x8`) |
| x86 AVX-512 | `+avx512f` | 512 bits (`f32x16`) |
| ARM NEON | `+neon` | 128 bits (`f32x4`) |
| ARM SVE | `+sve` | 256-bit class (fixed-width clone) |

The current implementation does not expose SVE's scalable vector length directly. `spmd/lane-count` is the compile-time constant `W` of the selected width clone, not the hardware vector length at runtime.

## `spmd/tile` — 2D Tiling

Combines an outer scalar loop with inner SPMD vectorization over two-dimensional data. This form is intended for image processing and matrix operations.

```lisp
(spmd/tile [y height x width]
  (let idx (+ (* y width) x))
  (Array/set result idx (* (Array/get data idx) 2)))
```

`y` is the outer loop variable and advances sequentially from 0 through `height-1`. `x` is the inner SPMD variable and is subject to vectorization. The form desugars internally to `(for [y (range 0 height)] (spmd [x width] body))`.

## Vectorizing `if` as a Value

When an `if` expression is used as a value inside an SPMD block, the compiler automatically generates `SimdSelect`.

```lisp
(spmd [i count]
  (let v (Array/get data i))
  (let masked (if (> v 0) v 0))   ; converted to SimdSelect(mask, v, 0)
  (Array/set result i masked))
```

During vectorization, the merge point of the conditional branch—the phi node—is detected and converted to `SimdSelect(mask, then_value, else_value)`. This executes conditional computation without branches.

## Multi-Accumulator Unrolling

This optimization improves instruction-level parallelism (ILP) in `spmd/reduce` by splitting the loop-carried dependency of a reduction. The compiler applies it automatically.

```
; Conceptual transformation; no user-code changes are required
; Before: acc += load(i)  <- dependency chain on every iteration
; After:  acc0 += load(i); acc1 += load(i+W); acc2 += load(i+2W); acc3 += load(i+3W)
;         final = reduce(acc0 + acc1 + acc2 + acc3)
```

With `UF=4`, an unroll factor of four, the compiler generates four independent accumulators and combines them with a final tree merge. This applies to `+` and `*` reductions. It is enabled only when `count >= W * 4`; otherwise, execution falls back to a single accumulator.

## Automatic Prefetching

The vectorization pass analyzes memory-access patterns in an SPMD loop and automatically generates prefetch scheduling. Users can also control prefetching manually with `prefetch/read` and `prefetch/write`.

## Roofline-Model Analysis

The SPMD vectorization pass uses a roofline model internally to estimate the kernel's arithmetic intensity in FLOP/byte. It determines whether the kernel is memory-bound or compute-bound and uses that result to select an optimization strategy.

```
- flops: number of operations in the loop body (BinOp + CmpOp + MathUnary + MathBinary)
- bytes: number of memory-access bytes (Load + Store + Gather + Scatter)
- arithmetic_intensity: flops / bytes
- is_compute_bound: arithmetic intensity > threshold (default: 2.0)
```

## Memory-Access Pattern Diagnostics

After vectorization, the compiler analyzes memory-access patterns and produces performance warnings.

| Diagnostic Code | Description |
|-----------------|-------------|
| `W-PERF-0035` | Coherent branch pattern detected |
| `W-PERF-0036` | Noncontiguous gather/scatter access detected |
| `W-PERF-0037` | Large-stride access detected (`stride >= 4`) |
| `W-PERF-0038` | Redundant loads from the same base address detected |

These diagnostics appear in `gu-cli analyze --kind vectorize` output or in `optimization.entries` from `gu-cli build --report-opt`.

## Additional Error Codes

| Code | Description |
|---|---|
| E-SPMD-0045 | Too few arguments to `spmd/extract` |
| E-SPMD-0046 | Too few arguments to `spmd/broadcast-from` |
| E-SPMD-0047 | Empty `spmd/unmasked` body |
| E-SPMD-0048 | Too few arguments to `spmd/assume-uniform` |
| E-SPMD-0052 | Invalid `spmd/tile` arguments |
| E-SPMD-0053 | Empty `spmd/tile` body |
| E-SPMD-0054 | Too few arguments to `spmd/histogram` |

## Measuring Performance

The effect of SPMD depends on the target, CPU features, array length, alignment, and alias information. The specification does not define a fixed speedup. Use `defbench` and `gu-cli asm` or `analyze --kind vectorize` to compare the scalar and vectorized versions on the actual target and workload.

## Practical Example: Histogram Computation

```lisp
(defn compute-histogram
  [(data : [Array i32 4096])
   (histogram : [Array i32 256])] -> [Array i32 256]
  (let mut result histogram)
  (spmd [i 4096]
    (let val (Array/get data i))
    (spmd/atomic-add! result val 1))
  result)
```

This example represents updates that classify each data element into a bin. Because the current backend lowers `spmd/atomic-add!` to an ordinary load, add, and store, its accumulation semantics assume scalar fallback or a single CPU loop. Updating the same histogram from multiple OS threads is not thread-safe. The function returns the updated histogram array; arrays are value types.

## spmd/window — Sliding Window

`(spmd/window [i N :radius R] body)` declares a sliding window with the specified radius. It desugars internally to `(spmd [i N] body)` and records `window_radius` metadata. The user is responsible for boundary handling.

```lisp
(defn stencil [(data : [Array f32 1024])] -> [Array f32 1024]
  (let mut result data)
  (spmd/window [i 1024 :radius 1]
    (let left  (if (> i 0) (Array/get data (- i 1)) 0.0f))
    (let mid   (Array/get data i))
    (let right (if (< i 1023) (Array/get data (+ i 1)) 0.0f))
    (spmd/scatter! result i (/ (+ left mid right) 3.0f)))
  result)
```

`:radius` is an input to prefetch optimization and diagnostics. When the window radius exceeds the SIMD width, diagnostic `W-PERF-0039` is reported.

### Error Codes

| Code | Description |
|------|-------------|
| `E-SPMD-0060` | The binding does not have the form `[var count :radius R]` |
| `E-SPMD-0061` | The body is empty |
| `E-SPMD-0062` | The value of `:radius` is not a positive integer |

## spmd/lookup — Table Lookup

`(spmd/lookup table indices)` takes a fixed-length `Array` of at most 256 elements and an index from each lane, then returns the corresponding table elements. Dedicated register-resident lookup lowering keeps the table as a register value.

```lisp
(defn apply-lut [(data : [Array i32 256])
                 (lut : [Array i32 16])] -> [Array i32 256]
  (let mut result data)
  (spmd [i 256]
    (let idx (Array/get data i))
    (spmd/scatter! result i (spmd/lookup lut idx)))
  result)
```

The maximum table size is 256 elements.

### Error Codes

| Code | Description |
|------|-------------|
| `E-SPMD-0063` | The table exceeds the size limit |
| `E-SPMD-0064` | Two arguments are required: `table` and `indices` |

## spmd/group-reduce — Subgroup Reduction

`(spmd/group-reduce op init value :group-size G)` divides the SIMD lanes into subgroups of `G` lanes and performs a reduction within each group. It is implemented as a `log2(G)`-stage cascade of shuffle and binary operations.

```lisp
(defn group-sum [(data : [Array f32 1024])] -> [Array f32 1024]
  (let mut result data)
  (spmd [i 1024]
    (let x (Array/get data i))
    ;; Write each four-lane partial sum back to every lane in its group
    (spmd/scatter! result i (spmd/group-reduce + 0.0f x :group-size 4)))
  result)
```

`:group-size` must be a power of two. The operator must be associative, such as `+`, `*`, `min`, or `max`.

### Code-Generation Example (`G=4`, `W=8`, `op=Add`)

```
;; Stage 1: exchange adjacent pairs
s1 = shufflevector(val, undef, <1,0, 3,2, 5,4, 7,6>)
t1 = fadd(val, s1)
;; Stage 2: exchange half-groups
s2 = shufflevector(t1, undef, <2,3, 0,1, 6,7, 4,5>)
result = fadd(t1, s2)
```

### Error Codes

| Code | Description |
|------|-------------|
| `E-SPMD-0065` | Too few arguments; `op`, `init`, `value`, and `:group-size G` are required |
| `E-SPMD-0066` | `:group-size` is not a power of two |
| `E-SPMD-0067` | The operator is not associative |

## SPMD Kernel Fusion

When adjacent SPMD loops have the same count and a producer-consumer relationship, the compiler fuses them automatically. The `spmd_fuse` pass runs before `spmd_vectorize`.

### Fusion Patterns

**Same-array pattern**: Loop A writes `arr[i]` and loop B reads `arr[i]` → combine the bodies.

```lisp
;; Before fusion: two independent loops
(spmd [i 1024] (spmd/scatter! arr i (* (Array/get arr i) 2)))
(spmd [i 1024] (spmd/scatter! arr i (+ (Array/get arr i) 10)))

;; The compiler automatically fuses them into one loop:
;; compute arr[i] = (arr[i] * 2) + 10 in one pass
```

**Producer-consumer pattern**: Loop A writes `tmp[i]`, loop B reads `tmp[i]`, and `tmp` is unused after B → eliminate the intermediate array.

Application of fusion is reported by informational diagnostic `I-SPMD-0004`.

## Branch-Coherence Specialization

For a varying branch—one whose path may differ by lane—the compiler generates three-way dispatch that skips masked execution when every lane takes the same path at runtime.

```
mask_bits = bitcast mask to iW
if mask_bits == ALL_ONES → unmasked then-path (fast)
else if mask_bits == 0   → unmasked else-path (fast)
else                     → masked both-paths
```

This transformation is applied automatically when both the then-path and else-path contain at least two masked instructions. Its application is reported by informational diagnostic `I-SPMD-0003`.

## Vectorization of Mathematical Functions

Mathematical functions inside an SPMD block are automatically vectorized with LLVM vector intrinsics.

| Function | Vectorization Method |
|----------|----------------------|
| `sqrt`, `abs`, `floor`, `ceil` | LLVM vector intrinsic using native SIMD instructions |
| `sin`, `cos` | LLVM vector intrinsic, scalarized by LLVM |
| `tan`, `asin`, `acos` | Manual scalarization through extract, scalar call, and insert |
| `min`, `max`, `pow`, `copysign` | LLVM vector intrinsic |
| `atan2` | Manual scalarization |
