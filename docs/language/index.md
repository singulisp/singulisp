# Language Specification

This section describes Singulisp's surface language and its runtime-observable semantics.

## Syntax and Evaluation

- [Lexical Structure and S-expressions](syntax.md)
- [Definitions, Special Forms, and Expressions](forms.md)
- [Functions](functions.md)
- [Control Flow and Pattern Matching](control-flow.md)
- [Macros](macros.md)
- [Compile-time Computation](compile-time.md)

## Types and Data

- [Type System and Built-in Types](types.md)
- [First-class SIMD Mathematical Types](math-types.md)
- [Structs and Enums](structs-and-enums.md)
- [Generics and Traits](traits.md)
- [Option / Result and Error Propagation](error-handling.md)
- [Serialization](serialization.md)

In addition to numeric, string, reference, and collection types, the list of built-in types in `types.md` includes six low-level SIMD types and ten mathematical types. Mathematical types cannot be redefined as standard-library structs.

## Ownership and Execution Environment

- [Ownership and Borrowing](ownership.md)
- [Regions and Allocation](regions.md)
- [Effects](effects.md)
- [Concurrency](concurrency.md)
- [Modules](modules.md)
- [Attributes](attributes.md)

SIMD, SPMD, GPU, and performance contracts are covered under [Performance Features](../performance/index.md), while FFI is covered under [Platform Boundaries](../platform/ffi.md).
