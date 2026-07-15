# GPU Compute

Singulisp provides GPU compute facilities that use the same S-expression syntax for shader functions
and emit WGSL or SPIR-V. Large-scale data-parallel processing integrates with CPU-side types, ownership,
and layout rules.

## `:repr gpu` — layout for GPU transfers

A struct marked `:repr gpu` follows std430 layout and can be used for CPU-to-GPU data transfer.

```lisp
(defstruct Particle :repr gpu
  (pos : Vec3) (vel : Vec3) (life : f32))
```

Under std430 layout, field alignment and ordering are adjusted automatically to meet GPU requirements.

## `#[shader]` — shader functions

A function annotated with `#[shader]` is compiled as a compute shader. `:thread-group-size` specifies
the thread-group dimensions and defaults to `[64 1 1]` when omitted.

| Form | Description |
|------|------|
| `#[shader]` | Use the default thread-group size `[64 1 1]` |
| `#[shader :thread-group-size [X Y Z]]` | Use an X × Y × Z three-dimensional thread group |

```lisp
#[shader :thread-group-size [64 1 1]]
(defn my-shader
  [(data : [& mut [GpuBuffer f32]])] -> ()
  (let tid (gpu/thread-id-x))
  (when (< tid (gpu-buf/len data))
    (gpu-buf/write data tid (* (gpu-buf/read data tid) 2.0f))))
```

A shader function's only permitted return type is `()`.

## GPU subset constraints

The following operations are prohibited in the body of a `#[shader]` function. Violations are compile-time errors.

| Error code | Prohibited operation |
|-------------|---------|
| `E-GPU-0101` | A `:repr gpu` struct contains a GPU-incompatible field |
| `E-GPU-0102` | Use of a GPU-incompatible type such as `f64` or `String` |
| `E-GPU-0103` | Heap allocation such as `Box/new` or `Vec/new` |
| `E-GPU-0104` | String operations |
| `E-GPU-0105` | I/O operations such as `println!` |
| `E-GPU-0106` | Direct recursion (self-recursion) |
| `E-GPU-0107` | Indirect calls, function pointers, closures, or dynamic trait dispatch |
| `E-GPU-0108` | Closure captures |
| `E-GPU-0109` | Reference operations other than permitted GPU arguments and atomic targets |
| `E-GPU-0110` | Region or arena operations |
| `E-GPU-0111` | Concurrency operations such as `with-scope` or `scope/spawn` |
| `E-GPU-0112` | Access to the borrow source while a wrapped buffer is borrowed or before dispatch completes |
| `E-GPU-0113` | Transitive recursion, such as an a→b→a call cycle |

These constraints are checked statically at compile time. Shader parameters of type
`[& [GpuBuffer T]]` or `[& mut [GpuBuffer T]]`, and the `[& mut i32]` target of `gpu/atomic-add`,
are intentional exceptions to E-GPU-0109.

## GpuBuffer operations

`GpuBuffer<T>` represents an array in GPU memory.

```lisp
;; Read
(gpu-buf/read buf index)

;; Write
(gpu-buf/write buf index value)

;; Get the length
(gpu-buf/len buf)
```

## GPU built-in functions

These built-in functions are available inside shader functions.

| Function | Type | Description |
|---|---|---|
| `(gpu/thread-id-x)` | `u32` | Global thread ID in the X dimension |
| `(gpu/thread-id-y)` | `u32` | Global thread ID in the Y dimension |
| `(gpu/thread-id-z)` | `u32` | Global thread ID in the Z dimension |
| `(gpu/group-id-x)` | `u32` | Workgroup ID in the X dimension |
| `(gpu/group-id-y)` | `u32` | Workgroup ID in the Y dimension |
| `(gpu/group-id-z)` | `u32` | Workgroup ID in the Z dimension |
| `(gpu/group-barrier)` | `()` | Synchronization barrier for threads within the group |
| `(gpu/shared (let mut name : T init))` | — | Allocate workgroup-shared memory |
| `(gpu/atomic-add (& mut target) delta)` | `i32` | Atomic addition on `i32`, returning the value before addition |

### Declaring shared storage

```lisp
#[shader :thread-group-size [64 1 1]]
(defn shared-shape [] -> ()
  ;; gpu/shared wraps a let expression
  (gpu/shared (let mut shared-values : [Array i32 64]
                    (Array/fill 64 0))))
```

`gpu/shared` can declare scalars and indexable fixed-length `Array` values. The workgroup-memory surface provides
atomic operations on shared storage. Ordinary unsynchronized writes by multiple invocations to the same
location constitute a data race.

Shared storage is shared within each workgroup. `gpu/group-barrier` synchronizes a workgroup and makes
shared writes before the barrier visible to invocations in the same workgroup. The CPU reference backend
also preserves the workgroup semantics of shared storage, barriers, and atomics.

## Dispatch from the CPU

Launch a shader and transfer data from the CPU side.

### Synchronous dispatch

```lisp
;; Create a GpuBuffer from a Vec (two arguments: region and slice reference)
(let gpu-data (GpuBuffer/from-slice @gpu (& my-vec)))

;; Dispatch the shader (shader name + :threads [x y z] + :args [...])
(gpu/dispatch my-shader :threads [1024 1 1] :args [(& mut gpu-data)])

;; Convert the result back to a Vec
(let result (GpuBuffer/to-vec gpu-data))
```

Every element of `:threads` must be a **32-bit integer (`i32` or `u32`) expression**. Other types such as
`f32` and `i64` are rejected with stable frontend type diagnostics.

### Asynchronous dispatch

```lisp
;; Asynchronous dispatch—returns a u64 task handle
(let task (gpu/dispatch-async my-shader :threads [1024 1 1] :args [(& mut gpu-data)]))

;; Wait for the task to complete
(await task)
```

`dispatch-async` starts a worker on a separate OS thread through the platform thread API and returns a
`u64` task handle. It uses pthread on POSIX and CreateThread on Windows, joining with `await`. It is
unavailable on WASI targets without `ext.thread`. The Vulkan backend performs Vulkan dispatch from the worker thread.

### Creating a GpuBuffer

```lisp
;; Create by copying from a reference to a Vec (data is duplicated in the buffer)
(GpuBuffer/from-slice @region (& values))  ;; values : [Vec T]
                                           ;; -> [GpuBuffer T]

;; A buffer that borrows a Vec
(GpuBuffer/wrap (& mut values))            ;; -> [GpuBuffer T]
```

The second argument to `from-slice` is not a general slice; it is exactly `[& [Vec T]]` or
`[& mut [Vec T]]`. `T` must be both Copy and GPU POD: `i32`, `u32`, or `f32`; a two- to four-lane SIMD
type over one of them; a mathematical type other than `Float3`; a fixed-length `Array` of a supported
type; or a `:repr gpu` struct whose fields all satisfy these conditions. `bool` may be used as a scalar
shader argument or condition, but not as a `GpuBuffer` element.

The lifetime of a buffer returned by `GpuBuffer/wrap` is tied to the borrowed `Vec`. The original borrow
is retained while the buffer is live and, for asynchronous dispatch, until `await` completes.

### Passing scalar arguments

On the Vulkan backend, scalar shader arguments such as `i32`, `u32`, `f32`, and `bool` are packed
automatically into a uniform buffer at `@group(0) @binding(0)`. Storage buffers (`GpuBuffer`) are mapped
starting at `binding(1)` when uniforms are present and at `binding(0)` otherwise.

## Example: a buffer-updating shader

The following shader updates the position and velocity of buffer elements. Because `Vec3` is a
compiler-provided mathematical type, it is used directly in the GPU layout rather than redefined.

```lisp
(defstruct Particle :repr gpu
  (pos : Vec3) (vel : Vec3) (life : f32))

(const GRAVITY : Vec3 (Vec3 0.0f -9.8f 0.0f))

#[shader :thread-group-size [64 1 1]]
(defn update-particles
  [(particles : [& mut [GpuBuffer Particle]]) (dt : f32)] -> ()
  (let tid (gpu/thread-id-x))
  (when (< tid (gpu-buf/len particles))
    (let p (gpu-buf/read particles tid))
    (let new-vel (Vec3/add p.vel (Vec3/scale GRAVITY dt)))
    (let new-pos (Vec3/add p.pos (Vec3/scale new-vel dt)))
    (gpu-buf/write particles tid (Particle new-pos new-vel (- p.life dt)))))
```

Call it as follows from CPU code that has prepared `initial-particles : [Vec Particle]`:

```lisp
#[external]
(defn main [] -> ()
  (let mut particles (GpuBuffer/from-slice @gpu (& initial-particles)))
  (let dt 0.016f)  ; 60FPS
  ;; Update 1,024 particles
  (gpu/dispatch update-particles :threads [1024 1 1] :args [(& mut particles) dt])
  (let result (GpuBuffer/to-vec particles)))
```

See [gpu-vector-scale.gu](../../examples/performance/gpu-vector-scale.gu) for a complete,
runnable single-file example that creates a buffer, dispatches the shader, and reads the result
back.

## Backends

| Backend | Description | Activation |
|------------|------|--------|
| CPU reference (default) | Runs invocations on the CPU while preserving workgroup-shared, barrier, and atomic semantics | Leave `SINGULISP_GPU_BACKEND` unset or set it to `cpu` |
| Vulkan | Executes on a real GPU through Singulisp → WGSL → naga → SPIR-V → ash/Vulkan | `SINGULISP_GPU_BACKEND=vulkan` |

```
SINGULISP_GPU_BACKEND=vulkan \
  gu-cli run examples/performance/gpu-vector-scale.gu --release
```

Set `SINGULISP_DUMP_WGSL=1` to print generated WGSL for debugging.

Testing with lavapipe, Mesa's software Vulkan implementation:

```
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json \
VK_LOADER_DRIVERS_SELECT=lvp_icd.json \
SINGULISP_GPU_BACKEND=vulkan \
  gu-cli run examples/performance/gpu-vector-scale.gu --release
```

## WGSL generation

The compiler automatically generates WGSL (WebGPU Shading Language) from shader functions. This
transformation runs as part of the compilation pipeline.

Singulisp types and control structures are converted to corresponding WGSL syntax, and `GpuBuffer`
access is converted to WGSL storage-buffer bindings.

```bash
gu-cli asm examples/performance/gpu-vector-scale.gu \
  --function scale_values \
  --format wgsl
```

| Singulisp | WGSL |
|-----------|------|
| `i32` | `i32` |
| `u32` | `u32` |
| `f32` | `f32` |
| `bool` | `bool` |
| `[Array T N]` | `array<T, N>` |
| `:repr gpu` struct | WGSL `struct` |
| `[GpuBuffer T]` | `array<T>` (storage buffer) |
