![logo](https://github.com/user-attachments/assets/92f67fdc-aa2c-4f32-9bf9-c84fa19e37c8)

# Singulisp

Singulisp is a **statically typed Lisp that prioritizes execution speed and predictable costs**.
Performance contracts are written into the source, and the compiler refuses to emit code that
cannot satisfy them. It combines a consistent S-expression syntax with ownership, regions,
first-class SIMD, built-in mathematical types, SPMD, GPU code generation, and per-function
performance inspection.

> Of machines, by machines, for machines.

## What it looks like

The following function computes the squared distance from a query point to the nearest point in a
collection stored in a structure-of-arrays (SoA) layout.

```lisp
(defstruct Point
  (x : f32)
  (y : f32)
  (z : f32))

#[no-alloc]
#[fp-mode :fast]
(defn nearest_distance_sq
  [(points : [& [SoaVec Point]])
   (query : Vec3)] -> f32
  (let qx query.x)
  (let qy query.y)
  (let qz query.z)
  (let count (SoaVec/len points))

  #[spmd :width 8 :require]
  (spmd [i count]
    (let dx (- (SoaVec/get-field points i x) qx))
    (let dy (- (SoaVec/get-field points i y) qy))
    (let dz (- (SoaVec/get-field points i z) qz))

    (spmd/reduce min (f32/INF)
      (+ (+ (* dx dx) (* dy dy))
         (* dz dz)))))
```

This example makes three points explicit in the source.

1. **The data layout matches the computation**

   `SoaVec` stores a variable-length collection of points by placing their `x`, `y`, and `z` fields
   in separate contiguous buffers. The query point is passed as a first-class `Vec3` value.

2. **Eight-lane SPMD is a performance contract**

   `SoaVec/len` determines the iteration count at runtime. If the count is not a multiple of eight,
   inactive lanes in the final group are masked off. With `:width 8 :require`, a release build must
   vectorize the block to eight lanes; otherwise, compilation fails with a diagnostic explaining
   why.

3. **Execution constraints are explicit**

   The function takes `points` by shared reference and reads from its existing storage without
   modifying it. `#[no-alloc]` prohibits dynamic allocation from `heap` or `arena`, including
   allocation performed by callees, while `#[fp-mode :fast]` declares which floating-point
   optimizations are permitted within the function.

You can inspect the generated assembly for this function directly:

```bash
gu-cli asm examples/performance/nearest-distance.gu \
  --function nearest_distance_sq \
  --release \
  --format asm
```

The last mile of hot-path optimization is usually iterative: change the source shape, compile,
inspect the assembly, and repeat until the compiler produces the code you want.

Singulisp is built around that loop. Function-unit compilation lets it check, optimize, and emit
assembly for the function being tuned without passing unrelated function bodies through the
backend. Even in a massive codebase, the edit–compile–inspect cycle stays short enough for
aggressive hot-path tuning.

## Performance contracts are compiler-checked

Performance requirements are written into the source and checked by the compiler. When a contract
cannot be satisfied, compilation reports why instead of silently selecting a slower path.

- In optimized builds, SPMD marked with `:require` does not silently fall back to scalar execution.
- Conflicting writes are not silently converted into atomic operations.
- A function marked `#[no-alloc]` fails to compile if it may perform dynamic memory allocation.
- If a slower path is required, its cost must be stated in the source.

## Core design

### 1. GC-free, region-based memory management

Ownership and borrowing control aliasing. For each allocation region, the allocator, lifetime,
threading constraints, capacity, and reuse policy can be declared as a contract. Memory management
remains an inspectable part of the program rather than being delegated to an implicit garbage
collector.

### 2. Explicit parallel computation

CPU SIMD, SPMD, and GPU code generation each have dedicated types, syntax, and diagnostics.
For SPMD, `:require` turns the requested vectorization into a performance contract. In addition
to low-level SIMD types, mathematical types such as `Vec2`, `Vec3`, `Quat`, and `Mat4` are
first-class language types rather than library conventions.

### 3. Machine-verifiable design

At function boundaries, types, effects, ownership, and region requirements are fixed. Allocation,
aliasing, alignment, address spaces, and slow-path fallbacks are represented explicitly in the
program structure, while generics and traits are statically monomorphized. Diagnostics are strict
and structured, and builds and analyses are also available through the MCP server.

## Quick start

```lisp
#[external]
(defn main [] -> ()
  (println! "Hello, Singulisp!"))
```

```bash
gu-cli run examples/getting-started/hello.gu
```

See [Installation](docs/getting-started/installation.md) for setup instructions and the
[Quick Tour](docs/getting-started/quick-tour.md) for an overview of the language.

## What Singulisp is not

- There is no GC, and none is planned.
- There is no implicit dynamic dispatch. Traits are statically monomorphized.
- There is no operator overloading. User definitions cannot replace the meaning of an operator.
- Singulisp is not ergonomics-first. When human convenience conflicts with machine verifiability,
  machine verifiability takes priority.

## Documentation

- [Examples](examples/README.md)
- [Documentation index](docs/index.md)
- [Language specification](docs/language/index.md)
- [Built-in types](docs/language/types.md)
- [First-class SIMD mathematical types](docs/language/math-types.md)
- [Standard library](docs/standard-library/index.md)
- [Performance features](docs/performance/index.md)
- [CLI reference](docs/tooling/cli.md)

## License

[Copyright 2026 singulisp](NOTICE)

Singulisp is provided under the [Apache License, Version 2.0](LICENSE).
