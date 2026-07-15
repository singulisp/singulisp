# Examples

Each example is self-contained in a single file. From the repository root, run executable examples
directly or inspect individual functions. Performance-oriented examples include benchmark kernels
with input sizes chosen for quick local runs.

```bash
gu-cli run examples/algorithms/heap-sort.gu --release
```

## Getting started

| Example | What it demonstrates |
|---|---|
| [hello.gu](getting-started/hello.gu) | A minimal executable program |

## Language features

| Example | What it demonstrates |
|---|---|
| [arena-tree.gu](language/arena-tree.gu) | Regions, a recursive algebraic data type, pattern matching, and arena reuse |
| [channels.gu](language/channels.gu) | Structured concurrency and a producer–consumer channel |
| [serialization.gu](language/serialization.gu) | The schema-evolution format with a type descriptor, and Raw BBin for identical schemas |

## Standard library

| Example | What it demonstrates |
|---|---|
| [string-parsing.gu](standard-library/string-parsing.gu) | Building strings with `StringBuilder`, then splitting them and parsing integers with `std::strings` |

## Performance features and numerical computing

| Example | What it demonstrates |
|---|---|
| [gpu-vector-scale.gu](performance/gpu-vector-scale.gu) | A shader that multiplies the values in a GPU buffer by a scalar, with CPU-side dispatch and WGSL output |
| [math-simd.gu](performance/math-simd.gu) | Low-level SIMD and first-class mathematical types |
| [nearest-distance.gu](performance/nearest-distance.gu) | `SoaVec`, `Vec3`, eight-lane SPMD, and reduction |
| [spectral-norm.gu](performance/spectral-norm.gu) | A numerical kernel over a variable-length `Vec<f64>` with `#[fp-mode :fast]` |

`nearest-distance.gu` is intended for function-unit assembly inspection.

```bash
gu-cli asm examples/performance/nearest-distance.gu \
  --function nearest_distance_sq \
  --release \
  --format asm
```

`gpu-vector-scale.gu` runs on the CPU reference backend by default. You can inspect the generated
WGSL without running the program.

```bash
gu-cli asm examples/performance/gpu-vector-scale.gu \
  --function scale_values \
  --format wgsl
```

When the Vulkan backend is available, the same file can be run with Vulkan.

```bash
SINGULISP_GPU_BACKEND=vulkan \
  gu-cli run examples/performance/gpu-vector-scale.gu --release
```

## Algorithms

| Example | What it demonstrates |
|---|---|
| [heap-sort.gu](algorithms/heap-sort.gu) | In-place heap sort using shared and mutable borrows of a `Vec` |
| [xoshiro256.gu](algorithms/xoshiro256.gu) | A xoshiro256++ pseudorandom number stream using `u64` rotations, shifts, and XOR operations |
