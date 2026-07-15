# ABI Specification

This document specifies Singulisp's application binary interface (ABI), including C runtime symbols, type layouts, serialization formats, and calling conventions.

## C Runtime Symbols

Runtime functions use the `__singulisp_` prefix for internal symbols or the `singulisp_` prefix for core functions. These prefixes prevent name collisions with user code and external libraries.

### Memory Management

| Symbol | Signature | Description |
|---------|-----------|------|
| `__singulisp_mem_alloc` | `(size: size_t) -> *mut void` | Allocates memory |
| `__singulisp_mem_realloc` | `(ptr: *mut void, new_size: size_t) -> *mut void` | Reallocates memory |
| `__singulisp_mem_free` | `(ptr: *mut void) -> ()` | Frees memory |
| `__singulisp_mem_dup` | `(src: *const void, size: size_t) -> *mut void` | Duplicates memory |

### String Operations

String operations use a two-layer design: core functions with the `singulisp_` prefix and operation functions with the `__singulisp_str_` prefix.

`String/len` directly accesses a field (`SingulispString.len`) and does not make a C function call.

**Core functions**:

| Symbol | Description |
|---------|------|
| `singulisp_string_init_from_utf8` | Initializes a SingulispString from a UTF-8 byte sequence |
| `singulisp_string_clone` | Clones a string |
| `singulisp_string_drop` | Releases string resources (a no-op when the STATIC flag is set) |
| `singulisp_string_eq` | Compares strings for equality (with hash short-circuiting) |
| `singulisp_string_cmp` | Compares strings lexicographically |

**Operation functions**:

| Symbol | Description |
|---------|------|
| `__singulisp_str_concat` | Concatenates two strings |
| `__singulisp_str_contains` | Tests whether a substring is present |
| `__singulisp_str_starts_with` | Tests a prefix |
| `__singulisp_str_ends_with` | Tests a suffix |
| `__singulisp_str_replace` | Replaces text in a string |
| `__singulisp_str_split` | Splits a string |
| `__singulisp_str_trim` | Removes leading and trailing whitespace |
| `__singulisp_str_to_upper` | Converts a string to uppercase |
| `__singulisp_str_to_lower` | Converts a string to lowercase |
| `__singulisp_str_index_of` | Searches for a substring (byte offset or -1) |
| `__singulisp_str_substring` | Extracts a substring |
| `__singulisp_str_parse_i64` | Parses a string as an i64 |
| `__singulisp_str_parse_f64` | Parses a string as an f64 |

### Collections

Vec operations are generated inline during code generation, so they have no C runtime symbols.

### Persistent Data Structures

| Symbol | Description |
|---------|------|
| `__singulisp_pvec_new` | Creates an empty PVec |
| `__singulisp_pvec_push` | Appends an element (returns a new PVec) |
| `__singulisp_pvec_get` | Gets an element by index |
| `__singulisp_pvec_set` | Updates the value at an index (returns a new PVec) |
| `__singulisp_pvec_len` | Returns the length of a PVec |
| `__singulisp_pvec_retain` | Increments the reference count of a PVec |
| `__singulisp_pvec_release` | Decrements the reference count of a PVec (frees it at zero) |
| `__singulisp_pmap_new` | Creates an empty PMap |
| `__singulisp_pmap_assoc` | Inserts a key and value (returns a new PMap) |
| `__singulisp_pmap_get` | Gets a value by key |
| `__singulisp_pmap_contains` | Tests whether a key exists |
| `__singulisp_pmap_len` | Returns the number of entries |
| `__singulisp_pmap_retain` | Increments the reference count of a PMap |
| `__singulisp_pmap_release` | Decrements the reference count of a PMap (frees it at zero) |

### Input and Output

`println!` writes to standard output through the platform bundle's `__singulisp_platform_printf`.

| Symbol | Description |
|---------|------|
| `__singulisp_platform_printf` | Writes formatted output to standard output through the platform bridge |
| `__singulisp_fs_read_to_string` | Reads an entire file into a string |
| `__singulisp_fs_write_string` | Writes a string to a file |
| `__singulisp_fs_exists` | Tests whether a file exists |
| `__singulisp_fs_delete` | Deletes a file |
| `__singulisp_io_read_line` | Reads one line from standard input |
| `__singulisp_time_now_ms` | Returns the current time in milliseconds since the epoch (i64) |
| `__singulisp_process_exit` | Exits the process with the specified code |

### Concurrency

| Symbol | Description |
|---------|------|
| `__singulisp_scope_new` | Creates a scope |
| `__singulisp_scope_spawn` | Spawns a task within a scope |
| `__singulisp_scope_wait` | Waits for all tasks in a scope |
| `__singulisp_scope_free` | Frees a scope |

### Channels

| Symbol | Description |
|---------|------|
| `__singulisp_channel_new` | Creates a channel (MPMC ring buffer) |
| `__singulisp_channel_send` | Sends a value through a channel |
| `__singulisp_channel_recv` | Receives a value from a channel |
| `__singulisp_channel_close` | Closes a channel |
| `__singulisp_channel_free` | Frees a channel |

### GPU

| Symbol | Description |
|---------|------|
| `__singulisp_gpu_buf_from_slice` | Creates a GPU buffer from a slice |
| `__singulisp_gpu_buf_read` | Reads a value from a buffer |
| `__singulisp_gpu_buf_write` | Writes a value to a buffer |
| `__singulisp_gpu_buf_len` | Returns the length of a buffer |
| `__singulisp_gpu_buf_to_vec_data` | Obtains buffer data for use by a Vec |
| `__singulisp_gpu_buf_free` | Frees a buffer |

### Runtime Traps

There is a single unified trap function, `__singulisp_trap(int32_t code)`. No individual trap functions are defined.

| Trap code | Description |
|--------------|------|
| 1001 | Division by zero |
| 1002 | Signed integer MIN / -1 overflow |
| 3001 | Out-of-bounds array access |
| 4001 | Receive from a closed, empty channel |
| 4002 | Send to a closed channel |
| 5001 | Region capacity exceeded (OOM) |

## Type Layouts

### Primitive Types

| Type | Size (bytes) | Alignment (bytes) |
|----|---------------|---------------------|
| `bool` | 1 | 1 |
| `i32` / `u32` / `f32` | 4 | 4 |
| `i64` / `u64` / `f64` | 8 | 8 |

### Compound Types

| Type | Layout | Size (bytes) | Alignment (bytes) |
|----|-----------|---------------|---------------------|
| `String` | `{ ptr data, i32 len, i32 cap, i64 hash, i32 flags, i32 reserved }` | 32 | 8 |
| `Vec<T>` | `{ ptr data, i32 len, i32 cap }` | 16 | 8 |
| `Option<T>` | tag + T | 4 + size(T) + padding | max(align(T), 4) |
| `Result<T, E>` | tag + max(T, E) | 4 + max(size(T), size(E)) + padding | max(align(T), align(E), 4) |

`String` is a 32-byte fat struct. Strings of **15 bytes or fewer**, not 16 bytes or fewer, use SSO and use the trailing 16 bytes (the `hash` / `flags` / `reserved` area) as inline storage. For non-SSO strings, `hash` is a lazy xxHash64 (0 means not computed; a computed result of 0 is stored as 1), and bit 0 of `flags` is STATIC (0x01), which prevents long string literals from being freed.

The tag field of an ordinary `Option` or `Result` is an `i32` (4 bytes). The Nullable Pointer Optimization applies, however, to a two-variant enum containing only an empty variant and a payload guaranteed to be non-null. Thus, types such as `[Option [& T]]`, `[Option [& mut T]]`, and `[Option [Box T]]` have no tag and represent the empty variant with null.

### SIMD Types

| Type | Size (bytes) | Alignment (bytes) |
|----|---------------|---------------------|
| `f32x2` | 8 | 8 |
| `f32x4` | 16 | 16 |
| `f64x2` | 16 | 16 |
| `i32x2` | 8 | 8 |
| `i32x4` | 16 | 16 |
| `u32x4` | 16 | 16 |

### Math Types

Math types are dedicated built-in types, not types expanded into user-defined structs. The physical representations below determine their ABI classification. All have Copy semantics, but this does not mean that they are always held in SIMD registers.

| Type | ABI representation | Components and storage order | Size (bytes) | Alignment (bytes) |
|------|--------------------|------------------------------|--------------|-------------------|
| `Vec2` | `<2 x f32>` | `x`, `y` | 8 | 8 |
| `Vec3` | `<4 x f32>` | `x`, `y`, `z` (fourth lane unused) | 16 | 16 |
| `Vec4` | `<4 x f32>` | `x`, `y`, `z`, `w` | 16 | 16 |
| `Quat` | `<4 x f32>` | `x`, `y`, `z`, `w` | 16 | 16 |
| `Float3` | Three consecutive `f32` values with no padding | `x`, `y`, `z` | 12 | 4 |
| `Mat3` | `[3 x <4 x f32>]` | Three column vectors; the fourth lane of each vector is unused | 48 | 16 |
| `Mat3r` | `[3 x <4 x f32>]` | Three row vectors; the fourth lane of each vector is unused | 48 | 16 |
| `Mat4` | `[4 x <4 x f32>]` | Four column vectors | 64 | 16 |
| `Mat4r` | `[4 x <4 x f32>]` | Four row vectors | 64 | 16 |
| `Mat3x4` | `[3 x <4 x f32>]` | Three affine rows; each row's fourth component is translation | 48 | 16 |

The fourth lane of `Vec3` is not a language-level component, and its value is unspecified. The array stride of `Vec3` is 16 bytes, while that of `Float3` is 12 bytes; they cannot substitute for each other at an ABI boundary. Column-major and row-major types are distinct types with different storage order and operation semantics, even when the physical shape of their vector arrays is the same.

This table defines native CPU memory layout, not the BBin wire format or the std430 layout of `:repr gpu`. BBin encodes only logical components, so `Vec3` occupies 12 bytes and `Mat3` and `Mat3r` occupy 36 bytes in BBin. See [Serialization](../language/serialization.md) and [GPU](../performance/gpu.md) for details.

A low-level SIMD type remains distinct from a math type of the same physical vector width. A matching physical representation does not imply implicit conversion, operation compatibility, or C FFI compatibility.

### Struct Layouts

| repr | Rule |
|------|--------|
| Default | Field declaration order (ordinary ABI alignment and padding) |
| `:repr C` | C-compatible layout (declaration order and standard padding) |
| `:repr simd` | Requires 16-byte alignment at the corresponding allocation site. It does not change the struct's ABI alignment, tail padding, or array stride |
| `:repr gpu` | std430 layout (GPU-buffer compatible) |
| `:repr packed` | Removes padding while preserving declaration order and handles unaligned field access |
| `:repr ordered` | Preserves field order (no compiler reordering) |

## BBin v1 Serialization

The output of `T/serialize` consists, in order, of the `SGSV` magic, format version, schema ID, header size, type descriptor, and body. `T/bbin-write` appends only the headerless body to an existing `[Vec u8]` and is used only between identical schemas. See [Serialization](../language/serialization.md#binary-format-bbin-v1) for the complete wire format and per-type coverage.

## Calling Conventions

Singulisp follows the target platform's standard calling convention.

### Linux / macOS

Singulisp uses the System V AMD64 ABI.

- Integer/pointer arguments: `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` (up to 6)
- Floating-point arguments: `xmm0` through `xmm7` (up to 8)
- Return values: `rax` (integer), `xmm0` (floating point)
- Stack alignment: 16 bytes

### Windows

Singulisp uses the Microsoft x64 ABI.

- Integer/pointer arguments: `rcx`, `rdx`, `r8`, `r9` (up to 4)
- Floating-point arguments: `xmm0` through `xmm3` (up to 4)
- Return values: `rax` (integer), `xmm0` (floating point)
- Shadow space: 32 bytes

### Passing Structs

The passing method depends on the size of the struct.

| Condition | Method |
|------|------|
| Size <= 16 bytes; all members are integers or pointers | Split across registers |
| Size <= 16 bytes; contains floating-point members | Combination of SSE registers |
| Size > 16 bytes | Caller copies it to the stack and passes a pointer |

### Passing Math Types

Math-type parameters and return values follow the target ABI's vector or aggregate classification for the physical representations above. `Vec2`, `Vec3`, `Vec4`, and `Quat` are classified as vector values, `Float3` as a 12-byte aggregate, and matrix types as 48-byte or 64-byte aggregates. Copy semantics describes language-level passing by value; it does not guarantee register passing. The target ABI determines the exact registers, stack layout, indirect passing, and return-value locations.

The physical representations above are not, by themselves, declarations of C compatibility. At a C boundary, use a C-compatible scalar or reference signature, or a `:repr C` struct with fields matching the C side, and explicitly convert to and from math types inside the boundary.

### MIR ABI Optimizations

Phase G of the MIR optimization pipeline applies the following ABI optimizations.

- **param_optimize**: optimizes passing small structs in registers
- **return_optimize**: optimizes returning values in registers and coordinates with NRVO
