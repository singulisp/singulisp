# Singulisp

A statically-typed Lisp that refuses to emit slow code.\
**Of machines, by machines, for machines.**

------------------------------------------------------------------------

## What it looks like

``` lisp
;; SoA particle update — the compiler must vectorize this, or fail loudly.
#[spmd :width 8]
(defn integrate
  [(pos : [& mut [Vec f32]])
   (vel : [& [Vec f32]])
   (dt : f32)]
  -> ()
  (for [i (range (Vec/len pos))]
    (Vec/set! pos i
      (+ (Vec/get pos i)
         (* (Vec/get vel i) dt)))))
```

Three things happened when this compiled:

1.  **Ownership proved `pos` and `vel` don't alias**\
    → `noalias` reached LLVM as fact.

2.  **`#[spmd :width 8]` is a contract**\
    If vectorization failed, compilation would fail with the reason.

3.  **No allocation, no bounds check survives in the loop**\
    You can verify that yourself:

``` bash
gu-cli asm particles.gu --function integrate
```

Per-function compilation, per-function assembly.

Optimization is an inspection-driven loop, not an act of faith.

------------------------------------------------------------------------

# The compiler is the reviewer

Most languages generate slower code silently when an optimization fails.

Singulisp makes the failure your problem --- at compile time:

``` bash
$ gu-cli build sim.gu
```

``` text
error[E-SPMD-XXXX]: lanes may write the same address
`out[idx[i]]`
  --> sim.gu:14:5

  = help: intended conflict? Express its cost:
    `scatter-add!` / `atomic-add!`
```

No silent scalar fallback.\
No implicit atomics.\
No hidden allocation.

If a slow path is needed, the source must spell its cost.

------------------------------------------------------------------------

# Why

## The three pillars

### 1. GC-free, region-based memory

Every allocation commits to a **Region contract**:

-   allocator kind
-   legal operations
-   placement

Memory behavior is explicit and verifiable.

------------------------------------------------------------------------

### 2. One compute model

CPU SIMD, SPMD, and GPU are one coherent model with hard `require`
constraints.

SIMD math types:

-   `Vec3`
-   `Mat4`
-   `Quat`

are built into the language, not provided as a library.

------------------------------------------------------------------------

### 3. Machine-first ergonomics

Function boundaries seal:

-   types
-   effects
-   ownership
-   region requirements

Diagnostics are strict and structured.

**MCP is a first-class control plane.**

------------------------------------------------------------------------

# What exists today

TBD.

------------------------------------------------------------------------

# Quick start

TBD.

------------------------------------------------------------------------

# What Singulisp is not

-   No GC, and none planned.
-   No dynamic dispatch. Traits monomorphize statically.
-   No operator overloading, no shadowing of builtins. One name, one
    meaning.
-   Not ergonomics-first. When human convenience conflicts with machine
    verifiability, the machine wins.

------------------------------------------------------------------------

## Status

In development.

# License

Apache-2.0.
