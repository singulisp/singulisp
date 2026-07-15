# Attributes

Attributes attach metadata to definitions such as functions, types, and structs. They control compiler behavior or provide optimization hints. Write an attribute with the `#[...]` syntax immediately before its definition.

## Attribute Summary

| Attribute | Target | Description |
|------|------|------|
| `#[pub]` | Function/type/trait/use | Exports outside the package |
| `#[priv]` | Function/type/trait | Restricts visibility to the defining module |
| `#[external]` | Function | Declares I/O and OS side effects |
| `#[diagnostics]` | Function | Declares diagnostic side effects |
| `#[tailrec]` | Function | Requires tail-call optimization |
| `#[reloadable]` | Function | Marks a function for hot reload |
| `#[no-mangle]` | Function | Exports a C symbol |
| `#[require ext.*]` | Function | Declares a capability dependency |
| `#[shader :thread-group-size [X Y Z]]` | Function | Declares a GPU compute shader |
| `#[simd :hint :width N]` | Function | Hints a SIMD vectorization width |
| `#[spmd :width N]` | spmd block | Specifies an SPMD width |
| `#[spmd :require]` | spmd block | Requires vectorization (`E-SPMD-0010` on failure) |
| `#[spmd :min-trip N]` | spmd block | Specifies a minimum trip count |
| `#[spmd :targets [W1 W2]]` | spmd block | Specifies multiple target widths |
| `#[fp-mode :fast]` | Function | Permits all floating-point optimizations |
| `#[fp-mode :contract]` | Function | Permits FMA contraction |
| `#[associative]` | Function | Permits floating-point reassociation |
| `#[deterministic]` | Function / fn expression | Enables deterministic mathematical-kernel mode |
| `#[inline :hint]` | Function | Relaxes the inlining threshold |
| `#[inline :always]` | Function | Forces inlining of a pure, single-block function |
| `#[no-alloc]` | Function / extern declaration | Prohibits all allocation |
| `#[no-heap-alloc]` | Function / extern declaration | Prohibits heap allocation (bumping in a fixed trap arena is permitted) |
| `#[cfg-case <expr>]` | Function/type/const | Selects a definition conditionally |
| `#[cfg-when <expr> :on-missing <policy>]` | Function | Transforms a call conditionally |
| `#[opt :pass-name false]` | Function / defbench | Disables an optimization pass for one function |
| `#[specialize-helper-chain ...]` | Function | Sets limits for helper-chain specialization |
| `#[derive Trait1 Trait2 ...]` | defstruct / deftype | Automatically derives traits (Eq/Hash/Ord) |
| `#[serialize-opaque]` | defstruct | Suppresses automatic structural encoding of physical fields; participates through a Wire type when `T/to-wire` and `T/from-wire` exist |
| `#[unroll N]` | loop / for | Specifies the loop-unroll count |
| `#[unsafe]` | impl | Manually implements the Send/Sync marker |

---

## Visibility Attributes

### `#[pub]` --- Export Outside the Package

Makes a definition accessible from outside the package. Use it for a library's public API.

```lisp
#[pub]
(defn create-point [(x : f32) (y : f32) (z : f32)] -> Point3
  (Point3 x y z))

#[pub]
(defstruct Point3
  (x : f32)
  (y : f32)
  (z : f32))

#[pub]
(deftrait Drawable
  (draw [(self : Self)] -> ()))
```

### `#[priv]` --- Restricted to the Same Folder

Makes a definition accessible only from the same folder. It cannot be accessed from other folders in the same package.

```lisp
#[priv]
(defn internal-helper [(x : i32)] -> i32
  (* x 3))
```

With no visibility attribute, a definition has Package visibility and is accessible from within the same package.

---

## Effect Attributes

### `#[external]` --- External Side Effects

Declares that a function has side effects such as I/O or OS API calls. It cannot be called from a pure function.

```lisp
#[external]
(defn save-to-disk [(data : String)] -> ()
  (fs/write-string "output.txt" data))

#[external]
(defn main [] -> ()
  (save-to-disk "hello"))
```

### `#[diagnostics]` --- Diagnostic Side Effects

Apply this attribute to a function whose side effects are limited to diagnostic output such as debug logs. Unlike `#[external]`, it does not make the caller impure.

```lisp
#[diagnostics]
(defn log-debug [(msg : String)] -> ()
  (println! "[DEBUG] {}" msg))
```

See [Effect Call Boundaries](effects.md) for details.

---

## Allocation-Discipline Attributes

### `#[no-alloc]` --- No Allocation

Requires a function to have no allocation effect. The check applies not only to allocation nodes in the function body but transitively to allocations in callees. A violation produces `E-ALLOC-0001`.

```lisp
#[no-alloc]
(defn fixed-step [(dt : f32)] -> ()
  (update-positions dt))
```

An extern function called from a `#[no-alloc]` function must itself declare `#[no-alloc]`. An extern without that declaration is treated as potentially calling malloc internally.

```lisp
(extern "C"
  #[no-alloc]
  (defn c_pure_step [(dt : f32)] -> ()))
```

### `#[no-heap-alloc]` --- No Heap Allocation

This discipline is intended for real-time hot paths. It prohibits ordinary heap allocation, persistent collections, closure capture, channel or GPU-buffer creation, calls to undeclared externs, and similar operations. It does permit bump allocation in an `(arena :bump)` region whose settings are `:grow :fixed` and `:on-oom :trap`.

```lisp
(defregion @frame
  :allocator (arena :bump)
  :lifetime :scoped
  :default-capacity (KiB 64)
  :grow :fixed
  :on-oom :trap)

#[no-heap-alloc]
(defn frame-update [] -> i32
  (@frame fa
    (let mut v : [Vec i32 @frame] (Vec/new-in fa))
    (Vec/push (& mut v) 7)
    (Vec/len (& v))))
```

Bumping in a growable arena is prohibited under `#[no-heap-alloc]` because exhausting its capacity may allocate another backing block; the diagnostic is `E-ALLOC-0002`. Even with a fixed-capacity arena, `#[no-alloc]` prohibits the arena bump itself.

---

## GPU Attributes

### `#[shader :thread-group-size [X Y Z]]` --- GPU Shader

Marks a function as a GPU compute shader. The function is converted to WGSL code. `:thread-group-size` specifies the thread-group size; the default when omitted is `[64 1 1]`.

```lisp
#[shader :thread-group-size [256 1 1]]
(defn particle-update
  [(particles : [& mut [GpuBuffer Particle]])
   (dt : f32)]
  -> ()
  (let idx (gpu/thread-id-x))
  (let p (gpu-buf/read particles idx))
  (let new-pos (Vec3/add p.pos (Vec3/scale p.vel dt)))
  (gpu-buf/write particles idx (Particle new-pos p.vel)))
```

Types and operations used within a shader function are restricted to GPU-compatible ones. A user-defined struct requires `:repr gpu`. Primitive types, supported SIMD types, and fixed-size arrays are also available.

---

## Layout Selection

In Singulisp, select AoS or SoA through the collection type, not a struct attribute.

- Fixed-size SoA: `[SoaArray Particle N]`
- Variable-size SoA: `[SoaVec Particle]`
- Persistent-vector SoA: `[SoaPVec Particle]`
- Persistent-map SoA: `[SoaPMap Key Particle]`
- Whole-struct gather/scatter operations are separate, explicit APIs

```lisp
(defstruct Particle
  (x : f32)
  (y : f32)
  (z : f32)
  (life : f32))

(let ps : [SoaArray Particle 1024]
  (array ...))

(let x (SoaArray/get-field ps 10 x))
```

---

## SIMD / SPMD Attributes

### `#[simd :hint :width N]` --- SIMD Width Hint

Provides the compiler with a hint for the SIMD vectorization width. The compiler follows this hint when vectorizing the loop whenever possible.

```lisp
#[simd :hint :width 8]
(defn add-arrays [(a : [Array f32 1024])] -> ()
  ;; The compiler vectorizes this to process eight elements at a time with SIMD
  (Array/map! a (fn [(x : f32)] -> f32 (+ x 1.0f))))
```

### `#[spmd :width N]` --- SPMD Width

Specifies the width of SPMD (Single Program, Multiple Data) execution. Data is processed in parallel across the specified number of lanes.

Apply `#[spmd ...]` immediately before an `spmd` block, not a `defn`; it has no effect on a function definition.

```lisp
(defn process-data [(data : [Array f32 1024])] -> ()
  #[spmd :width 8]
  (spmd [idx 1024]
    (let val (Array/get data idx))
    (Array/set data idx (* val 2.0f))))
```

### `#[spmd :require]` --- Required Vectorization

This attribute makes vectorization of an `spmd` block mandatory. If the compiler cannot vectorize it, compilation fails with `E-SPMD-0010`; scalar fallback is not permitted. Like `#[spmd ...]`, apply the attribute immediately before the `spmd` block. Use it in performance-critical code to guarantee that vectorization has not silently been lost.

> **Enforcement behavior**: The requirement is checked only in optimized builds where the vectorization pass runs (O1 by default and O2 with `--release`). Because `--debug` (O0) performs no vectorization, it does not evaluate `:require`. If a reduction is folded into a closed form and the loop itself disappears, the requirement is considered satisfied because this is not scalar fallback. If conflicting `:width` / `:min-trip` settings cause vectorization to be skipped, `E-SPMD-0010` also applies, with the reason included in the error message. Inspect vectorization results with `gu-cli analyze src/main.gu --kind vectorize` or `gu-cli build src/main.gu --report-opt`.

```lisp
(defn double-all [(data : [Array f32 1024])] -> ()
  #[spmd :require]
  (spmd [idx 1024]
    (Array/set data idx (* (Array/get data idx) 2.0f))))
```

---

## Floating-Point Attributes

### `#[fp-mode :fast]` --- Fast Floating Point

Relaxes IEEE 754 conformance and permits every floating-point optimization. This enables FMA contraction, reassociation, reciprocal approximation, and similar transformations.

```lisp
#[fp-mode :fast]
(defn fast-magnitude [(x : f32) (y : f32) (z : f32)] -> f32
  (f32/sqrt (+ (* x x) (+ (* y y) (* z z)))))
```

> **Warning**: With `#[fp-mode :fast]`, results may differ from those required by IEEE 754. Do not use it for calculations where precision is critical.

### `#[fp-mode :contract]` --- Permitted FMA Contraction

Permits a multiplication and addition to contract into an FMA (Fused Multiply-Add) instruction. `a * b + c` compiles to a single FMA instruction.

```lisp
#[fp-mode :contract]
(defn dot-product [(a : Vec3) (b : Vec3)] -> f32
  (+ (* a.x b.x)
     (+ (* a.y b.y)
        (* a.z b.z))))
```

### `#[deterministic]` --- Deterministic Mode for Lockstep

Compiles a function or `fn` expression through deterministic mathematical kernels while retaining strict floating-point flags. `spmd-vectorize` is disabled by default to avoid changes in floating-point reduction order caused by target-specific SIMD widths. Combining this attribute with `#[opt :math-kernel-expand false]` is an error because falling back to libm or LLVM intrinsics would break determinism.

```lisp
#[deterministic]
(defn sim-step [(x : f64)] -> f64
  (f64/sin x))
```

Transcendental-function kernels in deterministic mode use `llvm.fma.*`, so they require a target with IEEE FMA, such as modern x86 FMA3 or ARM. Code generation returns `E-DET-0002` for a target on which FMA cannot be established through `+fma`, a known CPU baseline that includes FMA, or the AArch64 baseline.

### `#[associative]` --- Permitted Reassociation

Permits changes to the association order of floating-point operations. This is useful for parallelizing operations within a loop. It is an independent attribute, not an option of `#[fp-mode ...]`.

A closure passed to `Array/reduce` or `Array/scan` must have either `#[associative]` or `#[fp-mode :fast]`; omitting both is a compile-time error.

```lisp
#[associative]
(defn sum-array [(xs : [Array f32 1024])] -> f32
  (Array/fold 0.0f xs (fn [(acc : f32) (x : f32)] -> f32 (+ acc x))))
```

---

## Inlining Attributes

### `#[inline :hint]`, `#[inline :always]` --- MIR Inlining Requests

Provides a per-function hint to the `inline_small` pass. It is intended for small, pure helper functions.

```lisp
#[inline :hint]
(defn dot2 [(x : f32) (y : f32)] -> f32
  (+ (* x x) (* y y)))
```

```lisp
#[inline :always]
(defn madd [(a : f32) (b : f32) (c : f32)] -> f32
  (+ (* a b) c))
```

- `#[inline :hint]`: slightly relaxes the statement-count threshold for automatic inlining
- `#[inline :always]`: removes the statement-count threshold for a pure, non-recursive, single-block function
- `#[opt :inline false]` takes precedence when present
- The attributes have no effect on multi-block, impure, `#[reloadable]`, or recursive functions, and produce `W-PERF-0043`
- Applying `#[inline :always]` to a large function may produce `W-PERF-0044`

Use `#[inline :always]` as a last resort. First split the function into small, pure helpers and verify the result with `asm --release` and `bench`.

### `#[specialize-helper-chain :max-depth N :budget M]`

Overrides, for one function, the search limits used when specializing a build/consume chain for a temporary `Vec` returned by an internal pure helper chain into the caller's reduction loop.

- `:max-depth N`: expected limit on how many levels of a helper chain to follow; defaults to `3`
- `:budget M`: budget for the number of scalar statements that may be brought into the caller; defaults to the optimizer's value
- Apply the attribute to the caller; it has no effect on a helper
- Selection is limited by the budget as well as `max-depth`. Even with `depth=5`, specialization is not performed if the scalar slice is too large
- Begin with the default `depth=3`, and increase to `:max-depth 4` or `:max-depth 5` only for hot functions that win in same-tree A/B measurements
- `#[opt :specialize-helper-chain false]` takes precedence over this attribute

```lisp
#[specialize-helper-chain :max-depth 5 :budget 96]
#[external]
(defn hot-sum [] -> i32
  ...)
```

`#[opt :pass-name false]` can also disable local code-generation optimizations. Use it for performance A/B measurements and regression isolation. It may also be applied to `defbench`.

**Note**: `#[opt :pass-name true]` is a no-op. The parser collects only pairs whose value is `false`, so a `true` setting is silently ignored without an error or warning. There is no semantic operation that re-enables a pass after it has been disabled.

The following switches are useful for performance A/B measurements.

- `#[opt :block-param-phi false]`: disables only the fast path that lowers cheap block arguments in merge blocks and loop headers to `phi`
- `#[opt :simd-shuffle-build false]`: disables the fast path that constructs `SimdSplat` / `SimdIota` with `shufflevector` plus vector addition, using a chain of `insertelement` operations instead
- `#[opt :entry-invariant-direct false]`: disables the fast path that keeps invariant scalar or SIMD locals and scalar parameters defined exactly once in the entry/preheader in direct SSA form
- `#[opt :reuse-bounds-check false]`: disables the fast path that shares a bounds check already passed within the same block with subsequent accesses, emitting `icmp + branch + trap` every time instead
- `#[opt :aggregate-param-indirect false]`: disables the fast path that receives fixed-size `Array` / `SoaArray` value parameters through a read-only pointer plus copy-on-write, using `alloca + store` at entry instead
- `#[opt :array-construct-direct false]`: disables the fast path that materializes an `(array ...)` literal directly into the destination alloca when initializing a fixed-size `Array` / `SoaArray` local, routing it through `arr_tmp -> load -> store dst` instead
- `#[opt :const-array-rodata false]`: disables the fast path that interns and deduplicates constant fixed-size `Array` / `SoaArray` locals in a read-only global (`.rodata`), materializing them in the callee instead. This also covers nested arrays, arrays of struct elements, payloadless enum elements, NPO empty enum elements, and non-NPO payload enum elements with scalar, nested-array, or struct payloads. The default fast path seeks cross-module / LTO deduplication with deterministic global names plus `linkonce_odr`, `unnamed_addr`, and `comdat any`. An enum table containing a relocatable pointer payload automatically falls back to local materialization instead of this fast path
- `#[opt :ip-return-elide false]`: disables the fast path that elides an immediate dereference of `Box/new` / `arena/alloc!` returned by an internal direct helper across a call boundary, leaving `call -> alloc -> deref`
- `#[opt :small-domain-pmap-specialize false]`: disables the fast path that specializes count, histogram, and contains probes on a small-domain integer `PMap` into a fixed-size direct-address table, retaining the generic `PMap`
- `#[opt :dense-flag-specialize false]`: disables the entire family of specializations that converts a local dense-flag buffer implemented as a wide integer `Vec`, together with the internal direct helpers that consume it, to a bitset (`Vec<u64>`) or `Vec<u8>`
- `#[opt :narrow-const-vec-specialize false]`: disables the fast path that narrows a local integer `Vec` shape receiving only constant writes to `Vec<u8/i8/u16/i16>`, retaining the original integer element type
- `#[opt :fixed-local-vec-specialize false]`: disables both conversion of a ring-buffer local `Vec` to a fixed array and decomposition of a flat, word-stride local buffer into field-split local `Array` values
- `#[opt :word-stride-node-specialize false]`: disables only the `fixed-local-vec-specialize` fast path that decomposes a flat, word-stride local buffer into field-split local `Array` values
- `#[opt :scalar-bit-loop-specialize false]`: disables the entire family of fast paths that fold fixed-width bit-count, first-set, and leading-zero counted loops for `i32/u32/i64/u64` into `popcount`, `ctz`, or `clz` intrinsics
- `#[opt :bitset-scan-specialize false]`: disables the entire family of fast paths that convert count, first, next, last, prefix-count, and range-count operations that scan `bitset/get` one bit at a time into word-at-a-time popcount/ctz/clz/masked scans
- `#[opt :flip-prefix-specialize false]`: disables the fast path that specializes a prefix-reverse helper for a fixed-size `Array<i32, N>` (`N <= 16`) into a small-case switch plus swap ladder
- `#[opt :perm-rotation-specialize false]`: disables the fast path that specializes a prefix-left-rotation helper for a fixed-size `Array<i32, N>` (`N <= 16`) into a small-case switch plus shift ladder
- `#[opt :wrap-add-collapse false]`: disables the fast path that collapses the canonical `u32` wrapping-add helper (`to-u32(bit-and(+ (to-u64 a) (to-u64 b)) (to-u64 4294967295))`) into a direct add, retaining the u64 promotion plus mask
- `#[opt :rolling-bytes-kernel-specialize false]`: disables the family fast path that converts repeated rolling consumers over the same read-only byte source into a multi-target rolling comparison helper and tiny direct-table helpers
- `#[opt :affine-lcg-loop-specialize false]`: disables the fast path that converts an affine LCG helper loop visible after inlining (`seed = (seed * a + c) % m`) into a jump-ahead helper, retaining the counted loop
- `#[opt :specialize-helper-chain false]`: disables the fast path that specializes the build/consume chain of a temporary `Vec` returned by an internal direct helper, forwarding wrapper, or pure transform-helper chain into the caller's reduction loop, retaining `call -> VecLen -> VecGet`. This setting takes precedence even when a depth-override attribute is present
- `#[opt :specialize-helper-filter-count false]`: disables the fast path that folds a caller that only observes `Vec/len` on a result filtered by conditional `Vec/push` in an internal direct helper into a count loop across the call boundary, retaining `call -> VecLen`
- `#[opt :delayed-option-materialize false]`: disables the fast path that converts an immediate `match` on a two-variant sum type (`Option<T>`, `Result<T,E>`, or a custom two-variant enum) returned by an internal direct helper from `call -> GetTag -> Switch -> GetPayload` into a direct `Branch + payload scalar slice`. Multi-variant enum families are not covered
- `#[opt :complete-tree-flat-payload-bufferize false]`: disables the fast path that replaces repeated traversal through an iterative payload-sum wrapper or wrapper chain over a complete binary tree with scalar payloads with materialization of a flat payload-buffer local (`Vec<i32>`) plus linear-sum helpers, retaining repeated traversal of the materialized tree
- `#[opt :complete-tree-flat-payload-mixed-consumer-specialize false]`: disables the fast path that replaces mixed repeated traversal by a direct iterative payload sum and a weighted wrapper or wrapper chain over a complete binary tree with scalar payloads with materialization of a flat payload-buffer local (`Vec<i32>`) plus mixed helpers, retaining repeated traversal of the materialized tree
- `#[opt :complete-tree-flat-payload-even-sum-specialize false]`: disables the fast path that replaces repeated iterative even-sum traversal over a complete binary tree with scalar payloads with materialization of a flat payload-buffer local (`Vec<i32>`) plus a branchless even-sum helper, retaining repeated traversal of the materialized tree
- `#[opt :complete-tree-flat-payload-compare-sum-specialize false]`: disables the fast path that replaces repeated iterative compare-sum traversal over a complete binary tree with scalar payloads with materialization of a flat payload-buffer local (`Vec<i32>`) plus a linear compare-sum helper, retaining repeated traversal of the materialized tree
- `#[opt :complete-tree-flat-payload-generic-consumer-specialize false]`: disables the fast path that replaces repeated traversal by an unknown iterative consumer over a complete tree with scalar payloads with materialization of a generic flat payload-buffer local (`Vec<i32>`) plus a straight-line scalar recurrence helper, retaining repeated traversal of the materialized tree
- `#[opt :aos-field-bufferize false]`: disables the fast path that expands a hot loop over a fixed-size `[Array Struct N]` parameter into shadow field arrays, retaining whole-struct AoS access
- `#[opt :soa-field-scalarize false]`: disables the transformation that turns narrow `SoaArray/gather` / `scatter` operations back into the `get-field` / `set-field` fast path, retaining whole-struct access
- `#[opt :vec-len-elide false]`: disables the fast path that folds a temporary `Vec` used only by `Vec/push` and `Vec/len` into a length counter, constructing the actual `Vec`
- `#[opt :vec-pipeline-elide false]`: disables the fast path that folds an append-only temporary `Vec` directly into a following `Vec/get` plus `+` / `max` / `min` reduction loop without materializing it, constructing and then reading the `Vec`

Many of the latter switches are intended for regression isolation. `simd-shuffle-build`, `entry-invariant-direct`, `reuse-bounds-check`, `aggregate-param-indirect`, `array-construct-direct`, `const-array-rodata`, `ip-return-elide`, `narrow-const-vec-specialize`, `flip-prefix-specialize`, `perm-rotation-specialize`, `wrap-add-collapse`, `affine-lcg-loop-specialize`, `specialize-helper-chain`, `specialize-helper-filter-count`, `delayed-option-materialize`, the tree/array field families, and similar optimizations have individual toggles. By contrast, the public surface for `dense-flag-specialize`, `fixed-local-vec-specialize`, `scalar-bit-loop-specialize`, `bitset-scan-specialize`, and `rolling-bytes-kernel-specialize` consists of family toggles.

---

## Conditional-Compilation Attributes

### `#[cfg-case <expr>]` --- Conditional Definition Selection

Provide multiple definitions with the same name and select at compile time which one to use based on the build configuration. Exactly one definition in a same-name group must be active.

```lisp
#[cfg-case (os :linux)]
(defn get-timer [] -> f64
  (clock-gettime))

#[cfg-case (os :windows)]
(defn get-timer [] -> f64
  (query-performance-counter))

#[cfg-case (os :macos)]
(defn get-timer [] -> f64
  (mach-absolute-time))
```

Related error codes:

| Error code | Description |
|------------|------|
| E-CFG-0001 | A cfg-case condition evaluated to Missing |
| E-CFG-0002 | No definition is active in a cfg-case group |
| E-CFG-0003 | Multiple definitions are active in a cfg-case group |

### `#[cfg-when <expr> :on-missing ...]` --- Conditional Call Transformation

Controls the presence of a function according to the build configuration. `:on-missing` specifies what happens when the condition is not satisfied.

```lisp
#[cfg-when (key= :profiling "true") :on-missing :erase-call]
(defn profile-begin [(label : String)] -> ()
  ;; Begin profiling
  )
```

Options for `:on-missing`:

| Option | Behavior |
|-----------|------|
| `:erase-call` | Removes the call completely |
| `:stub` | Replaces the call with a stub (a function that does nothing) |
| `:error` | Produces a compile-time error |

Related error codes:

| Error code | Description |
|------------|------|
| E-CFG-0004 | An argument to cfg-when :erase-call is not an atom |
| E-CFG-0005 | The cfg-when policy is :error, so the function is unavailable |

Adding `:kind :diagnostics` to `#[cfg-when]` makes it a call gate for a `#[diagnostics]` function. In a non-diagnostic build, the call is removed completely and does not make the caller impure. See [Effect Call Boundaries](effects.md) for details.

```lisp
#[cfg-when (:key= :diagnostics "true") :on-missing :erase-call :kind :diagnostics]
(defn trace-event [(label : String)] -> ()
  (log-debug label))
```

---

## Trait-Derivation Attributes

### `#[derive Trait1 Trait2 ...]` --- Automatic Trait Derivation

Automatically generates implementations of the specified traits for a struct (`defstruct`) or enum (`deftype`).

Supported traits and generated methods:

| Trait | Generated methods |
|---------|-----------------|
| `Eq` | `TypeName/eq` (compares every field of a struct for equality; for an enum, compares the variant and then payloads) |
| `Hash` | `TypeName/hash-into` + `TypeName/hash`; for a struct, also generates the internal `TypeName/keq` helper used by the collection runtime |
| `Ord` | `TypeName/cmp` (lexicographic field order for a struct; variant declaration order followed by payload order for an enum) |

```lisp
#[derive Eq Hash]
(defstruct Point
  (x : i32)
  (y : i32))

#[derive Eq Ord]
(deftype Color Red Green Blue)

#[derive Ord]
(defstruct Priority
  (level : i32)
  (seq   : u64))
```

---

## Test / Benchmark Syntax

Tests and benchmarks are defined with dedicated keyword syntax, not attributes.

### `deftest` --- Test Definition

Define a test with the `deftest` keyword. Tests are run by the `gu-cli test` command.

```lisp
(deftest "addition"
  (assert-eq! (+ 2 3) 5)
  (assert-eq! (+ -1 1) 0))
```

### `defbench` --- Benchmark Definition

Define a benchmark with the `defbench` keyword. Benchmarks are run by the `gu-cli bench` command.

The body of a `defbench` is treated as a pure context. It cannot call side-effecting APIs such as `println!`, `fs/*`, `io/*`, or `stream/*`. Run measurements that include I/O as an ordinary program.

```lisp
(defbench "vector-add benchmark" :iterations 1000000
  ;; Operation being benchmarked
  (let a (Vec3 1.0f 2.0f 3.0f))
  (let b (Vec3 4.0f 5.0f 6.0f))
  (Vec3/add a b))
```

---

## Hot-Reload Attribute

### `#[reloadable]` --- Subject to Hot Reload

Allows a function to be reloaded while running in Debug + hot mode, improving iteration speed during development.

Constraints:

- Cannot be applied to a generic function (E-HOT-0001)
- Cannot be combined with `#[shader]` (E-HOT-0002)
- Can be applied only to a non-capturing function with a fixed ABI

```lisp
#[reloadable]
(defn update-state [(state : AppState)] -> AppState
  ;; This operation can be changed while the program is running
  (let new-state (process-input state))
  (update-physics new-state))
```

---

## Capability Attribute

### `#[require ext.*]` --- Capability Requirement

Declares that a function requires a particular external capability. Use it to control access to platform-specific functionality. An unknown capability name produces `E-CAP-0002`.

Known capability names:

| Name | Meaning |
|------|------|
| `ext.time` | Time access |
| `ext.thread` | Threads |
| `ext.sync` | Synchronization primitives |
| `ext.fs` | File system |
| `ext.dl` | Dynamic-library loading |
| `ext.io` | I/O |
| `ext.network` | Networking |

```lisp
#[require ext.fs]
(defn load-asset [(path : String)] -> [Result Asset String]
  (let data (? (fs/read-to-string path)))
  (parse-asset data))
```

### `#[tailrec]` --- Tail-Call Optimization

Verifies that every self-recursive call in the function is in tail position and requires automatic conversion to `loop`/`recur`. Self-recursion outside tail position is a compile-time error.

```lisp
#[tailrec]
(defn sum-to [(n : i32) (acc : i32)] -> i32
  (if (<= n 0)
    acc
    (sum-to (- n 1) (+ acc n))))
```

---

## C Linkage Attribute

### `#[no-mangle]` --- Export as a C Symbol

Exports the function name as a C symbol without mangling, making it available to other languages at link time. The function is also treated as a root for dead-code elimination (DCE), so it is retained even when unreferenced.

```lisp
#[pub]
#[no-mangle]
(defn app_init [(width : i32) (height : i32)] -> i32
  ...)

#[no-mangle]
(defn app_update [(dt : f32) (state : [& mut AppState])] -> ()
  ...)
```

Use `gu-cli emit-object` to generate an object file (`.o`) that can be linked from C or C++.

---

## Compile-Time Execution Syntax

### `const-fn` --- Compile-Time Function

`const-fn` is keyword syntax that defines a function evaluable at compile time. It may be used on the right-hand side of a `const` declaration or to compute type arguments.

```lisp
(const-fn max-particles [] -> i32
  (* 1024 64))

(const MAX_PARTICLES (max-particles))
```

Operations with side effects, such as I/O and memory allocation, cannot be used within a `const-fn` function.

---

## The `:repr` Modifier

Specifying `:repr` on a `defstruct` or `deftype` controls its in-memory representation.

**Note**: `:repr` is not a `#[...]` attribute. It is a keyword modifier written in the body of a `defstruct` / `deftype` definition. There is no `#[layout ...]` attribute; using one is a compile-time error.

| `:repr` | Description |
|---------|------|
| `:repr C` | C-compatible memory layout |
| `:repr simd` | Hint requiring 16-byte alignment at the corresponding stack / arena allocation site |
| `:repr gpu` | GPU-compatible layout (std430) |
| `:repr packed` | Removes padding in declaration order and handles unaligned field access |
| `:repr ordered` | Fixes field order to definition order |
| `:repr aligned N` | Pads to an N-byte boundary (N must be a power of two) |

### `:repr C` --- C-Compatible Layout

Use this representation for structs exchanged with C libraries through FFI. It follows C memory-layout rules.

```lisp
(defstruct SDL_Rect :repr C
  (x : i32)
  (y : i32)
  (w : i32)
  (h : i32))
```

### `:repr simd` --- SIMD Allocation Hint

Requires 16-byte alignment at the corresponding stack or arena allocation site. It does not change the struct type's own ABI alignment or tail padding, so the size and array stride of a struct containing three `f32` fields remain 12 bytes.

```lisp
(defstruct Aligned4 :repr simd
  (x : f32)
  (y : f32)
  (z : f32)
  (w : f32))
```

### `:repr gpu` --- GPU-Compatible Layout

Follows the GPU std430 layout rules. It is required for a struct used in a `#[shader]` function.

```lisp
(defstruct GpuParticle :repr gpu
  (position : Vec3)
  (velocity : Vec3)
  (life : f32)
  (size : f32))
```

### `:repr packed` --- Packed Representation

Lays out fields in declaration order without padding. Reads and writes of a field that does not satisfy its natural alignment are performed as unaligned accesses.

```lisp
(defstruct CompactHeader :repr packed
  (magic : i32)
  (version : i32)
  (flags : i32))
```

### `:repr ordered` --- Fixed Definition Order

Prohibits compiler optimization that reorders fields and preserves definition order directly in the memory layout.

```lisp
(defstruct FileHeader :repr ordered
  (magic : i32)
  (version : i32)
  (size : i64))
```

### `:repr aligned N` --- Alignment Specification

Pads the type's size to an N-byte boundary. N must be a power of two (`E-LOWER-0107`). This representation may be used with both `defstruct` and `deftype`.

Use it to improve cache-line efficiency or guarantee alignment for arena allocations. Alignment is guaranteed automatically for both stack variables and arena allocations.

```lisp
; Struct alignment
(defstruct CacheLine :repr aligned 64
  (data : i64)
  (tag  : i32))

; Enum alignment
(deftype Tree :repr aligned 16
  Leaf
  (Node [& Tree] [& Tree]))
```

---

## Loop Attribute

### `#[unroll N]` --- Loop Unrolling

Apply this attribute immediately before a `loop` or `for` block to specify the loop-unroll count. N must be a positive integer. A constant symbol may also be used.

```lisp
#[unroll 4]
(loop [(i : i32 0)]
  (when (< i 100)
    (process i)
    (recur (+ i 1))))

#[unroll 8]
(for [x data]
  (process x))
```

---

## impl-Only Attribute

### `#[unsafe]` --- Manual Send / Sync Marker Implementation

Declares a manual marker implementation of the `Send` / `Sync` trait. It may be used only on an `impl` with no methods. Use it only when manually guaranteeing the type's safety.

```lisp
#[unsafe]
(impl Send for MyHandle)

#[unsafe]
(impl Sync for SharedState)
```

---

## Attribute-Application Restrictions

| Restriction | Description |
|------|------|
| Attributes prohibited on `defmacro` | Attributes on a `defmacro` definition are unsupported and ignored by `warn_unknown_attributes`. |
| Attributes prohibited on `deftype-fn` / `defregion` | These definition forms cannot have attributes. |
| `#[layout ...]` does not exist | Specify layout with the `:repr` keyword in the `defstruct` / `deftype` body. |
| `#[inline :always]` cannot be combined with `#[reloadable]` | Produces W-PERF-0043. |
| `#[deterministic]` cannot be combined with `#[opt :math-kernel-expand false]` | Produces E-DET-0001. |
