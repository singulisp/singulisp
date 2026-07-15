# Cross-Compilation

Singulisp cross targets are defined by platform bundles shipped with the implementation, not by
`[target.*]` in `Singulisp.toml`. Users select only `--target <platform>`; Singulisp alone is responsible
for selecting the triple and runtime bundle.

---

## Basic usage

```bash
gu-cli build src/main.gu --target linux-x64 --release
gu-cli build data-pipeline/ --target wasm32-wasip1 --release
```

When `--target` is omitted, the host-native platform bundle is selected automatically.

---

## Platform-bundle structure

A platform bundle combines target metadata with the runtime-bridge implementation required by the platform.

### Target metadata

```toml
llvm-triple = "wasm32-wasip1"
missing = ["ext.thread", "ext.sync", "ext.dl"]
```

- `llvm-triple`: the basis for LLVM code generation and linking
- `missing`: capabilities absent from the platform

Singulisp constructs target information from this metadata and uses it to evaluate `cfg-case (os ...)`
and `package.hard-needs`.

---

## Relationship to hard-needs

A project build compares `package.hard-needs` in `Singulisp.toml` with `missing` in the target bundle.

```toml
[package]
name = "data-pipeline"
version = "0.1.0"
edition = "2026"
type = "bin"
hard-needs = ["ext.io", "ext.fs"]
```

If a capability in `hard-needs` also appears in the target's `missing` list, the build fails.

---

## Treatment of `#[require]`

`#[require ext.*]` is soft-dependency metadata and is not a build gate by default. Only
`package.hard-needs` stops a build.

This design separates per-function dependency declarations from the viability requirements of the entire package.

---

## run / test / bench

`build` can handle a cross target for a target triple. `run`, `test`, and `bench` are permitted only for
host-native families for which Singulisp has a built-in runner.

- `build`: cross targets allowed
- `run`: host-native only
- `test`: host-native only
- `bench`: host-native only

For a cross target, therefore, successful `gu-cli build --target <platform>` is the initial success criterion.
