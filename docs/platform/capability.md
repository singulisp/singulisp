# Capability Model

A Singulisp capability is metadata describing what code depends on; it is not normally a build gate.
Only `hard-needs` in the package manifest can stop a build.

---

## Capabilities

| Capability | Purpose |
|-----------|------|
| `ext.time` | Time measurement |
| `ext.thread` | Thread creation and join |
| `ext.sync` | Mutexes and condition variables |
| `ext.fs` | Filesystem |
| `ext.dl` | Dynamic libraries |
| `ext.io` | Standard input and output |

The complete Singulisp capability set consists of the six entries above. A platform bundle declares
only absent capabilities through the `missing` blacklist.

---

## `#[require]`

`#[require ext.*]` is soft-dependency metadata.

```lisp
#[require ext.time]
(defn sample-now [] -> i64
  (time/now-ms))
```

### Properties

- Retained as dependency information through lowering, IR, and tooling
- Does not stop a build by default
- Not used directly to evaluate package hard requirements

In other words, `#[require]` declares a dependency; it does not let a function stop a build by itself.

---

## Conditions that stop a build

Only `package.hard-needs` in `Singulisp.toml` stops a build.

```toml
[package]
hard-needs = ["ext.thread", "ext.sync"]
```

Given the following target-bundle metadata:

```toml
llvm-triple = "wasm32-wasip1"
missing = ["ext.thread", "ext.sync", "ext.dl"]
```

the conflict between `hard-needs` and `missing` produces a build error.

---

## Runtime semantics

When code with a soft dependency runs on a target whose `missing` list contains that dependency,
behavior is determined by the build mode:

- `debug`: trap
- `release`: UB

Singulisp semantics fix this policy; users cannot select another behavior.

---

## Relationship to target selection

Available capabilities are the complete set minus `missing`. Because the selected target also determines
`os`, `abi`, `pointer-width`, and `llvm-triple`, `cfg-case (os ...)` is evaluated against the selected
target rather than the host.
