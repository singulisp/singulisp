# Writing Fast Code

In Singulisp, speed is verified through types, ownership, data layout, function attributes, and analysis
results—not treated as something the optimizer happens to discover. First measure a correct baseline,
then remove obstructing contracts one at a time.

## Use a release build as the baseline

```bash
gu-cli build program.gu --release
gu-cli run program.gu --release
```

Debug builds, hot reload, and JIT serve different purposes. Make final performance decisions using an
AOT release binary with a fixed target and build mode.

## Make allocation visible

`String`, `Box`, and `Vec` have global provenance by default. If temporary allocations cluster on a hot
path, consider moving them to a bump arena or pool.

```lisp
(defregion @scratch
  :allocator (arena :bump)
  :lifetime :scoped
  :thread :local
  :default-capacity (MiB 4)
  :grow :fixed
  :on-oom :trap
  :reuse :reset)
```

Arenas are not universal. Long-lived values, values whose ownership must be released individually, and
values that cross thread boundaries require different policies. See [Regions](../language/regions.md) for details.

## Keep aliases and borrows simple

Separate read references from mutable references so the optimizer can prove that memory accesses are
independent. Do not pass overlapping `& mut` references to the same call, and borrow only the required
range. MIR and the backend also use ownership and borrowing information for `readonly` and `noalias` decisions.

## Write loops whose bounds checks can be eliminated

Keep the relationship between indices and lengths simple.

- Do not modify a loop's induction variable irregularly within the loop.
- Establish the length outside the loop.
- Pair a length and index obtained from the same collection.
- Reshape the loop instead of hiding unchecked access that cannot be proved safe.

Inspect remaining bounds checks with `gu-cli analyze`, MIR, and assembly.

## Choose a data layout

- `Array` — compile-time fixed-length contiguous data
- `Vec` — variable-length contiguous data
- `SoaArray` / `SoaVec` — Structure of Arrays with each field stored contiguously
- `Float3` — 12-byte packed storage
- `Vec3` — a 16-byte first-class SIMD mathematical type

Choose between AoS and SoA based on the access pattern. SoA is natural when scanning the same field
across many elements; AoS is natural when frequently handling entire records.

## Choose the level of numeric abstraction

Use [Built-in Mathematical Types](../language/math-types.md) for ordinary vector and matrix computations.
Use [Low-Level SIMD](simd.md) when direct control of lanes, masks, or shuffles is required. Inspect the
generated code before implementing the same computation twice in scalar and SIMD forms.

## Floating-point policy

```lisp
#[fp-mode :contract]
(defn fused [(a : f32) (b : f32) (c : f32)] -> f32
  (+ (* a b) c))
```

- The default mode preserves ordinary IEEE semantics.
- `#[fp-mode :contract]` permits contraction.
- `#[fp-mode :fast]` permits broader transformations such as reassociation.
- `#[deterministic]` prioritizes reproducibility using implementation-provided kernels and target conditions.

Changing modes can change result bit patterns. Decide the acceptable numeric semantics before optimizing for speed.

## SPMD and GPU

[SPMD](spmd.md) makes CPU lane parallelism explicit and exposes the costs of conflicting scatters,
reductions, and atomics. [GPU](gpu.md) has a separate address space and shader subset. Transfers,
synchronization, and fallbacks must remain explicit.

## Inspect individual functions

```bash
gu-cli analyze program.gu --kind vectorize --function hot_function
gu-cli dump program.gu --stage mir --function hot_function
gu-cli asm program.gu --function hot_function
```

See the [CLI Reference](../tooling/cli.md) for available `--kind`, `--stage`, and output formats.
Compare diagnostics, MIR, and assembly using the same function name and target.

## Benchmark

Use `defbench` after fixing semantics with `deftest`. Keep I/O, first-use cache effects, and allocation
warmup out of the measured work, and record the input size and target.

```bash
gu-cli test project/
gu-cli bench project/ --timeout 30
```

## Checklist

- [ ] Are the release AOT mode, target, and input fixed?
- [ ] Is allocation provenance on the hot path understood?
- [ ] Are borrows and aliases simple?
- [ ] Does the loop shape permit bounds-check proofs?
- [ ] Do AoS, SoA, packed, and SIMD types match the access pattern?
- [ ] Has the floating-point policy been chosen explicitly?
- [ ] Are SPMD/GPU conflicts and transfers explicit?
- [ ] Have measurements and assembly been checked in addition to optimization reports?
