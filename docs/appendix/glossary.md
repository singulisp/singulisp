# Glossary

This glossary lists terms related to Singulisp alphabetically, followed by additional reference entries.

---

## A

### ABI (Application Binary Interface)
The calling convention between compiled code units. It defines how function arguments are passed, how return values are returned, how registers are used, and related details. Singulisp performs ABI-level optimizations, including parameter and return-value optimization, in MIR optimization passes.

### ADT (Algebraic Data Type)
A sum type (tagged union) defined with `deftype`. It has multiple variants, selected through pattern matching.

```lisp
(deftype Shape
  (Circle f32)
  (Rect f32 f32))
```

### arena allocator
A general term for allocators that allocate memory within a region, including bump allocators and pool allocators. Individual `free` operations are unnecessary because the entire region is freed at once.

## B

### BBin (Singulisp Binary)
Singulisp's positional binary encoding. `serialize` adds a header containing a type descriptor to support schema evolution, while `bbin-write` outputs only a body intended for identical schemas. The header's `schema_id` is a 64-bit FNV-1a value computed from the type descriptor, not a payload checksum. When payload validation is required, compute a state hash separately.

## C

### Capability
A static mechanism for checking platform functionality. It requires capabilities such as `ext.time` and `ext.thread` on functions and verifies at compile time that they are available on the target platform.

### cfg-case / cfg-when
A conditional-compilation mechanism. `#[cfg-case <cfg-expr>]` selects definitions (exactly one must be active), while `#[cfg-when <cfg-expr>]` transforms calls (`:on-missing :erase-call` / `:stub` / `:error`).

### const-fn
A function evaluated at compile time. It is a pure function defined with the keyword syntax `(const-fn name [...] -> Type body)` and may be called within constant expressions.

### const-gen
A typed-staging mechanism. It generates and composes code fragments of type `Expr<T>` at compile time and embeds them in runtime code. The `~` (splice) operator expands expressions.

### CST (Concrete Syntax Tree)
A tree representation of source code that preserves information such as whitespace and comments. Singulisp lowers the CST into an AST.

## E

### effect call boundary
A mechanism that statically checks whether calls to ordinary, `external`, and `diagnostics` functions are permitted. A diagnostic gate removes calls through `cfg-when`; it is not a separate purity level. Ordinary functions may also perform local mutation, allocation, and traps.

### Expr\<T\>
The code-fragment type used by typed staging (`const-gen`). It represents an expression of type `T` at compile time and can be spliced with `~` into runtime code.

## H

### HAMT (Hash Array Mapped Trie)
The internal data structure of `PMap`. `PVec` uses a 32-way vector trie.

## I

### inkwell
An LLVM binding library for Rust. Singulisp uses inkwell 0.8.0, which supports LLVM 20, to generate LLVM IR and perform JIT execution.

## M

### MCP (Model Context Protocol)
A protocol for integration with large language models (LLMs). The `singulisp-mcp` binary provides a JSON-RPC 2.0-based MCP server that exposes build results, MIR dumps, and analysis results to LLMs.

### MIR (Mid-level Intermediate Representation)
An intermediate representation between the high-level IR and LLVM IR. It is based on a control-flow graph (CFG) and is processed by optimization passes such as constant folding, DCE, inlining, copy propagation, and NRVO.

### MPMC (Multi-Producer Multi-Consumer)
A channel implementation that multiple senders and multiple receivers can access concurrently. Singulisp's `ChannelHandle` refers to a mutex-and-condition-variable-based MPMC ring buffer. Its only payload type is `i32`; there is no generic `Channel<T>` surface type.

## N

### NRVO (Named Return Value Optimization)
A MIR optimization pass that eliminates unnecessary copies by constructing a function's return value directly in the caller's memory.

## P

### PMap (Persistent Map)
A HAMT-based persistent map. Update operations return a new version without destroying the original data. Structural sharing minimizes copying costs.

### PVec (Persistent Vector)
A persistent vector using a 32-way persistent vector trie and a tail. Appending or updating an element returns a new version without destroying the original data and structurally shares unchanged nodes.

## S

### S-expression
Lisp's fundamental syntactic form, consisting of parenthesized lists and atoms such as symbols, numbers, and strings. All Singulisp code is written as S-expressions.

```lisp
(defn add [(a : i32) (b : i32)] -> i32
  (+ a b))
```

### SoA (Structure of Arrays)
A memory layout in which a collection stores struct values as one array per field. In Singulisp, it is specified with `SoaArray` / `SoaVec` / `SoaPVec` / `SoaPMap`. This layout is useful for SIMD processing and improved cache efficiency.

Ordinary AoS (Array of Structures):
```
[{x,y,z}, {x,y,z}, {x,y,z}]
```

SoA:
```
{[x,x,x], [y,y,y], [z,z,z]}
```

### SPMD (Single Program, Multiple Data)
A model that executes one program over multiple data elements in parallel. Singulisp's `spmd` syntax performs automatic vectorization through strip mining based on uniformity analysis.

### std430
A standard memory-layout convention for GPU buffers. It applies to structs declared with `:repr gpu`, arranging field alignment and padding according to std430 rules.

## U

### uniformity
In SPMD analysis, the property that determines whether a variable has the same value in every lane (parallel execution unit). Uniform variables remain scalar, while only varying variables are vectorized. This avoids unnecessary vector operations.

## V

### vtable (Virtual Function Table)
A table of function pointers used at ABI boundaries such as platform and external-module interfaces. Singulisp surface traits are resolved statically and monomorphized; they have neither dynamic trait objects nor vtables for trait-method dispatch.

---

## Additional Terms

### arena allocator (cross-reference)
See **arena allocator** under A.

### bump allocator
A fast allocator that only advances a pointer. Individual allocations cannot be freed; the entire area is freed at once. It is ideal for the frame-arena pattern, in which the arena is reset every frame. Allocation cost is effectively O(1).

### region
A unit for managing a memory area. A region is defined with `defregion` and ties memory management to a scope. Objects within the region are freed together when the region ends. Regions are the foundation of deterministic memory management without a garbage collector.
