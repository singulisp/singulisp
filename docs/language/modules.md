# Modules and Packages

Singulisp uses folders as module identities and combines multiple files as source shards of the same module. File names themselves are not part of module paths.

## Path Mapping

`src/` is the root module.

| File | Module |
|---|---|
| `src/main.gu` | root |
| `src/helpers.gu` | root |
| `src/codec/read.gu` | `codec` |
| `src/codec/write.gu` | `codec` |
| `src/codec/json/value.gu` | `codec::json` |

Because `src/codec/read.gu` and `src/codec/write.gu` belong to the same module, each can refer to definitions in the other without `use`. Shards are combined in a deterministic order.

The entry point is `main.gu` in the root module. Test sources have the `tests` prefix. A build combines the modules reachable from the entry point through `use`.

## use

You may import an entire module, select named imports, or specify an alias.

```lisp
(use codec)
(use codec [Record decode])
(use codec::json :as json)
```

To expose an import outside the package unchanged, attach `#[pub]` and re-export it.

```lisp
#[pub]
(use codec [Record decode])
```

## Visibility

| Annotation | Visibility |
|---|---|
| `#[pub]` | Visible from dependent packages |
| None | Within the same package |
| `#[priv]` | Within the module corresponding to the same folder |

`#[pub]` and `#[priv]` cannot be specified together.

```lisp
#[pub]
(defstruct Record
  (id : i64)
  (score : f64))

(defn normalize-score [(value : f64)] -> f64
  (f64/clamp value 0.0 1.0))

#[priv]
(defn decode-tag [(raw : i32)] -> i32
  (bit-and raw 255))
```

## Package manifest

`Singulisp.toml` at the project root defines the package.

```toml
[package]
name = "data-pipeline"
version = "0.1.0"
edition = "2026"
type = "bin"

[optimization]

[dependencies]
fast-codec = { path = "../fast-codec" }

[dev-dependencies]
test-support = { path = "../test-support" }
```

The principal sections are `[package]`, `[optimization]`, `[dependencies]`, and `[dev-dependencies]`. Import the public surface of a dependency by placing the package name first.

```lisp
(use fast-codec [decode])
(use fast-codec::binary [decode-u64])
```

For adding, removing, vendoring, and verifying local dependencies, see [Package Management](../tooling/package-manager.md).

## Cyclic Dependencies

Module dependencies must be acyclic. If a cycle arises, move shared types and functions to an independent module upstream in the dependency direction. Note that splitting files alone does not create separate modules.

## External Modules

A dynamic library with an ABI declared by `definterface` can be loaded with `module/load`, which returns a `ModuleHandle`. A `ModuleHandle` cannot cross a thread boundary.

Pass the actual `RegionHandle` of an active region block as the first argument to `module/load`. The returned `ModuleHandle` cannot escape the region in which it is registered. See [FFI](../platform/ffi.md) for a complete example.

For `definterface`, SDK header generation, and C ABI constraints, see [FFI](../platform/ffi.md).
