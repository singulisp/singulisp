# Performance Features

Singulisp expresses performance through types, data layout, function attributes, and diagnostics rather
than leaving it solely to compiler heuristics. This section covers language features and usage guidance
directly related to performance.

- [Low-Level SIMD](simd.md) — surface-language SIMD types, lane operations, and explicit loads and stores
- [First-Class SIMD Mathematical Types](../language/math-types.md) — vectors, quaternions, matrices, and all their operations
- [SPMD](spmd.md) — lane-parallel execution and conflict rules
- [GPU](gpu.md) — the shader subset, buffers, dispatch, and output formats
- [Writing Fast Code](writing-fast-code.md) — allocation, aliasing, bounds, and data layout
- [Memory Management Guide](../guides/memory-management.md) — ownership and regions in practice
- [Per-Function CLI Tools](../tooling/cli.md#function-unit-inspection) — IR, MIR, assembly, and analysis

Low-level SIMD and mathematical types are separate type systems. The lane width selected internally by
the SPMD optimizer does not necessarily correspond to one of the SIMD types constructible in the surface language.
