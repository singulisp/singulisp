# Installation

This page explains how to build and install the Singulisp compiler.

## Prerequisites

Building Singulisp requires the following toolchain:

| Tool | Version | Notes |
|--------|-----------|------|
| Rust | 1.93 or later | Installation through `rustup` is recommended. |
| LLVM | 20.1.x | Must include development headers and libraries, including the **libclang C API**, which `gu-cli bindgen` uses to generate FFI bindings. |
| C compiler | gcc or clang | Used to build the runtime library; the `cc` crate detects it automatically. |

### Supported platforms

- **Ubuntu 24.04** (recommended)
- **macOS** (Apple Silicon / Intel)
- **Windows x64** (`x86_64-pc-windows-gnu`, MSYS2 / mingw-w64)

### Installing LLVM

On Ubuntu:

```bash
# Add the official LLVM 20 repository
wget https://apt.llvm.org/llvm.sh
chmod +x llvm.sh
sudo ./llvm.sh 20

# Development packages
#   - llvm-20-dev / libpolly-20-dev: code generation for the compiler itself
#   - libclang-20-dev / libclang1-20: the libclang C API used for FFI binding
#     generation by gu-cli bindgen. This is distinct from the libclang-cpp C++
#     API; without it, bindgen fails at runtime.
sudo apt install llvm-20-dev libpolly-20-dev libclang-20-dev libclang1-20
```

On macOS:

```bash
# llvm@20 includes the libclang C API
brew install llvm@20
```

Ensure that `llvm-config` is on `PATH`.

```bash
llvm-config --version
# => 20.1.x
```

## Build

Clone the repository and create a release build.

```bash
git clone https://github.com/singulisp/singulisp
cd singulisp
cargo build --release
```

When the build completes, the binaries are generated at:

```
target/release/gu-cli
target/release/singulisp-mcp
```

## Configure PATH

For a source build, add `target/release` within the checkout to `PATH`. The compiler also relies on the
locations of the runtime and `std`, so do not copy only the binaries elsewhere.

```bash
# Add this to ~/.bashrc or ~/.zshrc
export PATH="$HOME/path/to/singulisp/target/release:$PATH"
```

Restart your shell or reload its configuration file.

```bash
source ~/.bashrc
```

## Verify the installation

Start the REPL to confirm that the installation works.

```bash
gu-cli repl
```

If the prompt appears, the installation succeeded.

```
singulisp>
```

Enter `:quit` to leave the REPL.

```
singulisp> :quit
```

## Next steps

After installation, continue to the [Hello, World!](hello-world.md) tutorial.
