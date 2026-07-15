# CLI Reference

The compiler front-end executable is `gu-cli`.

```text
gu-cli <command> [options]
```

Run `gu-cli <command> --help` to inspect the options for each command.

## Root Commands

There are 22 root commands.

| Command | Purpose |
|---------|---------|
| `build` | Build a native binary |
| `run` | Build and run |
| `repl` | Run an interactive session |
| `asm` | Produce function-unit LLVM IR, assembly, IR/MIR, or GPU output |
| `fmt` | Format source code |
| `lsp` | Run the language server |
| `dump` | Produce a structured MIR dump |
| `analyze` | Produce a static-analysis report |
| `explain` | Explain a diagnostic code |
| `test` | Run `deftest` cases in isolation |
| `bench` | Build and run `defbench` cases |
| `bench-build` | Build a benchmark runner |
| `bench-run` | Run a previously built runner |
| `build-stdlib` | Build the target-specific standard-library cache |
| `emit-object` | Emit an `.o` file without linking |
| `bindgen` | Generate FFI bindings from a C header |
| `mod-sdk` | Generate a C SDK header from `definterface` |
| `doc` | Generate Markdown from `;;;` and `#[pub]` |
| `inject` | Inject a function patch into a hot process |
| `hot` | Query a hot process's function list or status |
| `pkg` | Manage packages |
| `cache` | Manage compiler caches |

## Build

```bash
gu-cli build src/main.gu
gu-cli build src/main.gu --release -o build/app
gu-cli build src/main.gu --target linux-x64
gu-cli build . --release
```

Primary options:

| Option | Meaning |
|--------|---------|
| `-o, --output PATH` | Output path |
| `--release` | MIR O2 plus aggressive LLVM optimization |
| `--debug` | No optimization plus DWARF |
| `--no-cache` | Do not use compiler caches |
| `--target PLATFORM` | Select a platform bundle |
| `--target-cpu CPU` | Select the CPU tuning target |
| `--target-features FEATURES` | Select CPU features |
| `--report-opt[=summary\|verbose]` | Optimization report |
| `--cfg KEY=VALUE` | Build configuration; may be specified multiple times |
| `--overflow-trap` | Trap on integer addition, subtraction, and multiplication overflow in debug builds |
| `--link PATH` | Link an additional `.so`, `.a`, or `.o`; may be specified multiple times |
| `--hot` | Embed a debug hot-reload IPC server |
| `--format text\|json` | Build-report format |
| `--pgo` | Automatically perform instrumentation, training, and optimization |
| `--pgo-generate FILE` / `--pgo-use FILE` | Manual PGO flow |

With `--pgo`, you may specify `--pgo-runs`, `--keep-pgo-artifacts`, and training arguments after `--`. `--pgo` cannot be combined with manual PGO, and `--pgo-generate` cannot be combined with `--pgo-use`.

The input to `build` may be a single `.gu` file or a project directory. Build options—including `--hot`, `--link`, `--no-cache`, and `--cfg`—have the same meaning for either input form.

`--link` is supported for both file and project builds, but cannot be combined with `--debug`, `--hot`, or automated `--pgo`.

## Run

```bash
gu-cli run examples/getting-started/hello.gu
gu-cli run src/main.gu --release
gu-cli run src/main.gu --link path/to/libcodec.so
```

`run` builds for the host-native platform and then executes the result. Use `build` for a cross target. Primary options are `--release`, `--debug`, `--target`, `--report-opt`, `--cfg`, `--hot`, `--no-cache`, `--link`, and manual PGO. The input may be either a file or a project directory.

## Tests and Benchmarks

```bash
gu-cli test tests/math.gu
gu-cli test tests/math.gu --filter vector --memory-limit 512
gu-cli test .

gu-cli bench benches/kernel.gu --timeout 30
gu-cli bench . --timeout 30
gu-cli bench-build benches/kernel.gu -o build/kernel.bench-runner
gu-cli bench-run build/kernel.bench-runner --timeout 30
```

`test` runs each `deftest` in an independent subprocess. `--memory-limit 0` means unlimited. `bench` and `bench-run` require `--timeout` to prevent hangs.

`test`, `bench`, `bench-build`, and `doc` all formally accept either a single `.gu` file or a project directory. See [Testing and Benchmarking](testing.md) for details.

## Function-Unit Inspection

### asm

```bash
gu-cli asm src/kernel.gu --function transform --format llvm-ir --release
gu-cli asm src/kernel.gu --function transform --format asm --release
gu-cli asm examples/performance/gpu-vector-scale.gu --function scale_values --format wgsl
```

Supported formats are `llvm-ir`, `asm`, `normalized`, `json`, `ir`, `mir`, `wgsl`, and `spirv`. Specify CPU tuning with `--target-cpu` and `--target-features`. `asm` has no platform-level `--target` option.

### dump

```bash
gu-cli dump src/kernel.gu --stage mir --function transform
gu-cli dump src/kernel.gu --list-passes
gu-cli dump src/kernel.gu --before dce
```

Only `mir` may be specified for `--stage`. Select pass boundaries with `--before` and `--after`.

### analyze

```bash
gu-cli analyze src/kernel.gu --function transform --kind vectorize --release
```

Choose `--kind` from `alias`, `effect`, `vectorize`, `escape`, `throughput`, `recurrence`, `speed`, `shape`, `profitability`, `substrate`, `divergence`, `bitloop`, `backend-summary`, and `stack-footprint`.

## Source Tools

```bash
gu-cli fmt src/main.gu --check
gu-cli fmt src/main.gu --stdout
gu-cli doc src/library.gu -o doc
gu-cli lsp
gu-cli repl
gu-cli explain E-TYPE-1006
```

`doc` collects `;;;` comments from public functions, structs, enums, traits, and constants. See [Formatter](formatter.md), [Documentation Generator](doc-generator.md), [LSP](lsp.md), and [REPL](repl.md) for individual descriptions.

## FFI Tools

```bash
gu-cli emit-object src/api.gu -o build/api.o --release
gu-cli bindgen library.h -o bindings.gu -I /usr/include/library
gu-cli mod-sdk src/interface.gu --interface Codec -o codec.h
```

`emit-object` retains `#[no-mangle]` functions as DCE roots. See [C FFI](../platform/ffi.md) for FFI type restrictions.

## Hot Reload

```bash
gu-cli run src/main.gu --hot
gu-cli inject --pid 1234 --fn transform --src src/main.gu
gu-cli hot list 1234
gu-cli hot status 1234 --format json
```

The `hot` subcommands are `list` and `status`. Both accept the PID as a positional argument and support `--format table|json`. See [Hot Reload](hot-reload.md) for details.

## Packages and Caches

The six `pkg` subcommands are `init`, `add`, `vendor`, `remove`, `update`, and `verify`. The three `cache` subcommands are `status`, `doctor`, and `clear`.

```bash
gu-cli pkg init --name data-pipeline
gu-cli pkg add fast-codec --path ../fast-codec
gu-cli pkg update
gu-cli pkg verify

gu-cli cache status
gu-cli cache doctor
gu-cli cache clear
```

See [Package Manager](package-manager.md) for the package format.
