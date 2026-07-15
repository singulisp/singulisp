# C FFI and External Modules

Singulisp provides direct C ABI declarations, binding generation from C headers, and dynamic module
ABIs through `definterface`. The only `extern` ABI is `"C"`.

## extern "C"

```lisp
(extern "C"
  (defn cos [(value : f64)] -> f64)
  (defn read-value [(address : u64)] -> i32))
```

Extern calls are treated as external side effects. For declarations with borrowed returns, lifetimes
cannot be proved unless the necessary contracts—such as `:readonly`, `:non-escape`, `:noalias`, and
`:returns-borrow-of`—are declared.

```lisp
(extern "C"
  (defn ffi-id [(input : [& i32])] -> [& i32]
    :readonly [input]
    :non-escape [input]
    :returns-borrow-of input))
```

## C type mapping

| C | Singulisp |
|---|---|
| `int8_t`, `uint8_t` | `i8`, `u8` |
| `int16_t`, `uint16_t` | `i16`, `u16` |
| `int32_t`, `uint32_t` | `i32`, `u32` |
| `int64_t`, `uint64_t` | `i64`, `u64` |
| `float`, `double` | `f32`, `f64` |
| `void` | `()` |
| pointer | `u64`, or a reference type with a provable ABI |
| C struct | A `:repr C` struct |

The surface language has no `*const u8` or `*mut T` type syntax. bindgen generates C pointers,
including struct pointers, as `u64`. A byte buffer is normally passed as an address of type `u64`
paired with an explicit length.

```lisp
(defstruct CBuffer :repr C
  (address : u64)
  (length : u64))
```

Inside Singulisp, `String` is a 32-byte owned object. `[& String]` is a reference to that object, while
by-value `String` and `StrSlice` extern parameters receive special treatment that lowers them to data
pointers. Neither automatically guarantees a general NUL-terminated `const char*` contract; state the
C API's termination, length, and ownership explicitly.

## C layout

```lisp
(defstruct CPoint :repr C
  (x : f32)
  (y : f32))
```

Aggregates exposed across an FFI boundary must use `:repr C`, with field order, widths, and alignment
matching the C declaration. `#[no-mangle]` preserves the symbol name and makes it a DCE root; it does
not automatically convert an arbitrary Singulisp aggregate to the C ABI.

```lisp
#[pub]
#[no-mangle]
(defn transform_i32 [(value : i32)] -> i32
  (* value 2))
```

Exports called from C must be limited to C-compatible scalar or reference signatures. `[Fn ...]` is a
Singulisp closure representation, not a C function pointer.

## bindgen

```bash
gu-cli bindgen library.h -o bindings.gu -I /usr/include/library
```

bindgen loads the libclang C API dynamically and generates C ABI bindings from functions, callbacks and
function pointers, structs, unions, bitfields, enums, variadic functions, and macro constants.

- Pointers are converted to `u64`.
- Use `--allow-unsupported` only when generation can continue by representing an unsupported type as a
  warning comment or a `u64` fallback.

With LLVM 20, specify the location of the libclang C API library when necessary, for example
`LIBCLANG_PATH=/usr/lib/llvm-20/lib`. `libclang-cpp` alone is insufficient.

## Linking and object output

For file or project builds and runs, repeat `--link` to pass `.so`, `.a`, or `.o` files to the final link.

```bash
gu-cli build src/main.gu --link path/to/libcodec.so --release
gu-cli run src/main.gu --link path/to/libcodec.so
gu-cli build . --link path/to/libcodec.so --release
```

`--link` cannot be combined with `--debug`, `--hot`, or automatic `--pgo`.

When an external build system performs the link, emit an object file.

```bash
gu-cli emit-object src/api.gu -o build/api.o
```

## definterface and module/load

`definterface` declares a function table implemented by an external module.

```lisp
(definterface CodecInterface
  (decode [(address : u64) (length : u64)] -> i32))

(defregion @modules
  :allocator (arena :bump)
  :lifetime :scoped
  :default-capacity (KiB 64)
  :on-oom :trap)

#[external]
(defn use-module [(path : String)] -> ()
  (@modules handle
    (let module (module/load handle path CodecInterface host-services))
    (let status (CodecInterface/decode module (to-u64 0) (to-u64 0)))
    (println! "status={}" status)))
```

The first argument to `module/load` is an actual `RegionHandle` from an active region block, not a type
name such as `@global`. The returned `ModuleHandle` cannot escape that region or cross a thread boundary.

## C SDK header

```bash
gu-cli mod-sdk src/codec.gu --interface CodecInterface -o sdk/codec.h
```

The generated header contains the function table and the `singulisp_module_bind` prototype. Its
header-specific ABI macros are `MOD_ABI_VERSION` and `MOD_ABI_TYPE_ID`; the runtime header
`singulisp/module.h` defines `SINGULISP_MODULE_ABI_VERSION`.

## ABI boundaries

- The only `extern` ABI is `"C"`.
- bindgen handles callbacks and function pointers, unions, bitfields, variadic functions, and macro constants.
- There is no dedicated raw-pointer surface type; pointers are represented primarily as `u64`.
- `#[no-mangle]` is not an ABI converter.
- A module handle's lifetime is bound to the region in which it is registered.
