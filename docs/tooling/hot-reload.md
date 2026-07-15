# Hot Reload

Hot reload is a development feature that routes `#[reloadable]` functions through an indirect dispatch table and replaces function bodies in a running process over IPC. It is intended to shorten iteration time during performance tuning, not to improve execution speed itself.

## Reloadable Functions

```lisp
#[reloadable]
(defn transform [(value : f32) (scale : f32)] -> f32
  (* value scale))
```

Lowering statically enforces two primary restrictions:

- The attribute cannot be applied to a generic function (`E-HOT-0001`)
- It cannot be combined with `#[shader]` (`E-HOT-0002`)

During injection, the signature hash of the running function must match that of the new function. Restart the process after changing the arguments or return type.

## Starting a Process

```bash
gu-cli run src/main.gu --hot
gu-cli run . --hot
```

The startup log displays the process PID and IPC endpoint. The input may be a single `.gu` file or a project directory.

## Injecting a Patch

After modifying the source, specify the PID, function name, and source file.

```bash
gu-cli inject --pid 1234 --fn transform --src src/main.gu
```

`inject` builds the target function as a shared library and sends it to the IPC server. Use `--out` to select the patch artifact's output path and `--verbose` for detailed logging.

## Observing a Process

```bash
gu-cli hot list 1234
gu-cli hot list 1234 --format json
gu-cli hot status 1234
gu-cli hot status 1234 --format json
```

`hot list` displays reloadable function names, signature hashes, successful generations, and the time of the most recent reload. `hot status` displays the PID, function count, and numbers of successful and failed injections.

## Execution Model

1. A `--hot` build embeds dispatch slots for reloadable functions and an IPC server.
2. `inject` generates code for the new body as a shared library.
3. The server compares signature hashes.
4. If they match, the dispatch slot is updated atomically.
5. Subsequent calls enter the new body.

Call frames that are already running complete the old body. Hot reload does not apply changes that cross global-data layouts, type layouts, or function signatures without restarting the process.

## Relationship to Release Builds

Hot mode is an indirect-call path for debug and development builds. Recheck final speed, assembly, and inlining with a normal `--release` build.
