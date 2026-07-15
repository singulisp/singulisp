# Standard Library Modules

This page indexes the public modules in the standard library. See
[Built-in Functions](builtins.md) for operations built directly into the language.

```lisp
(use std::collections [HashMap])
(use std::math :as math)
```

| Module | Main public APIs |
|---|---|
| `std::cmp` | `Eq[T]`, `Ord[T]`, and implementations for primitive types |
| `std::collections` | `HashMap`, `HashSet`, `Heap`, `Deque` |
| `std::color` | `Color32`, `ColorF`, color-space conversion, and interpolation |
| `std::ds` | Bitsets, `Dsu`, binary search, segment/Fenwick/sparse tables, and radix sort |
| `std::fixed` | `Fixed` fixed-point numbers with arithmetic, comparison, and square root |
| `std::geom` | `Ray`, `Aabb`, `Sphere`, `Plane`, `Rect2`, and intersection tests |
| `std::graph` | `GraphBuilder`, `Csr`, `DistNode` |
| `std::hash` | `Hash[T]`, FNV/Fx helpers, and floating-point normalization |
| `std::io` | `IoError`, file-existence checks, and whole-file I/O |
| `std::iter` | `Iterable[T]`, `Pair`, and eager helpers such as map, filter, fold, and zip |
| `std::json` | JSON parsing, object/array lookup, and typed accessors |
| `std::math` | Clamp, interpolation, angles, easing, Bézier curves, and noise |
| `std::num` | Wrapping arithmetic for `i32`, `i64`, `u32`, and `u64` |
| `std::option_result` | `Option` / `Result` map, and-then, or, and expect operations |
| `std::pool` | Generational `Handle` and `Pool[T]` |
| `std::rand` | PCG `Rng`, range generation, shuffle, and pick |
| `std::serialize` | `SerializeAs[Self Wire]` trait |
| `std::sort` | `sort`, `is-sorted` |
| `std::strings` | `StringBuilder`, search, split, join, formatting, and parsing |
| `std::spatial` | Morton order, grids, quadtrees, octrees, BVHs, SAP, MBP, and dynamic trees |
| `std::phys` | `World`, bodies/colliders, contacts, mass properties, and deterministic stepping |
| `std::text` | KMP, suffix arrays, LCP, edit distance, and Aho–Corasick |
| `std::numth` | GCD, LCM, modular exponentiation, primality testing, factorization, and CRT |
| `std::ai` | A*, JPS, flow fields, behavior trees, GOAP, alpha-beta search, and MCTS |
| `std::nd` | `NdF64`, views, broadcasting, element-wise operations, and reduction |
| `std::linalg` | `gemm`, LU, solving, Cholesky, QR, and eigenvalues |
| `std::fft` | FFT, IFFT, RFFT, convolution, and NTT |
| `std::signal` | Windows, biquads, and FIR filters |
| `std::stats` | Descriptive statistics, correlation, quantiles, histograms, and `describe` |
| `std::dist` | Normal, exponential, uniform, Poisson, and other distributions |
| `std::optim` | Root finding, interpolation, Nelder–Mead, and splines |
| `std::ode` | RK4, RK45, Verlet, and leapfrog integrators |
| `std::sparse` | `CsrF64`, sparse matrix multiplication, and CG |
| `std::compress` | `lz4/compress`, `lz4/decompress` |
| `std::encode` | Hex, Base64, SHA-256, and UUID |
| `std::bigint` | `BigInt`, `U128`, and arbitrary-precision arithmetic |
| `std::cgeo` | Triangulation, Delaunay, Voronoi, polygon operations, and packing |
| `std::net` | `Udp`, `Tcp`, address resolution, and polling |
| `std::os` | Environment, arguments, paths, and directory operations |
| `std::replay` | Replay containers, frame hashes, and divergence verification |
| `std::par` | `for-range`, `reduce`, `map-chunks`, and structured scopes |

## Collection Highlights

`HashMap` provides `new`, `with-capacity`, `len`, `put`, `insert`, `get`, `remove`,
`clear`, `to-vec`, `from-vec`, and index-based iteration. Key types must implement
`Eq[K]` and `Hash[K]`.

`HashSet` provides `new`, `insert`, `contains?`, `remove`, and `clear`; `Heap` provides
`push`, `peek`, and `pop`; and `Deque` provides push and pop operations at both ends.

## Strings and Conversion

`parse-i32`, `parse-i64`, and `parse-f64` in `std::strings` return `Option`.
By contrast, the language built-ins `String/parse-i64` and `String/parse-f64` return
`Result`, preserving detailed failure information.
