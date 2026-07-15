# Standard Library

Singulisp APIs are divided into **built-in APIs**, which the compiler lowers directly,
and the **`std` package**, which is implemented in Singulisp itself. This distinction is
important for understanding type semantics and optimization boundaries.

## Built-in APIs

- [Built-in Functions](builtins.md)
- [Built-in Math Functions](math.md)
- [Built-in Collections](collections.md)
- [Persistent Collections](persistent.md)
- [Strings](strings.md)
- [I/O and Processes](io.md)

## The `std` Package

The standard distribution includes the following modules.

| Module | Main facilities |
|---|---|
| `std::cmp` | `Eq`, `Ord` |
| `std::hash` | `Hash`, FNV-1a, and normalization helpers |
| `std::collections` | `HashMap`, `HashSet`, `Heap`, `Deque` |
| `std::option_result` | `Option` / `Result` combinators |
| `std::strings` | `StringBuilder` and string helpers |
| `std::math` | Interpolation, correction, curves, and noise |
| `std::num` | Explicit wrapping integer arithmetic |
| `std::sort` | Generic introsort |
| `std::ds` | Bitsets, DSU, segment trees, Fenwick trees, search, and radix sort |
| `std::graph` | CSR graph builder |
| `std::pool` | Generational handle pool |
| `std::rand` | PCG32-based deterministic RNG |
| `std::fixed` | Fixed-point arithmetic |
| `std::iter` | Iterator combinators for `Vec` |
| `std::json` | JSON parser and document handles |
| `std::serialize` | `SerializeAs` wire conversion |
| `std::geom` | Rays, AABBs, spheres, planes, and 2D rectangles |
| `std::color` | Packed and floating-point color conversion |
| `std::io` | `File` API and `IoError` |
| `std::spatial` | Morton order, grids, quadtrees, octrees, BVHs, SAP, and dynamic trees |
| `std::phys` | Deterministic rigid bodies, constraints, contacts, and mass properties |
| `std::text` | KMP, Z-arrays, suffix arrays, edit distance, and Aho–Corasick |
| `std::numth` | GCD, modular arithmetic, primality testing, factorization, and CRT |
| `std::ai` | A*, Dijkstra, behavior trees, GOAP, alpha-beta search, and MCTS |
| `std::nd` | `NdF64` arrays, broadcasting, mapping, reduction, and matrix multiplication |
| `std::linalg` | GEMM, LU, solving, Cholesky, QR, and eigenvalues |
| `std::fft` | FFT, inverse transforms, real FFT, convolution, and NTT |
| `std::signal` | Windows, biquads, FIR filters, and related signal processing |
| `std::stats` | Mean, variance, median, quantiles, correlation, and histograms |
| `std::dist` | Normal, exponential, and other distributions with batch generation |
| `std::optim` | Root finding, interpolation, Nelder–Mead, and related optimization |
| `std::ode` | RK4, RK45, Verlet, and leapfrog integrators |
| `std::sparse` | CSR, sparse matrix multiplication, and conjugate gradient |
| `std::compress` | LZ4 compression and decompression |
| `std::encode` | Hex, Base64, SHA-256, and UUID |
| `std::bigint` | Arbitrary-precision integers and `u128`-equivalent operations |
| `std::cgeo` | Triangulation, Delaunay, Voronoi, polygon processing, and rectangle packing |
| `std::net` | UDP, TCP, address resolution, and polling |
| `std::os` | Environment variables, arguments, paths, and directory operations |
| `std::replay` | Deterministic state replay and verification |
| `std::par` | Range partitioning, reduction, chunk mapping, and structured scopes |

See the [standard module reference](modules.md) for a summary of the public APIs.
