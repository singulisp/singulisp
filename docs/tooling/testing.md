# Testing and Benchmarking

The Singulisp compiler collects `deftest` and `defbench` forms directly and builds them as dedicated runners. A normal `main` function is not required.

## Tests

```lisp
(deftest "integer arithmetic"
  (assert-eq! (+ 1 2) 3)
  (assert-ne! (* 6 7) 41)
  (assert! (< 0 1)))

(deftest "Vec3 length"
  (let v (Vec3 3.0f 4.0f 0.0f))
  (assert! (< (f32/abs (- (Vec3/length v) 5.0f)) 0.0001f)))
```

```bash
gu-cli test tests/math.gu
gu-cli test tests/math.gu --filter Vec3
gu-cli test tests/math.gu --memory-limit 256
gu-cli test .
```

Test input may be a single `.gu` file or a project directory. `-f` / `--filter` performs a substring match against test names. `--memory-limit` sets the child process limit in MB; the default is 500, the minimum is 16, and `0` means unlimited.

Each test runs independently in its own subprocess. An assertion failure, abort, signal, or runtime trap marks that test as `FAIL`, and the remaining tests continue. `deftest` has no annotation that treats a trap as success, so cases such as division by zero must not be written directly as successful examples.

### Assertions

```lisp
(assert! condition)
(assert-eq! actual expected)
(assert-ne! actual unexpected)
```

A failure is a runtime trap, not a recoverable exception. See [Built-in APIs](../standard-library/builtins.md#assertions) for the types supported by `assert-eq!` and `assert-ne!`.

## Benchmarks

`defbench` requires `:iterations N`, where `N >= 3`. Its body is a pure context, so it cannot contain `println!` or file I/O.

```lisp
(defbench "Vec3 dot" :iterations 1000000
  (let a (Vec3 1.0f 2.0f 3.0f))
  (let b (Vec3 4.0f 5.0f 6.0f))
  (Vec3/dot a b))
```

```bash
# Build and run a release runner
gu-cli bench benches/vector.gu --timeout 30
gu-cli bench . --timeout 30

# Substring filter
gu-cli bench benches/vector.gu --filter dot --timeout 30

# Separate building from execution
gu-cli bench-build benches/vector.gu --filter dot -o build/vector.bench-runner
gu-cli bench-build . -o build/project.bench-runner
gu-cli bench-run build/vector.bench-runner --timeout 30
```

The input to `bench` and `bench-build` may also be a single `.gu` file or a project directory. `--timeout` is required for `bench` and `bench-run` to prevent hangs. As with tests, the default memory limit is 500 MB, the minimum is 16 MB, and `0` means unlimited. The `bench-build` filter is embedded into the generated runner; `bench-run` has no filter option.

After a warm-up, the runner measures and displays the iteration count and elapsed time for each benchmark. Absolute time depends on the CPU, compiler revision, and build options, so fixed values in documentation must not be treated as performance guarantees.

## Measuring I/O

Do not put an end-to-end workload that includes file I/O in `defbench`. Build a normal program with `gu-cli run --release` and measure it from an external harness. When comparing only a kernel, a pure input generator with a fixed seed makes the conditions easier to reproduce.
