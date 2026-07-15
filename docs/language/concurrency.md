# Structured Concurrency

Singulisp expresses CPU concurrency with `with-scope`, `scope/spawn`, and `sync`. Spawned tasks are joined before the scope exits, so they cannot leak.

## with-scope and scope/spawn

```lisp
(with-scope scope
  (scope/spawn scope (fn [] -> () (task-a)))
  (scope/spawn scope (fn [] -> () (task-b))))
;; Both tasks have completed here
```

The first argument to `scope/spawn` is the `ScopeHandle` that is active at that point. A task has type `[Fn [] -> ()]` and must be either an inline closure or a function reference whose capture information can be checked at the boundary. A function value previously stored in a local variable is rejected because its capture metadata cannot be verified.

Captures are not categorically prohibited. Values that are both `Copy` and thread-safe may be captured. Borrowed references, values originating from a thread-local arena, I/O handles, GPU buffers, `ModuleHandle`, and similar values are rejected by the boundary check.

## sync

`sync` runs multiple `[Fn [] -> ()]` branches concurrently and waits for all of them.

```lisp
(sync
  (fn [] -> () (transform-partition 0))
  (fn [] -> () (transform-partition 1))
  (fn [] -> () (transform-partition 2)))
```

A branch must likewise be an inline closure or a function reference that can be checked.

## ChannelHandle

A channel is a bounded MPMC queue. Its surface type is the non-generic `ChannelHandle`, not a generic `[Channel T]`. The only payload type is `i32`.

| Operation | Type |
|---|---|
| `channel/new` | `i32 -> ChannelHandle` |
| `channel/send` | `ChannelHandle i32 -> ()` |
| `channel/recv` | `ChannelHandle -> i32` |
| `channel/close` | `ChannelHandle -> ()` |

Send blocks when the channel is full, and receive blocks when it is empty. Sending to a closed channel traps with `T-CHAN-0002`; receiving from a closed, empty channel traps with `T-CHAN-0001`.

```lisp
#[external]
(defn run-pipeline [] -> ()
  ;; Creating the owner before the scope proves that it lives through the join
  (let channel (channel/new 32))
  (with-scope scope
    (scope/spawn scope (fn [] -> ()
      (loop [(i : i32 0)]
        (when (< i 8)
          (channel/send channel (* i i))
          (recur (+ i 1))))))
    (scope/spawn scope (fn [] -> ()
      (loop [(i : i32 0)]
        (when (< i 8)
          (println! "value={}" (channel/recv channel))
          (recur (+ i 1)))))))
  (channel/close channel))
```

When a task borrows a channel's owner, the compiler verifies that the owner existed before the scope began and remains alive until after the join. Because `i32` is the only payload type, there is no API for sending references or owned objects through a channel.

## Thread Boundaries and Regions

The principal rules are as follows.

- A task closure must not capture `[& T]` or `[& mut T]`.
- Handles and values from a `:thread :local` arena do not cross the boundary.
- Handles and owned values from a `:thread :async` heap arena may be captured even with a scoped lifetime if the join in the same `with-scope` can be proven to occur before the region exits.
- Retaining a value beyond the scope with `:async-hold true` requires all of `(arena :heap)`, `:lifetime :drop`, and `:thread :async`.
- A fresh closure or function reference is accepted, but an existing `[Fn ...]` local whose captures cannot be rechecked is rejected.
- `ModuleHandle`, `Reader`, `Writer`, and `GpuBuffer` do not cross CPU task boundaries.

See [Regions](regions.md) for region thread policies and [Ownership](ownership.md) for value-movement rules.

## Difference from GPU Tasks

`await` waits for a GPU task handle returned by `gpu/dispatch-async`. It does not await an individual CPU task spawned with `scope/spawn`. CPU tasks are joined according to the structure of `with-scope` and `sync`.
