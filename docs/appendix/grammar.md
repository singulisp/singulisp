# Syntax Reference (BNF-Style Overview)

This appendix presents Singulisp's principal syntax in a BNF-like notation. It is an overview intended to explain how to read the syntax, not an exhaustive parser grammar. The constraints and accepted forms for each definition are described on the corresponding language-specification pages.

## Notation

| Symbol | Meaning |
|------|------|
| `*` | Repeated zero or more times |
| `+` | Repeated one or more times |
| `?` | Optional |
| `\|` | Alternative |
| `'...'` | Literal token |

## Program Structure

```
program        ::= top-level-form*
top-level-form ::= definition | use-decl
```

The top level is a sequence of definitions and `use` declarations; arbitrary runtime expressions cannot appear there.

## Declarations

```
use-decl     ::= attribute* '(' 'use' module-path import-list? (':as' IDENT)? ')'
module-path  ::= IDENT ('::' IDENT)*
import-list  ::= '[' IDENT* ']'
```

## Definitions

```
definition   ::= fn-def | struct-def | type-def | trait-def
               | impl-def | region-def | macro-def
               | extern-def | test-def | bench-def | const-def
               | const-fn-def | const-gen-def | type-fn-def
               | interface-def
```

### Function Definitions

```
fn-def       ::= attribute* '(' 'defn' IDENT generic-params? params return-ann? where-clause? body ')'
generic-params ::= '[' generic-param+ ']'
generic-param ::= IDENT | LIFETIME | '(' IDENT ':' IDENT ')'
params       ::= '[' param* ']'
param        ::= IDENT | '(' IDENT ':' type ')'
return-ann   ::= '->' type
where-clause ::= ':where' '[' type-app+ ']'
body         ::= expression+
```

### Struct Definitions

```
struct-def   ::= attribute* '(' 'defstruct' IDENT generic-params? repr? field* ')'
repr         ::= ':repr' ('C' | 'simd' | 'gpu' | 'packed' | 'ordered' | 'aligned' INT)
field        ::= '(' IDENT ':' type ')'
```

`:repr` specifies the memory layout.

| repr | Description |
|------|------|
| `C` | C-compatible layout |
| `simd` | A 16-byte alignment hint for the corresponding allocation site; type size and stride remain unchanged |
| `gpu` | std430 layout |
| `packed` | Packed layout that removes padding while preserving declaration order |
| `ordered` | Preserves field order |
| `aligned N` | Pads to an N-byte boundary (N must be a power of two) |

The only repr permitted on `deftype` is `aligned N`. The other repr values apply only to `defstruct`.

### Algebraic Data Type Definitions

```
type-def     ::= attribute* '(' 'deftype' IDENT ('[' IDENT+ ']')? variant* ')'
variant      ::= IDENT | '(' IDENT variant-field* ')'
variant-field ::= type | '(' IDENT ':' type ')'
```

### Trait Definitions

```
trait-def    ::= attribute* '(' 'deftrait' IDENT generic-params? method-sig* ')'
method-sig   ::= '(' IDENT params '->' type ')'
```

### impl Blocks

```
impl-def     ::= attribute* '(' 'impl' generic-params? impl-head where-clause? method* ')'
impl-head    ::= type
               | type type+
               | type 'for' type
method       ::= '(' 'defn' IDENT params return-ann? body ')'
```

`impl-head` summarizes inherent impls, trait impls, and trait impls with `for`. See [Traits](../language/traits.md) for details on generic, const-generic, and lifetime parameters and `:where`.

### Region Definitions

```
region-def   ::= attribute* '(' 'defregion' REGION region-opt* ')'
region-block ::= '(' REGION SYMBOL expr+ ')'
               | '(' REGION ':reuse' SYMBOL loop-expr ')'
loop-expr    ::= '(' 'while' expr expr+ ')'
               | '(' 'for' vector expr+ ')'
               | '(' 'loop' vector expr ')'
region-opt   ::= ':' KEYWORD value
```

### Macro Definitions

```
macro-def    ::= '(' 'defmacro' IDENT '[' IDENT* ('&' IDENT)? ']' expr ')'
```

`&` introduces a variadic parameter.

### External Function Declarations

```
extern-def   ::= '(' 'extern' STRING extern-fn-decl* ')'
extern-fn-decl ::= '(' 'defn' IDENT params '->' type ')'
```

`STRING` specifies the ABI name, such as `"C"`.

### Test and Benchmark Definitions

```
test-def     ::= '(' 'deftest' STRING body ')'
bench-def    ::= '(' 'defbench' STRING ':iterations' INT body ')'
```

### Constant Definitions

```
const-def    ::= '(' 'const' IDENT (':' type)? expr ')'
```

### Compile-Time Function Definitions

```
const-fn-def  ::= attribute* '(' 'const-fn' IDENT generic-params? params return-ann? body ')'
const-gen-def ::= attribute* '(' 'const-gen' IDENT generic-params? params return-ann? body ')'
```

`const-fn` defines a function evaluated at compile time; `const-gen` defines a code-generation function that uses typed staging.

### Type-Function Definitions

```
type-fn-def  ::= '(' 'deftype-fn' IDENT '[' ('(' IDENT ':' IDENT ')')+ ']' ':repr' type ')'
```

`deftype-fn` defines a type-level function that determines a representation type from type parameters.

### ABI Interface Definitions

```
interface-def ::= '(' 'definterface' IDENT interface-method* ')'
interface-method ::= '(' IDENT params '->' type ')'
```

See [FFI](../platform/ffi.md#definterface-and-moduleload) for `definterface` type constraints and usage.

## Expressions

```
expression   ::= literal | IDENT | vector-literal | call | special-form
```

### Literals

```
literal      ::= INT | FLOAT | STRING | CHAR | BOOL | unit
unit         ::= '(' ')'
vector-literal ::= '[' expression* ']'
```

A keyword is not a value; it appears only as a syntactic marker such as `:where` or `:threads`.

### Function Calls

```
call         ::= '(' IDENT expression* ')'
```

The head of a call is a function-name symbol; an arbitrary expression cannot appear in callee position.

### Special Forms

```
special-form ::= let-expr | set-expr | if-expr | do-expr
               | match-expr | loop-expr | recur-expr | for-expr
               | break-expr | continue-expr
               | when-expr | unless-expr | cond-expr | while-expr
               | and-expr | or-expr | not-expr | try-expr
               | fn-expr | ref-expr | deref-expr | region-expr
               | unsafe-expr | scope-expr | spawn-expr | sync-expr | await-expr
               | channel-expr | spmd-expr | arena-expr
               | persist-expr | thaw-expr | freeze-expr
               | gpu-dispatch-expr
               | unquote-expr | expr-quote-expr
```

### Variable Bindings

```bnf
let-expr     ::= '(' 'let' 'mut'? IDENT (':' type)? expr ')'
set-expr     ::= '(' 'set!' place expr ')'
```

`let` creates an immutable binding, made mutable with `mut`; `set!` assigns to a mutable variable.

### Control Flow

```
if-expr      ::= '(' 'if' expr expr expr ')'
do-expr      ::= '(' 'do' expr* ')'
match-expr   ::= '(' 'match' expr match-arm* ')'
match-arm    ::= '[' guarded-pattern expr ']'
guarded-pattern ::= pattern | '(' pattern ':when' expr ')'
when-expr    ::= '(' 'when' expr expr+ ')'
unless-expr  ::= '(' 'unless' expr expr+ ')'
cond-expr    ::= '(' 'cond' cond-clause+ ')'
cond-clause  ::= '[' expr expr ']'
```

Because `if` is an expression, it always requires both a then clause and an else clause. `when` / `unless` desugar to `if`, and `cond` desugars to nested `if` expressions. Using `true` or `:else` in the final `cond` clause makes it the default clause.

### Logical Operations

```
and-expr     ::= '(' 'and' expr* ')'
or-expr      ::= '(' 'or' expr* ')'
not-expr     ::= '(' 'not' expr ')'
```

`and` and `or` use short-circuit evaluation. With no arguments, `(and)` returns `true` and `(or)` returns `false`.

### Error Propagation

```
try-expr     ::= '(' '?' expr ')'
```

`?` unwraps a `Result<T,E>` and returns early from the containing function when the result is `Err`.

### Loops

```
loop-expr    ::= '(' 'loop' '[' binding* ']' body ')'
binding      ::= '(' IDENT ':' type expr ')'
recur-expr   ::= '(' 'recur' expr* ')'
break-expr   ::= '(' 'break' ')'
continue-expr ::= '(' 'continue' ')'
for-expr     ::= '(' 'for' '[' IDENT '(' 'range' expr expr expr? ')' ']' body ')'
               | '(' 'for' '[' IDENT expr ']' body ')'
while-expr   ::= '(' 'while' expr expr+ ')'
```

`loop` is a tail-recursive loop, and `recur` advances to the next iteration. `break` / `continue` provide structured control over the nearest loop, and `while` desugars to `loop` plus `if`.

### Lambda Expressions

```
fn-expr      ::= '(' 'fn' '[' param* ']' return-ann? body ')'
```

### Reference Operations

```
ref-expr     ::= '(' '&' expr ')' | '(' '&' 'mut' expr ')'
deref-expr   ::= '(' '*' expr ')'
```

### Region Blocks

```
region-expr  ::= '(' REGION IDENT? body ')'
```

A block beginning with `REGION`, such as `@frame`, represents processing within a region. The second element is the region-handle name.

### Scopes and Concurrency

```
scope-expr   ::= '(' 'with-scope' IDENT body ')'
spawn-expr   ::= '(' 'scope/spawn' IDENT expr ')'
sync-expr    ::= '(' 'sync' expr+ ')'
await-expr   ::= '(' 'await' expr ')'
```

`with-scope` creates a structured-concurrency scope, and `scope/spawn` creates a task within it. `sync` runs multiple task functions in parallel and waits for all of them to complete.

### Channel Operations

```
channel-expr      ::= channel-new-expr | channel-send-expr
                     | channel-recv-expr | channel-close-expr
channel-new-expr  ::= '(' 'channel/new' expr ')'
channel-send-expr ::= '(' 'channel/send' expr expr ')'
channel-recv-expr ::= '(' 'channel/recv' expr ')'
channel-close-expr ::= '(' 'channel/close' expr ')'
```

`channel/new` creates a channel with a buffer size. `channel/send` and `channel/recv` send and receive messages. `channel/close` closes the channel.

### SPMD Blocks

```
spmd-expr    ::= '(' 'spmd' '[' IDENT expr ']' body ')'
               | '(' 'spmd/tile' '[' IDENT expr IDENT expr ']' body ')'
               | '(' 'spmd/window' '[' IDENT expr ':radius' INT ']' body ')'
               | '(' 'spmd/static-partition' '[' IDENT expr ':threads' INT ']' body ')'
```

`spmd` defines an SPMD (Single Program, Multiple Data) parallel-execution block.

### Arena Operations

```
arena-expr       ::= arena-alloc-expr | arena-try-alloc-expr | arena-reset-expr
arena-alloc-expr ::= '(' 'arena/alloc!' IDENT expr ')'
arena-try-alloc-expr ::= '(' 'arena/try-alloc' IDENT expr ')'
arena-reset-expr ::= '(' 'arena/reset!' IDENT ')'
```

`arena/alloc!` allocates a value in an arena and returns a reference. `arena/try-alloc` returns `[Option [& T]]` for `:on-oom :option` and `[Result [& T] AllocError]` for `:on-oom :result`. `try-alloc` itself cannot be used in a `:on-oom :trap` region.

### Persistence Operations

```
persist-expr ::= '(' 'persist' REGION expr ')'
thaw-expr    ::= '(' 'thaw' REGION expr expr expr ')'
freeze-expr  ::= '(' 'freeze' expr ')'
```

`persist` persists data across regions, `thaw` restores persistent data, and `freeze` freezes data.

### GPU Operations

```
gpu-dispatch-expr ::= '(' 'gpu/dispatch' IDENT ':threads' '[' expr expr expr ']' ':args' '[' expr* ']' ')'
                    | '(' 'gpu/dispatch-async' IDENT ':threads' '[' expr expr expr ']' ':args' '[' expr* ']' ')'
```

`gpu/dispatch` dispatches a GPU shader kernel.

### GPU Shared-Storage Built-in

`(gpu/shared (let ...))` is an ordinary builtin call, not a special form. It declares the argument's `let` expression as GPU shared storage. It has workgroup-shared semantics in both Vulkan and the CPU reference implementation. See [GPU Compute](../performance/gpu.md#declaring-shared-storage) for details.

### unquote During Macro Expansion

```
unquote-expr ::= '(' 'unquote' expr ')'
```

`unquote`, the formal name of `~`, splices an expression within a macro body. It is also used for `~` splicing in `const-gen`.

### Typed Staging

```
expr-quote-expr ::= '(' 'expr' expr ')'
```

`expr` converts an expression into a value of type `Expr<T>`. It is used within a `const-gen` function.

### unsafe Blocks

```
unsafe-expr  ::= '(' 'unsafe' body ')'
```

`unsafe` defines a syntactic boundary that marks unsafe operations. It lowers transparently into its body and does not relax type, ownership, or borrow checking.

## Patterns

```
match-arm       ::= '[' guarded-pattern body+ ']'
guarded-pattern ::= pattern | '(' pattern ':when' expr ')'
pattern         ::= '_'
                  | IDENT
                  | INT | STRING | BOOL | CHAR
                  | '(' IDENT pattern* ')'
                  | '(' 'ref' IDENT ')'
                  | '(' 'ref' 'mut' IDENT ')'
                  | '[' pattern* ']'
```

| Pattern | Description |
|---------|------|
| `_` | Wildcard |
| `IDENT` | Name of a variant without arguments, or a value binding |
| literal | Matches an integer, string, boolean, or character |
| `(Ctor p...)` | Destructures a variant with arguments; each `p` may be nested further |
| `(ref x)` | Reference pattern |
| `(ref mut x)` | Mutable-reference pattern |
| `[p0 p1 ...]` | Tuple pattern for a vector scrutinee |
| `(p :when expr)` | Guard evaluated after the pattern matches |

See [Control Flow](../language/control-flow.md#match-expressions) for examples of pattern exhaustiveness, nesting, and guards.

## Types

```
type         ::= IDENT
               | '(' ')'
               | reference-type
               | fn-type
               | type-app

reference-type ::= '[' '&' LIFETIME? type ']'
                 | '[' '&' 'mut' LIFETIME? type ']'
fn-type        ::= '[' 'Fn' '[' type* ']' '->' type ']'
type-app       ::= '[' IDENT type-arg+ ']'
type-arg       ::= type | NONNEG-INT | REGION-NAME
```

`type-app` is also the general form for user-defined generic types. Representative built-in applications are `[Box T]` / `[Box T @region]`, `[Vec T]` / `[Vec T @region]`, `[Array T N]`, `[SoaVec T]`, `[SoaArray T N]`, `[PVec T]`, `[PMap K V]`, `[SoaPVec T]`, `[SoaPMap K V]`, `[Option T]`, `[Result T E]`, `[GpuBuffer T]`, and `[Expr T]`.

### Basic Types

| Type | Description |
|----|------|
| `i8`, `i16`, `i32`, `i64` | Signed integers |
| `u8`, `u16`, `u32`, `u64` | Unsigned integers |
| `f32`, `f64` | Floating-point numbers |
| `bool` | Boolean |
| `String` | String |
| `()` | Unit type |

### SIMD Types

| Type | Description |
|----|------|
| `f32x2` | Two-element f32 vector |
| `f32x4` | Four-element f32 vector |
| `f64x2` | Two-element f64 vector |
| `i32x2` | Two-element i32 vector |
| `i32x4` | Four-element i32 vector |
| `u32x4` | Four-element u32 vector |

### Mathematical Types

| Type | Purpose and representation |
|----|-----------|
| `Vec2`, `Vec3`, `Vec4` | Two-, three-, or four-component f32 vector |
| `Quat` | Four-component quaternion |
| `Float3` | Packed 12-byte, three-component value for array storage |
| `Mat3`, `Mat4` | 3×3 or 4×4 column-major matrix |
| `Mat3r`, `Mat4r` | 3×3 or 4×4 row-major matrix |
| `Mat3x4` | Three-row by four-column row-major affine matrix |

See [Type System and Built-in Types](../language/types.md) and [First-Class SIMD Mathematical Types](../language/math-types.md) for each type's layout and operations.

## Attributes

```
attribute    ::= '#[' attr-name attr-arg* ']'
attr-name    ::= IDENT
attr-arg     ::= IDENT | KEYWORD | INT | STRING | '[' attr-arg* ']'
```

### Principal Attributes

| Attribute | Description |
|------|------|
| `#[pub]` | Exports outside the package |
| `#[priv]` | Visible only within the same folder |
| `#[external]` | Function with external side effects |
| `#[diagnostics]` | Function with diagnostic side effects |
| `#[shader]` | GPU shader kernel |
| `#[reloadable]` | Subject to hot reload |
| `#[tailrec]` | Requires tail-call optimization |
| `#[inline :hint]` | Relaxes the automatic-inlining threshold |
| `#[inline :always]` | Forces inlining of a pure, single-block function |
| `#[simd :hint :width N]` | SIMD-width hint |
| `#[spmd :width N]` | Specifies an SPMD width |
| `#[spmd :require]` | Requires SPMD transformation |
| `#[fp-mode :fast]` | Fast floating-point mode |
| `#[fp-mode :contract]` | Permits FMA contraction |
| `#[associative]` | Permits reassociation of floating-point operations |
| `#[require ext.*]` | Requires a capability |
| `#[cfg-case <expr>]` | Selects a definition conditionally |
| `#[cfg-when <expr>]` | Transforms a call conditionally |
| `#[derive Eq]` | Automatically derives a trait |
| `:repr C` | C-compatible layout |

## Tokens

```
IDENT        ::= [A-Za-z_][A-Za-z0-9_!?-]* ('/' [A-Za-z_][A-Za-z0-9_!?-]*)* ('.' [A-Za-z_][A-Za-z0-9_!?-]*)*
REGION       ::= '@' [A-Za-z_][A-Za-z0-9_!?-]*
LIFETIME     ::= '^' [A-Za-z_][A-Za-z0-9_!?-]*
INT          ::= [0-9]+ | '0x' [0-9a-fA-F]+ | '0b' [01]+
               | ('+' | '-') ([0-9]+ | '0x' [0-9a-fA-F]+ | '0b' [01]+)
FLOAT        ::= [0-9]+ '.' [0-9]* ('e' [+-]? [0-9]+)? 'f'?
STRING       ::= '"' char* '"'
CHAR         ::= '#\\' (named-char | .)
KEYWORD      ::= ':' [A-Za-z_][A-Za-z0-9_!?-]*
BOOL         ::= 'true' | 'false'
```

### Identifier Features

- `/` supports namespace-style names: `f32/sqrt`, `String/len`
- A `!` suffix suggests a mutating operation: `set!`, `map!`
- A `?` suffix suggests a predicate: `empty?`, `valid?`
- `-` supports kebab-case: `my-function`, `read-to-string`

### Integer-Literal Bases

| Prefix | Base | Example |
|--------------|------|-----|
| None | Decimal | `42` |
| `0x` | Hexadecimal | `0xFF` |
| `0b` | Binary | `0b1010` |

When a sign (`+` / `-`) is immediately followed by a digit, the sequence is parsed as a signed integer literal, for example `-128` or `+10`. With a space, `- 3` is parsed as an operator and a separate token. Digit separators such as `1_000` are not supported.

### Named Character Literals

| Literal | Character |
|---------|------|
| `#\space` | Space |
| `#\newline` | Newline |
| `#\tab` | Tab |
| `#\a` | The character 'a' |
