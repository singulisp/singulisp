# Singulisp Documentation

Singulisp is a general-purpose, statically typed Lisp that prioritizes execution speed and predictable
costs. This manual is organized to be read in order from introductory material through the normative
language specification, standard library, performance features, and tools.

## Getting started

1. [Installation](getting-started/installation.md)
2. [Hello, World!](getting-started/hello-world.md)
3. [Language Quick Tour](getting-started/quick-tour.md)
4. [Project Structure](getting-started/project-structure.md)
5. [Examples](../examples/README.md)

## Language specification

The [language specification index](language/index.md) covers lexical structure, special forms, types,
ownership, regions, generics, modules, and concurrency.

In addition to scalar types, the type system includes these compiler-provided types:

- Low-level SIMD: `f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, `u32x4`
- Mathematical types: `Vec2`, `Vec3`, `Vec4`, `Quat`, `Float3`, `Mat3`, `Mat3r`, `Mat4`, `Mat4r`, `Mat3x4`

See [Type System and Built-in Types](language/types.md) for the complete type list and
[First-Class SIMD Mathematical Types](language/math-types.md) for all mathematical operations.

## Standard library

The [standard library index](standard-library/index.md) clearly distinguishes compiler-provided APIs
from the `std` package implemented in Singulisp.

- [Built-in Functions](standard-library/builtins.md)
- [Collections](standard-library/collections.md)
- [Persistent Collections](standard-library/persistent.md)
- [Option / Result](standard-library/option-result.md)
- [Strings](standard-library/strings.md)
- [Mathematical Functions](standard-library/math.md)
- [I/O](standard-library/io.md)

## Performance features

- [Performance Features Index](performance/index.md)
- [Low-Level SIMD](performance/simd.md)
- [SPMD](performance/spmd.md)
- [GPU](performance/gpu.md)
- [Writing Fast Code](performance/writing-fast-code.md)
- [Memory Management Guide](guides/memory-management.md)

## Tools and platforms

- [CLI](tooling/cli.md)
- [Package Management](tooling/package-manager.md)
- [REPL](tooling/repl.md)
- [Formatter](tooling/formatter.md)
- [LSP](tooling/lsp.md)
- [MCP](tooling/mcp.md)
- [Testing and Benchmarking](tooling/testing.md)
- [Targets](platform/targets.md)
- [Capability](platform/capability.md)
- [Cross-Compilation](platform/cross-compilation.md)
- [FFI](platform/ffi.md)
- [Windows](platform/windows.md)

## Appendices

- [Grammar](appendix/grammar.md)
- [ABI](appendix/abi.md)
- [Glossary](appendix/glossary.md)
- [Diagnostic Codes](diagnostics/error-codes.md)
