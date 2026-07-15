# VS Code Extension

The Singulisp VS Code extension combines an LSP client, block visualization, analysis overlays, pipeline visualization, and debugger integration.

---

## Requirements

- VS Code 1.85 or later
- The `gu-cli` executable must be available

If `gu-cli` is not on `PATH`, set `singulisp.serverPath` to its absolute path in the VS Code settings.

```json
{
  "singulisp.serverPath": "/absolute/path/to/gu-cli"
}
```

## Features

- Opening a `.gu` file starts `gu-cli lsp`
- Hover, completion, go to definition, find references, and rename are available
- The `Singulisp: Restart LSP Server` command restarts the server
- Pipeline visualization and analysis overlays run as needed

## Troubleshooting

If LSP results appear inconsistent, first try restarting the server.

- VS Code Command Palette: `Singulisp: Restart LSP Server`
- CLI: `gu-cli cache doctor --scope incremental`
- CLI: `gu-cli cache doctor --scope all --fix`
- CLI: `gu-cli cache clear --scope all`

When operating through MCP, use `singulisp.cache-status`, `singulisp.cache-doctor`, and `singulisp.cache-clear`.
