# Package Manager

Singulisp project configuration is written in `Singulisp.toml`. In platform bundle v1, the only platform-dependent hard gate declared in the manifest is `package.hard-needs`. Do not use CPU-specific settings or `target-profile`.

---

## Minimal Example

```toml
[package]
name = "data-pipeline"
version = "0.1.0"
edition = "2026"
type = "bin"
hard-needs = ["ext.time", "ext.io"]

[optimization]
report = "summary"

[dependencies]
math-lib = { path = "../math-lib" }
```

---

## [package]

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Package name |
| `version` | string | Version string |
| `edition` | string | Language edition; `pkg init` generates `"2026"` |
| `type` | `"bin"` / `"lib"` | Package type |
| `hard-needs` | string array | Capabilities that the entire package unconditionally requires |

### Meaning of hard-needs

`hard-needs` declares that the package cannot function without the listed capability. The build stops only when the declaration conflicts with `missing` in the selected platform bundle.

```toml
[package]
name = "asset-packer"
version = "0.1.0"
edition = "2026"
type = "bin"
hard-needs = ["ext.fs", "ext.io"]
```

The decision has the following form:

```text
if hard-needs ∩ target.missing != ∅:
    build error
```

### Difference from `#[require]`

- `package.hard-needs`: Build gate
- `#[require ext.*]`: Soft-dependency metadata

`#[require]` is used to visualize dependencies, but does not stop the build by default.

---

## [optimization]

The only manifest build settings are for optimization reports.

```toml
[optimization]
report = "summary"  # or "verbose"
```

- `summary`: Includes a pass summary and diagnostic entries
- `verbose`: Includes pass-specific actions in addition to `summary`

---

## [dependencies] / [dev-dependencies]

The resolver supports local `path` dependencies and resolves `version` requirements from a local registry or source named by `registry = "path-or-file-url"`. Network fetching is not part of compiler-resolver semantics; mirror or vendor dependencies into a local registry or source before resolution. `features` are unified deterministically within the graph and included in the lockfile and package-summary checksum. Features do not implicitly expand optional dependencies: every dependency edge must be explicit in the manifest. `dev-dependencies` are inputs to test and benchmark build graphs and do not enter the normal build dependency graph. Only the root package's direct `dev-dependencies` are added to a test or benchmark graph. A dependency package's own `dev-dependencies` are resolved only in a test or benchmark graph where that package is the root.

```toml
[dependencies]
physics = { path = "../physics", version = "1.0", features = ["simd"] }
math = { version = ">=1.0, <2.0", registry = "../registry", features = ["fast"] }

[dev-dependencies]
test-utils = { path = "../test-utils" }
```

`gu-cli build` writes the resolved graph to `Singulisp.lock`. When a lockfile already exists, the compiler compares its root, compilation order, package metadata, source identity, resolved source checksum, feature set, and direct dependency names with the current manifest resolution. A mismatch is reported as `E-PKG-0007`. After the build, the lockfile records generated summary artifact keys, summary checksums, public-signature checksums, body-bearing template checksums, and dependency object artifact keys and checksums. On the next build, a leaf dependency's summary cache can be reused without full lowering when its source fingerprint, cache key, and checksum match.

`gu-cli pkg` has exactly **six** subcommands: `init`, `add`, `remove`, `vendor`, `update`, and `verify`. There are no `fetch`, `lock`, or `list` subcommands.

```bash
gu-cli pkg init [--name NAME] [--type bin|lib]   # Initialize a new package
gu-cli pkg add physics --path ../physics
gu-cli pkg add math --version ">=1.0, <2.0" --registry ../registry --feature fast
gu-cli pkg add test-utils --path ../test-utils --dev
gu-cli pkg remove test-utils --dev
gu-cli pkg update   # Regenerate Singulisp.lock from the current manifest
gu-cli pkg vendor   # Copy resolved dependencies into ./vendor and pin the manifest to path dependencies
gu-cli pkg verify   # Verify the manifest, lockfile, summary cache, and object-cache artifacts
```

`pkg add` can add a `--path` dependency or a local `--registry` plus `--version` dependency. `--feature <name>` may be specified more than once, and `--dev` writes the entry to `[dev-dependencies]`. `pkg remove --dev` removes an entry from `[dev-dependencies]`. Adding, removing, or vendoring regenerates `Singulisp.lock`.

`pkg vendor [dir]` copies every non-root dependency source that the current resolver can resolve into a local directory. The default destination is `vendor/`. It performs no network fetch and considers only sources that already resolve as a path or local registry. Direct dependencies in the root manifest are rewritten to vendored paths, while `version` and `features` are preserved. A dependency entry inside a vendored package is also rewritten to a relative path when its target belongs to the same vendor set.

`pkg verify` checks the package identity, summary source fingerprint, summary cache key, full summary checksum, public-signature checksum, target/cfg, template-cache checksum, and object-cache key and checksum recorded in the lockfile. A mismatch is reported as `E-PKG-0008`.

---

## Singulisp.lock (Lockfile)

This TOML file records the result of dependency resolution. It is generated automatically in the package root and should be committed to version control.

### Lockfile Format (Schema Version 1)

```toml
version = 1
root = "data-pipeline"
compile_order = ["std", "fast-codec", "data-pipeline"]

[[package]]
name = "std"
version = "0.1.0"
type = "lib"
edition = "2026"
source = "bundled+std:///path/to/std#abc123"
resolved_checksum = "..."

[[package]]
name = "fast-codec"
version = "1.2.0"
type = "lib"
edition = "2026"
source = "path+file:///absolute/path/to/fast-codec"
resolved_checksum = "..."
features = ["simd"]
dependencies = ["std"]

[[package]]
name = "data-pipeline"
version = "0.1.0"
type = "bin"
edition = "2026"
source = "path+file:///absolute/path/to/data-pipeline"
resolved_checksum = "..."
dependencies = ["fast-codec", "std"]

[[package.artifacts]]
target = "x86_64-unknown-linux-gnu"
cfg = ["release=true"]
source_fingerprint = "..."
summary_cache_key = "package-summary-v3-..."
summary_checksum = "..."
public_signature_checksum = "..."
template_checksum = "..."       # When a generic-body artifact exists
object_cache_key = "..."         # When a dependency object exists
object_checksum = "..."
```

`compile_order` is the topologically sorted compilation order. The root section contains the schema version, root package, and compilation order. Each `[[package]]` records its identity, source, checksum, features, and direct dependencies. After a build, `[[package.artifacts]]` entries add target- and cfg-specific summary keys and checksums for use by `pkg verify` and incremental builds.

---

## Module-Name Generation Rules

The module name of a `.gu` file under `src/` is generated from its **parent-directory path**. The file name is not part of the module name; it is used only as the source-shard name, an identifier for diagnostics and caching.

| File Path | Module Name | Source-Shard Name |
|-----------|-------------|-------------------|
| `src/main.gu` | `""` (root module) | `main` |
| `src/helpers.gu` | `""` (root module) | `helpers` |
| `src/math/add.gu` | `math` | `math::add` |
| `src/physics/collision/narrow.gu` | `physics::collision` | `physics::collision::narrow` |

File and directory names may contain only lowercase ASCII letters, digits, `_`, and `-`. A hyphen `-` is normalized to `_` in module names. Uppercase or non-ASCII characters produce error `E-PKG-0006`.

---

## Error Codes

| Error Code | Condition |
|------------|-----------|
| E-PKG-0001 | A cyclic dependency was detected |
| E-PKG-0003 | A directory named by `path` does not exist or contains no `Singulisp.toml` |
| E-PKG-0004 | A `version` requirement does not match, neither `path` nor `registry` was specified, `std` was specified explicitly, or a feature name is invalid |
| E-PKG-0006 | A module file or directory name contains a disallowed character |
| E-PKG-0007 | The contents of `Singulisp.lock` do not match the manifest's newly resolved result |
| E-PKG-0008 | An artifact recorded in `Singulisp.lock` is absent or its checksum does not match |

---

## Selecting a Platform

Select the target with the CLI option `--target <platform>`, not in the manifest.

```bash
gu-cli build . --target linux-x64 --release
gu-cli build . --target wasm32-wasip1 --release
```

The available `platform` values are defined by the platform bundles included with the implementation.
