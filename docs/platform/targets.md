# Target Platforms

Singulisp target selection is standardized on platform bundle v1. Users normally need to consider only two axes:

- `--target <platform>`
- `--debug` / `--release`

Per-CPU settings, target profiles, and provider switching are not exposed as user input. `platform` is
the name of a bundle shipped with the implementation; the Singulisp implementation is solely responsible
for the triple, runtime bridge, and cache separation.

The following bundles are included:

- `linux-x64`
- `linux-arm64`
- `macos-x64`
- `macos-arm64`
- `windows-x64` (GNU ABI)
- `wasm32-wasip1`

The Windows MSVC ABI, Windows arm64, Android, iOS, browser WebAssembly, and bare metal are not in this
list and are not supported targets.

---

## 1. Platform-bundle structure

Each target bundle consists of target metadata and a runtime-bridge implementation.

### Target metadata

```toml
llvm-triple = "wasm32-wasip1"
missing = ["ext.thread", "ext.sync", "ext.dl"]
```

- `llvm-triple`: the base triple for LLVM code generation and linking
- `missing`: capabilities absent from the platform

Singulisp defines the target's capabilities as the complete set minus `missing`, while also supplying
`os`, `abi`, `pointer-width`, and `llvm-triple` to the build configuration.

### Runtime bridge

The runtime bridge is the implementation entry point through which the standard library uses platform facilities.

- Provide ordinary implementations for available facilities.
- Retain linkable entry points for unsupported facilities, but do not make those facilities available.
- `debug`: trap
- `release`: UB

Because ordinary call sites do not perform capability checks, treatment of unsupported facilities is fixed in the runtime bridge.

Platform-specific runtime implementations are contained entirely within the bundle. The compiler and
linker treat only the bridges provided by the bundle as platform-specific input.

---

## 2. CLI usage

A typical build or run takes the following form:

```bash
gu-cli build app.gu --target linux-x64 --release
gu-cli run   app.gu --target linux-x64 --debug
```

### `build`

`build` succeeds when all of the following hold:

- The target platform bundle exists.
- The toolchain supports its `llvm-triple`.
- The package's `hard-needs` does not conflict with the bundle's `missing` list.

### `run` / `test` / `bench`

`build` can handle a cross target for a target triple. `run`, `test`, and `bench` are permitted only for
host-native families for which Singulisp has a built-in runner.

- Host-native family: executable
- Any other family: CLI error

Platform authors cannot customize runner behavior.

---

## 3. Relationship to capabilities

Singulisp capabilities fall into two categories:

- `package.hard-needs`: a build gate for the entire package
- `#[require ext.*]`: per-function soft-dependency metadata

### `hard-needs`

`Singulisp.toml`:

```toml
[package]
name = "my-app"
version = "0.1.0"
type = "bin"
hard-needs = ["ext.io", "ext.fs"]
```

Evaluation rule:

```text
if hard-needs ∩ target.missing != ∅:
    build error
```

### `#[require ext.*]`

`#[require]` declares a dependency and does not stop a build by default. It is retained as metadata in IR/MIR and tooling.

---

## 4. Cache separation

Different targets produce different artifacts even from the same source. Standard-library and build-cache keys include at least:

- platform name
- LLVM triple
- build mode (`debug` / `release`)
- the contents of `missing`

This prevents host and cross builds from being mixed silently.

---

## 5. Minimal procedure for adding a platform

Adding a platform is limited to the following steps:

1. Prepare a platform bundle.
2. Define the LLVM triple and unsupported capabilities in target metadata.
3. Implement the runtime bridge.
4. Confirm that `gu-cli build --target <platform>` succeeds.

Keep platform-specific knowledge inside this bundle.

---

## 6. Related documents

- capability: [capability.md](./capability.md)
- cross compilation: [cross-compilation.md](./cross-compilation.md)
- CLI: [../tooling/cli.md](../tooling/cli.md)
- package manifest: [../tooling/package-manager.md](../tooling/package-manager.md)
