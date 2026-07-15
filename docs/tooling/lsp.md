# LSP Server

Singulisp includes a Language Server Protocol (LSP) server. Through editor integration, it provides real-time type checking, completion, go to definition, and other development features.

---

## Starting the Server

```bash
gu-cli lsp
```

Normally, the editor starts the LSP server automatically, so there is no need to run it manually. Configure the editor to invoke `gu-cli lsp`.

---

## Request Handlers (12)

| LSP Method | Description |
|------------|-------------|
| `textDocument/hover` | Displays a symbol's type signature and documentation as Markdown at the cursor position |
| `textDocument/definition` | Goes to the definition location |
| `textDocument/references` | Returns all reference locations |
| `textDocument/rename` | Renames a symbol and returns a `WorkspaceEdit` |
| `textDocument/completion` | Completes keywords, built-in functions, and indexed symbols |
| `textDocument/signatureHelp` | Displays a signature during a function call |
| `textDocument/documentSymbol` | Returns document symbols in hierarchical or flat form |
| `workspace/symbol` | Searches for symbols across the workspace |
| `textDocument/formatting` | Formats the entire document, equivalent to `gu-cli fmt` |
| `textDocument/codeAction` | Returns code actions, including formatting suggestions |
| `textDocument/semanticTokens/full` | Returns semantic tokens for the entire document in one response |
| `textDocument/inlayHint` | Returns inlay hints for return types, local variable types, and argument names |

## Notification Handlers (7)

| Notification Method | Behavior |
|---------------------|----------|
| `textDocument/didOpen` | Registers the document with the server and publishes diagnostics immediately |
| `textDocument/didChange` | Applies incremental changes and publishes diagnostics after a **100 ms debounce** |
| `textDocument/didSave` | Records the document save and publishes diagnostics immediately |
| `textDocument/didClose` | Removes the document and sends a notification that clears its diagnostics |
| `workspace/didChangeConfiguration` | Marks the workspace dirty so it is reanalyzed on the next operation |
| `workspace/didChangeWatchedFiles` | Recollects the workspace file list |
| `$/cancelRequest` | Marks the request with the specified ID as canceled |

After receiving `textDocument/didChange`, the server waits for a 100 ms debounce window before publishing diagnostics. This prevents multiple diagnostic computations from starting during rapid editing.

## Inlay-Hint Kinds

| Kind | Display Format | Insertion Position |
|------|----------------|--------------------|
| Function return type | ` -> <type-name>` | End of the function-name identifier |
| Local variable type | ` : <type-name>` | End of the variable identifier |
| Call argument label | `<argument-name>:` | Immediately before each argument |

### Completion Trigger Characters

Completion suggestions appear automatically when any of the following characters is entered.

| Character | Purpose |
|-----------|---------|
| `(` | Beginning a function call |
| `/` | Namespaced functions, such as `f32/` |
| `.` | Field access |

---

## Editor Configuration

### VS Code

See [VS Code Extension](vscode-extension.md) for the official extension's requirements and features. To change the CLI location, add the following to `settings.json`:

```json
{
  "singulisp.serverPath": "/path/to/gu-cli"
}
```

If `gu-cli` is on `PATH`, the path may be omitted; the default is `gu-cli`.

```json
{
  "singulisp.serverPath": "gu-cli"
}
```

### Neovim (nvim-lspconfig)

Add the following to `init.lua`:

```lua
require('lspconfig').gu.setup {
  cmd = { 'gu-cli', 'lsp' },
  filetypes = { 'singulisp' },
  root_dir = function(fname)
    return vim.fn.fnamemodify(fname, ':h')
  end,
}
```

For a project containing `Singulisp.toml`, setting `root_dir` based on detection of `Singulisp.toml` is recommended.

```lua
require('lspconfig').gu.setup {
  cmd = { 'gu-cli', 'lsp' },
  filetypes = { 'singulisp' },
  root_dir = require('lspconfig.util').root_pattern('Singulisp.toml'),
}
```

### Emacs (lsp-mode)

```elisp
(with-eval-after-load 'lsp-mode
  (add-to-list 'lsp-language-id-configuration '(singulisp-mode . "singulisp"))
  (lsp-register-client
   (make-lsp-client
    :new-connection (lsp-stdio-connection '("gu-cli" "lsp"))
    :activation-fn (lsp-activate-on "singulisp")
    :server-id 'singulisp-lsp)))
```

### Helix

Add the following to `languages.toml`:

```toml
[[language]]
name = "singulisp"
scope = "source.gu"
file-types = ["singulisp"]
language-servers = ["singulisp-lsp"]

[language-server.singulisp-lsp]
command = "gu-cli"
args = ["lsp"]
```

---

## Feature Details

### Hover (Type Information)

Hovering over a function or variable name displays its type information in a pop-up.

```singulisp
(defn square [(n : i32)] -> i32)
```

For a struct, the signature includes field information.

```singulisp
(defstruct Point (x : f32) (y : f32))
```

### Completion

When completion is invoked inside an S-expression, the following suggestions appear:

- Keywords such as `defn`, `defstruct`, `let`, `if`, and `match`
- In-scope function and variable names
- Built-in functions from every namespace exposed by the compiler
- Special forms such as `assert!`, `assert-eq!`, `assert-ne!`, and `String/push!`, including forms processed by lower layers

Built-in function completion covers every namespace.

| Namespace | Examples |
|-----------|----------|
| Casts | `to-i32`, `to-f64`, `i32/from` |
| Bit operations | `bit-and`, `bit-or`, `bit-shl` |
| f32/f64 mathematics | `f32/sqrt`, `f32/sin`, `f32/fma`, `f64/lerp` |
| Mathematical types (SIMD built-ins) | `Vec3/cross`, `Vec3/dot`, `Mat4/mul`, `Quat/slerp` |
| SIMD (low level) | `simd/add`, `simd/splat`, `simd/dot` |
| Strings | `String/len`, `String/split`, `String/parse-i64` |
| Vec / Array | `Vec/push`, `Vec/get`, `Array/map`, `Array/fold` |
| SoA (dynamic) | `SoaVec/push`, `SoaVec/gather`, `SoaVec/get-field` |
| SoA (fixed length) | `SoaArray/get-field`, `SoaArray/gather`, `SoaArray/scatter` |
| Persistent collections | `PVec/push`, `PMap/assoc`, `PMap/get` |
| Files / streams | `fs/read-to-string`, `stream/open-reader` |
| Concurrency | `channel/new`, `scope/spawn` |
| Arenas / regions | `arena/alloc!`, `arena/try-alloc` |
| GPU | `gpu/dispatch`, `gpu/thread-id-x`, `gpu-buf/read` |
| SPMD | `spmd/reduce`, `spmd/scan`, `spmd/scatter!` |
| Prefetching | `prefetch/read`, `prefetch/write` |

### Go to Definition

Invoking Go to Definition on a function call or type name navigates to its definition. The workspace is constructed from `Singulisp.toml`, and resolution includes other files in the same package and local packages referenced by path dependencies.

### Find References

Invoking Find All References on a symbol lists its definition and reference locations throughout the workspace. Resolution is symbol-based rather than a word match, so it distinguishes same-named symbols and shadowed local variables.

### Rename

Invoking Rename Symbol changes the name at its definition and every reference. References that cross files are included in the same `WorkspaceEdit`.

### Diagnostics (Real-Time Error Display)

The compiler runs when a file is opened and after each change or save, displaying errors and warnings in the editor. During rapid changes, diagnostics are sent through `publishDiagnostics` after a short debounce. Error codes are included, so details can be inspected with `gu-cli explain`.

### Position Information and Progress

The Singulisp LSP explicitly returns `positionEncoding: utf-16` and treats LSP `Position.character` values as UTF-16 offsets. Requests containing a `workDoneToken` receive `$/progress` notifications, allowing supported clients to display progress.

---

## Communication Protocol

The LSP server communicates over standard input and output (stdio) using JSON-RPC 2.0. Its lifecycle with an editor is as follows:

1. The editor starts the process.
2. The `initialize` request negotiates server capabilities.
3. The server provides diagnostics and completion in response to file operations such as open, change, and save.
4. When the editor exits, `shutdown` followed by `exit` terminates the process.
