# MCP Server

`singulisp-mcp` exposes compiler operations such as building, analysis, formatting, and testing as Model Context Protocol tools over stdio.

## Starting the Server

```bash
singulisp-mcp
singulisp-mcp --project-root /absolute/path/to/project
```

The server starts `gu-cli` as a subprocess. If it is installed elsewhere, set `SINGULISP_MCP_SINGULISP_BIN` to the absolute path of `gu-cli`.

```bash
SINGULISP_MCP_SINGULISP_BIN=/absolute/path/to/gu-cli singulisp-mcp
```

Moving only `gu-cli` may prevent it from finding required resources. Preserve the distribution's directory layout.

`--project-root` selects the working context and the default paths for the staged toolchain, cache, and reload operations. It is not a security sandbox that restricts file-system access.

## Exposed Tools

`tools/list` exposes 22 tools.

| Tool | Purpose |
|------|---------|
| `singulisp.build` | Build |
| `singulisp.dump` | MIR dump |
| `singulisp.analyze` | Compiler analysis |
| `singulisp.asm` | LLVM IR, assembly, IR, MIR, or GPU output |
| `singulisp.explain` | Error-code explanation |
| `singulisp.llm-cheatsheet` | Embedded cheat sheet |
| `singulisp.llm-performance-guide` | Embedded performance guide |
| `singulisp.llm-development-workflow` | Embedded workflow |
| `singulisp.docs` | Access to embedded documentation |
| `singulisp.test` | Tests |
| `singulisp.bench` | Benchmarks |
| `singulisp.doc` | API documentation generation |
| `singulisp.fmt` | Format or check |
| `singulisp.inject` | Hot-patch injection |
| `singulisp.hot-list` | List reloadable functions |
| `singulisp.hot-status` | Hot-process status |
| `singulisp.bindgen` | C binding generation |
| `singulisp.cache-clear` | Cache removal |
| `singulisp.cache-status` | Cache status |
| `singulisp.cache-doctor` | Cache inspection and repair |
| `singulisp.capabilities` | Server capabilities |
| `singulisp.reload` | Update the MCP-specific toolchain |

Consult `tools/list` on the running server instance for the argument schema of each request.

## Example Request

An MCP client normally generates this JSON-RPC automatically. The following is a minimal protocol-level example.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "singulisp.analyze",
    "arguments": {
      "file": "/absolute/path/to/kernel.gu",
      "kind": "vectorize",
      "function": "transform",
      "release": true
    }
  }
}
```

## File Paths

Tools that accept a file normalize a relative path once before passing it to the CLI. The meaning of the path does not change when the subprocess working directory changes, and both absolute and relative paths are accepted.

`--project-root` is the working context for resolving file paths, not a security-sandbox boundary.

## Analysis and Builds

The `kind` values accepted by `singulisp.analyze` are the same as those accepted by the CLI: `alias`, `effect`, `vectorize`, `escape`, `throughput`, `recurrence`, `speed`, `shape`, `profitability`, `substrate`, `divergence`, `bitloop`, `backend-summary`, and `stack-footprint`.

The CPU tuning arguments for MCP builds map to the following CLI options and undergo the same validation.

| MCP Argument | CLI Option |
|--------------|------------|
| `target-cpu` | `--target-cpu` |
| `target-features` | `--target-features` |

## Cache

The cache scopes in the public schema are `incremental`, `object`, `runtime`, and `all`. Even if the server has internal summary or template caches, clients must not send values that are absent from `tools/list`.

## Hot Reload

```json
{
  "name": "singulisp.inject",
  "arguments": {
    "pid": 1234,
    "fn_name": "transform",
    "src": "/absolute/path/to/main.gu"
  }
}
```

`hot-list` and `hot-status` take the PID of a running process. See [Hot Reload](hot-reload.md) for the CLI workflow and restrictions.

## Reload

`singulisp.reload` is a development tool that updates the MCP-specific toolchain. It requires a development source tree, Cargo, and `--project-root`, and is unavailable in a standalone binary distribution. It does not update the running MCP server itself; reconnect the client after the update.

## Operational Considerations

- Run the MCP server only in trusted projects.
- Do not treat `--project-root` as access control.
- Build, test, bindgen, and reload operations start subprocesses and write to the file system.
- Always specify a timeout for benchmarks.
- Send only arguments present in the running server's `tools/list` schema.
