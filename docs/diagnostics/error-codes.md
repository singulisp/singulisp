# Diagnostic Codes

Singulisp compiler diagnostics generally use the form `E-{AREA}-{NUMBER}`. `E-` denotes an error,
`W-` a warning, `I-` optimization information, and `T-` a runtime trap.

This page lists the major codes users can act on; it is not an exhaustive catalog of every argument
check or optimization diagnostic. Because one code may be used by several closely related checks, use
the displayed message, span, and note as the final basis for interpretation.

## Look up a code on the command line

```bash
gu-cli explain E-MATCH-0001
gu-cli explain E-MATCH-0001 --format json
```

Even when the `explain` catalog has no detailed entry for a code, it reports the area and general
remediation guidance. Prefer a more specific message emitted during compilation when available.

## Interpreting areas

| Area | Primary stage |
|---|---|
| `LEX` / `READ` | Tokenization and reading S-expressions |
| `PARSE` / `LOWER` | Conversion from CST to language forms |
| `TYPE` / `IR` | Type inference, type checking, and IR construction |
| `OWN` / `BORROW` | Ownership, borrowing, and escape checks |
| `MATCH` | Pattern validity and ADT exhaustiveness |
| `CONST` / `STAGE` / `MACRO` | Compile-time evaluation and code generation |
| `RGN` / `ALLOC` | Regions, arenas, and allocation policies |
| `USE` / `MOD` / `COHERENCE` | Modules, imports, and trait implementations |
| `SPMD` / `SIMD` / `MATH` | Data-parallel operations and mathematical types |
| `GPU` | Validation of shaders and GPU resources |
| `DET` | Deterministic-math policy |

## Syntax and lowering

Representative syntax diagnostics include:

| Code | Meaning |
|---|---|
| `E-PARSE-0811` | A guard pattern does not have the three-element form `(<pattern> :when <expr>)` |
| `E-PARSE-0812` | A reference pattern is not of the form `(ref name)` or `(ref mut name)` |
| `E-PARSE-0813` | Invalid binding name in a reference pattern |
| `E-LOWER-0107` | `N` in `:repr aligned N` is not a power of two, or a trait-method parameter has an invalid form |
| `E-LOWER-0130` | `#[external]` was specified on a `const-fn` |
| `E-LOWER-0140` | `#[external]` was specified on a `const-gen` |
| `E-LOWER-0141` | A `const-gen` return type is not `[Expr T]` |
| `E-LOWER-0145` | An argument was passed to `break` |
| `E-LOWER-0146` | An argument was passed to `continue` |
| `E-IR-0220` | A `StrSlice` escapes into a return value, struct field, or enum payload |

See [Language Forms](../language/forms.md) and the [BNF Overview](../appendix/grammar.md) for form syntax.

## Types

| Code | Meaning |
|---|---|
| `E-TYPE-0501` | Cyclic expansion of `deftype-fn` |
| `E-TYPE-0502` | `deftype-fn` expansion depth exceeded the limit of 256 |
| `E-TYPE-1001` | Type mismatch during HM type inference |
| `E-TYPE-1002` | Infinite type detected by the occurs check |
| `E-TYPE-1003` | A type cannot be determined uniquely, or a function/trait call has the wrong arity |
| `E-TYPE-1006` | A numeric width cannot be determined uniquely |
| `E-TYPE-1007` | Unsafe implicit conversion, or no common type for branches or similar constructs |

First compare the function signature, annotations, and the types of both branches. Integers and
floating-point values, signed and unsigned values, and distinct nominal types are not implicitly
equated. Use explicit conversions such as `to-i32` or `to-f32` when necessary.

## Ownership and borrowing

| Code | Meaning |
|---|---|
| `E-OWN-0001` | A Move value was used after being moved |
| `E-OWN-0002` | Value construction, a move, or coercion violates ownership constraints |
| `E-BORROW-0001` | An attempt to create a mutable borrow while a shared borrow is active |
| `E-BORROW-0002` | An attempt to create a shared borrow while a mutable borrow is active |
| `E-BORROW-0003` | Overlapping mutable borrows |
| `E-BORROW-0004` | A closure captured a borrowed value whose capture is disallowed |
| `E-BORROW-0005` | A borrow or scoped-region value escaped or crossed a thread boundary |

A non-escaping closure passed directly to certain `Array` combinators may capture a shared borrow or
`& mut`. Region captures by task closures follow [Structured Concurrency](../language/concurrency.md).

## Pattern matching

| Code | Meaning |
|---|---|
| `E-MATCH-0001` | ADT variants are not covered exhaustively |
| `E-MATCH-0002` | A variant already covered by an unguarded arm is duplicated |
| `E-MATCH-0003` | Wrong payload arity in a variant pattern |
| `E-MATCH-0004` | Duplicate payload binding names within the same variant pattern |

```lisp
(match result
  [(Ok value) value]
  [(Err err)  (handle-error err)])
```

A `match` over scalar literals is lowered to nested `if` expressions, so it is handled separately from
the ADT exhaustiveness check `E-MATCH-0001`. See [Control Flow](../language/control-flow.md#match-expressions) for details.

## Const evaluation

| Code | Meaning |
|---|---|
| `E-CONST-0001` | An expression cannot be evaluated at compile time, or a variant cannot be resolved |
| `E-CONST-0002` | A variable cannot be referenced from a const expression |
| `E-CONST-0003` | The callee is not a `const-fn` or `const-gen` |
| `E-CONST-0004` | Arity mismatch in a const built-in, `const-fn`, or `const-gen` |
| `E-CONST-0005` | A divisor is zero in compile-time integer division or remainder |
| `E-CONST-0006` | An operator is unsupported by the compile-time evaluator |
| `E-CONST-0007` | Type mismatch in a compile-time operation |
| `E-CONST-0008` | Arity mismatch or nonnumeric argument in a compile-time numeric conversion |

## Typed staging

| Code | Meaning |
|---|---|
| `E-STAGE-0001` | A `const-gen` return type is not `[Expr T]` |
| `E-STAGE-0002` | Const-gen expansion exceeded the recursion limit of 128 |
| `E-STAGE-0003` | The target of `~` is not an allowed variable reference or const-gen call |
| `E-STAGE-0004` | `(expr ...)` was used outside a const-gen body, or the const-gen evaluator does not support an expression used in the body or within `(expr ...)` |

See [const-fn and Typed Staging](../language/compile-time.md) for the distinctions among `~name` and
`~(const-gen-call ...)` inside `(expr ...)` and top-level const splices.

## Macro expansion

| Code | Meaning |
|---|---|
| `E-MACRO-0001` | Call to an undefined macro |
| `E-MACRO-0002` | Wrong number of macro arguments |
| `E-MACRO-0003` | Invalid `defmacro` syntax, name, or parameter list |
| `E-MACRO-0901` | Expansion steps exceeded the limit of 65,536 |
| `E-MACRO-0902` | `unquote-splicing` used outside a list context |
| `E-MACRO-0903` | `unquote` or `unquote-splicing` used outside the corresponding quasiquote |

## Modules, use, and coherence

### use

| Code | Meaning |
|---|---|
| `E-USE-0001` | A module referenced by `use` was not found |
| `E-USE-0002` | A requested symbol is absent from the module |
| `E-USE-0003` | A symbol from another module was used without importing it |
| `E-USE-0004` | A symbol conflicts across reachable modules |

### module

| Code | Meaning |
|---|---|
| `E-MOD-0001` | Duplicate module name |
| `E-MOD-0002` | Duplicate symbol within a module, or a `definterface` boundary-type/contract violation |
| `E-MOD-0003` | A table type generated by `definterface` conflicts with an existing type |
| `E-MOD-0004` | Invalid arity, region, interface, or host service for `module/load` |
| `E-MOD-0005` | Invalid receiver, name, arity, or type on a module-interface method |

### Trait-implementation coherence

| Code | Meaning |
|---|---|
| `E-COHERENCE-0001` | Duplicate implementations for the same trait and unifiable type head |
| `E-COHERENCE-0002` | Orphan implementation in which neither the trait nor target type is defined in the current package |

To implement an external trait for an external type, define a newtype in the current package.

## Regions and arenas

### Definitions and attributes

| Code | Meaning |
|---|---|
| `E-RGN-0010` | The target region of `persist` lacks `:allow-persistent true` |
| `E-RGN-0011` | The target of `persist` is not a heap arena |
| `E-RGN-0101` | Invalid numeric pool slot size |
| `E-RGN-0102` | Unknown allocator or storage policy |
| `E-RGN-0103` | Required `:allocator` is missing |
| `E-RGN-0104` | Unknown lifetime policy |
| `E-RGN-0105` | Required `:lifetime` is missing |
| `E-RGN-0106` | Invalid thread policy, or use of `:async`, which is invalid as a region attribute |
| `E-RGN-0108` | Invalid growth strategy |
| `E-RGN-0109` | Unknown or disallowed out-of-memory policy |
| `E-RGN-0110` | Invalid reuse strategy |
| `E-RGN-0120` | A Boolean region attribute is neither `true` nor `false` |

### Allocation operations

| Code | Meaning |
|---|---|
| `E-RGN-0302` | Invalid combination of `arena/try-alloc` and out-of-memory policy |
| `E-RGN-0303` | `arena/alloc!` used with a fallible out-of-memory policy |
| `E-RGN-0310` | Raw arena allocation outside a bump arena |
| `E-RGN-0311` | `arena/reset!` used outside a bump arena |
| `E-RGN-0312` | A variable-sized or unknown-sized value stored in a pool |
| `E-RGN-0313` | A value exceeds the pool slot size |
| `E-RGN-0314` | Abort-scope allocation is outside a required active-region boundary, or a variable-length `String` / `Vec` uses pool-arena provenance |
| `E-RGN-0315` | A value requiring a destructor was allocated as a raw arena reference |
| `E-RGN-0316` | A region handle in an aggregate violates lifetime, definition, or field-order constraints |

The `C` series, such as `E-RGN-C01`, is used for constraints on region-attribute combinations. Consult
the message and the table in [Regions](../language/regions.md) for the specific combination.

## SPMD, SIMD, and mathematics

| Code | Meaning |
|---|---|
| `E-SPMD-0004` | A scalar local was modified with `set!` from an SPMD body |
| `E-SPMD-0006` | The count is not an integer, or injectivity of lane writes cannot be proved |
| `E-SPMD-0007` | Input and output may alias |
| `E-SIMD-0001` | Wrong arity for a fixed SIMD literal constructor |
| `E-SIMD-0007` | Wrong arity for `simd/splat` |
| `E-SIMD-0011` | SIMD types or widths differ between the two operands |
| `E-SIMD-0014` | Invalid lane-count literal |
| `E-SIMD-0016` | A SIMD lane fails a type constraint such as numeric, floating-point, or Boolean |
| `E-MATH-0020` | Wrong arity for a three-argument mathematical function |
| `E-MATH-0025` | Wrong arity for `remap` |

The write check for `E-SPMD-0006` uses affine indices derivable from `i`. To intentionally permit
conflicts among lanes within a single generated loop, use the `spmd/atomic-*` surface. The current
backend lowers these operations to ordinary load/compare/store sequences, however, so they are not
synchronization primitives across OS threads. A non-affine scatter is not an alternative proof of injectivity.

See [SPMD](../performance/spmd.md) for the SPMD surface and its diagnostics.

## GPU

The principal GPU-validator series are:

| Code | Meaning |
|---|---|
| `E-GPU-0101` | A `:repr gpu` type has a GPU-incompatible field |
| `E-GPU-0102` | A shader signature or body contains a GPU-incompatible type |
| `E-GPU-0103`–`E-GPU-0111` | A shader uses heap allocation, `String`, I/O, recursion, dynamic dispatch, closures, references, regions, or CPU concurrency |
| `E-GPU-0112` | The source of a wrapped buffer was accessed while borrowed or before dispatch completed |
| `E-GPU-0113` | The call graph contains transitive recursion |

See [GPU](../performance/gpu.md) for supported types and dispatch constraints.

## Allocation and deterministic mathematics

| Code | Meaning |
|---|---|
| `E-ALLOC-0001` | A `#[no-alloc]` function has an allocation effect |
| `E-ALLOC-0002` | A `#[no-heap-alloc]` function allocates on the heap or in a growable arena |
| `E-DET-0001` | `#[deterministic]` conflicts with `#[opt :math-kernel-expand false]` |
| `E-DET-0002` | IEEE FMA cannot be proved for the target, so deterministic mathematics cannot be generated |

## Warnings and optimization information

| Code | Meaning |
|---|---|
| `W-MATCH-0001` | Unreachable pattern |
| `W-MATCH-0002` | Redundant wildcard after every variant was listed explicitly |
| `W-SIMD-0003` | SPMD vectorization could not be applied and was skipped |
| `W-PERF-0012` | A heuristic suggests measuring the layout because a type size slightly exceeds a power-of-two boundary |
| `I-SPMD-0001` | SPMD vectorization was applied |

`W-PERF-0012` does not guarantee a performance improvement. Increasing alignment may increase size or
stride, so measure before and after field-layout changes on the actual target and workload.

## Runtime traps

| Code | Meaning |
|---|---|
| `T-ARITH-0001` | Integer division by zero |
| `T-ARITH-0002` | Signed `MIN / -1` or `MIN % -1`, or integer addition, subtraction, or multiplication overflow under `--overflow-trap` |
| `T-BOUND-0001` | Out-of-bounds array or vector access |
| `T-CHAN-0001` | Receive from a closed, empty channel |
| `T-CHAN-0002` | Send to a closed channel |
| `T-OOM-0001` | A region with trap policy exceeded its capacity |
| `T-GPU-0001` | GPU dispatch failed at runtime |

See [Language Forms](../language/forms.md) and [Regions](../language/regions.md) for trap policies and checked operations.
