# Singulisp on Windows

Singulisp can be built and run natively on Windows x64. The supported configuration is the MSYS2
mingw-w64 toolchain (`x86_64-pc-windows-gnu` ABI) with LLVM 20.1. Generated objects are linked with
mingw `gcc` or `clang`.

The MSVC target is not supported. MSYS2 is the standard Windows build environment because it provides
the complete set of required LLVM 20.1 static libraries and development tools.

## 1. Install prerequisites

### Rust (GNU host, 1.90 or later)

```powershell
rustup toolchain install 1.93.0-x86_64-pc-windows-gnu
```

Building the compiler requires Rust 1.90 or later.

### MSYS2 and the LLVM 20.1 toolchain

```powershell
winget install --id MSYS2.MSYS2 -e
```

Install the basic toolchain in the MSYS2 bash shell.

```bash
pacman -Sy
pacman -S --needed mingw-w64-x86_64-gcc mingw-w64-x86_64-lld
```

LLVM 20.x is required. If the MSYS2 rolling release provides version 21 or later, install exactly
20.1.8 from the archive.

```bash
B=https://repo.msys2.org/mingw/mingw64 ; V=20.1.8-2
pacman -U --noconfirm \
  $B/mingw-w64-x86_64-llvm-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-llvm-libs-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-llvm-tools-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-clang-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-clang-libs-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-lld-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-compiler-rt-$V-any.pkg.tar.zst
```

The LLVM 20.1.8 distribution uses the GCC 15 C++ ABI, so use GCC 15 as well. Mixing it with GCC 16 or
later can produce `0xC0000139 (ENTRYPOINT_NOT_FOUND)` when LLVM tools start.

```bash
B=https://repo.msys2.org/mingw/mingw64 ; V=15.2.0-14
pacman -U --noconfirm \
  $B/mingw-w64-x86_64-gcc-$V-any.pkg.tar.zst \
  $B/mingw-w64-x86_64-gcc-libs-$V-any.pkg.tar.zst
```

Verify the versions.

```bash
/mingw64/bin/llvm-config --version   # 20.1.8
/mingw64/bin/gcc --version           # 15.2.0
```

## 2. Build

In PowerShell, put the MSYS2 bin directory first on `PATH` and specify the LLVM location.

```powershell
$env:Path = "C:\msys64\mingw64\bin;$env:Path"
$env:LLVM_SYS_201_PREFIX = "C:\msys64\mingw64"

cargo +1.93.0-x86_64-pc-windows-gnu build --release --workspace
.\target\release\gu-cli.exe build-stdlib
```

The build uses MSYS2 mingw from `PATH` as its linker. The main-thread stack is set to 256 MB to support deep recursion.

## 3. Run

`gu-cli.exe` requires mingw runtime DLLs such as `libstdc++-6.dll`. When using a CLI built from source,
keep the MSYS2 bin directory on `PATH` at runtime as well.

```powershell
$env:Path = "C:\msys64\mingw64\bin;$env:Path"
.\target\release\gu-cli.exe run .\hello.gu
.\target\release\gu-cli.exe build .\hello.gu -o .\build\hello.exe
```

On Windows, `.exe` is added automatically even when the output name omits an extension.

## 4. Windows behavior

- Executables produced by build, run, and test have the `.exe` suffix.
- The runtime is located relative to `SINGULISP_RUNTIME_DIR` or the CLI installation directory.
- If the installation directory is not writable, caches are stored in `%LOCALAPPDATA%\Singulisp`.
- REPL history is stored under the Windows user profile.
- String case conversion and numeric parsing do not depend on the Windows display language or locale.
- The GPU backend can use the Windows Vulkan loader.

## 5. MSI installer

Running the distributed `singulisp-<version>-x64.msi` installs the CLI, MCP server, required mingw
runtime DLLs, and platform data. The installer adds the CLI bin directory to `PATH` and configures the
runtime search location.

After installation, open a new PowerShell window and verify the CLI.

```powershell
gu-cli.exe --version
```

## 6. Known limitations

- **LTO builds**: LTO requires `clang` capable of processing LLVM bitcode. If clang does not work in
  the MSYS2 environment with pinned LLVM, LTO is unavailable; ordinary release builds remain available.
- **Concurrent regeneration of the stdlib cache**: Multiple `gu-cli` processes can conflict when
  regenerating the same stdlib cache simultaneously. Regenerate it from one process at a time.
- **Test stack**: To run compiler tests with deep recursion, set `RUST_MIN_STACK=268435456` and run the tests serially.

## 7. Locale-independent string processing

Singulisp string APIs behave as follows regardless of the operating-system display language or C locale:

- ASCII case conversion and whitespace tests transform only the ASCII range and leave non-ASCII bytes unchanged.
- Substring searches such as `String/index-of` search UTF-8 byte sequences.
- `String/parse-f64` interprets `.` as the decimal separator regardless of locale.
- `String/parse-i64` is locale-independent.

String indices and offsets to `String/substring` are byte positions, not Unicode code-point positions.
Case conversion and collation for non-ASCII characters are out of scope.

## 8. GPU (GPGPU) backend

The backend that executes `gpu/dispatch` as Vulkan compute is available on Windows. It requires
`vulkan-1.dll`, provided by a GPU driver or the Vulkan Runtime/SDK. The Windows Vulkan loader selects a
registered GPU driver, so manual ICD selection is normally unnecessary.

```powershell
$env:Path = "C:\msys64\mingw64\bin;$env:Path"
$env:LLVM_SYS_201_PREFIX = "C:\msys64\mingw64"

cargo +1.93.0-x86_64-pc-windows-gnu build --release --workspace

$env:SINGULISP_GPU_BACKEND = "vulkan"
.\target\release\gu-cli.exe run .\gpu_program.gu
```

If `SINGULISP_GPU_BACKEND=vulkan` is not set, Singulisp uses the CPU reference backend. If the GPU runtime
library is outside the standard search paths, set `SINGULISP_GPU_RUNTIME_LIB` to its location. The Vulkan
backend cannot run without a Vulkan loader and an available device.
