# Project Structure

This page explains Singulisp package manifests, folder-based modules, and local dependencies.

## Create a project

`pkg init` initializes the current directory.

```bash
mkdir data-pipeline
cd data-pipeline
gu-cli pkg init --name data-pipeline
```

```text
data-pipeline/
├── Singulisp.toml
└── src/
    └── main.gu
```

To create a library, specify `--type lib`; this generates `src/lib.gu`.

## Singulisp.toml

A minimal executable package is:

```toml
[package]
name = "data-pipeline"
version = "0.1.0"
edition = "2026"
type = "bin"
```

The primary sections are `[package]`, `[optimization]`, `[dependencies]`, and `[dev-dependencies]`.
Set `type` to either `"bin"` or `"lib"`.

## Folders become modules

A module's identity is determined by its containing folder, not its filename. `.gu` files in the same
folder are combined as source shards of the same module.

| File | Module |
|---|---|
| `src/main.gu` | root |
| `src/helpers.gu` | root |
| `src/codec/read.gu` | `codec` |
| `src/codec/write.gu` | `codec` |
| `src/codec/json/value.gu` | `codec::json` |

For example, both `read.gu` and `write.gu` below belong to the `codec` module.

```text
src/
├── main.gu
├── helpers.gu
└── codec/
    ├── read.gu
    ├── write.gu
    └── json/
        └── value.gu
```

The entry point is `main.gu` in the root module. Modules reachable from the entry point through `use`
are included in compilation.

## Use modules

`use` can bring an entire module, selected definitions, or an alias into scope.

```lisp
(use codec)
(use codec [decode encode])
(use codec::json :as json)
```

```lisp
;; src/codec/read.gu
#[pub]
(defstruct Record
  (id : i64)
  (score : f64))

#[pub]
(defn decode [(id : i64) (score : f64)] -> Record
  (Record id score))
```

Built-in mathematical types such as `Vec2`, `Vec3`, and `Mat4` are reserved and cannot be used as
names for user-defined types.

## Visibility

| Declaration | Visibility |
|---|---|
| `#[pub]` | Public outside the package |
| None | Within the same package |
| `#[priv]` | Within the module in the same folder |

A `use` annotated with `#[pub]` is a re-export.

```lisp
#[pub]
(use codec [Record decode])
```

## Local dependencies

The package manager supports local path dependencies and local registries or `file:` URLs. It does not
fetch from network registries.

```bash
gu-cli pkg add fast-codec --path ../fast-codec
gu-cli pkg remove fast-codec
```

The manifest represents these dependencies as follows:

```toml
[dependencies]
fast-codec = { path = "../fast-codec" }

[dev-dependencies]
test-support = { path = "../test-support" }
```

Refer to a dependency's public modules with the package name as the leading component.

```lisp
(use fast-codec [decode])
(use fast-codec::binary [decode-u64])
```

See [Package Management](../tooling/package-manager.md) for all `pkg` operations and
[Modules](../language/modules.md) for module details.
