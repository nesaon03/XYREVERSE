---
name: rea-automated-analysis
description: Use when driving REA for automated binary analysis. CLI/MCP, script discovery, app graph extraction.
---

# REA Automated Analysis

## Toolchain
REA CLI v6.4.0 (`rea` / `rea-agents`), the REA MCP server (114 tools), Node 26, Playwright/CDP.

## Surface

```bash
npm i -g rea-agents            # CLI
rea --version
rea --help                     # command inventory
rea --mcp                      # run as an MCP stdio server
```

The CLI covers PE/ELF/Mach-O, .NET, Electron/ASAR, web bundles, IL2CPP, and driving IDA/Ghidra headlessly. The MCP server exposes the same engine as tools so an agent can call it directly (`--mcp`, stdio transport).

Register with a client:

```json
{
  "mcpServers": {
    "rea": { "command": "rea", "args": ["--mcp"] }
  }
}
```

## Typical flow

1. **Orient** — `rea` script discovery, then a session open on the target so the analysis result lives across calls.
2. **Extract** — artifact/script/assembly listing, string and function enumeration.
3. **Analyze** — call graph, xrefs, decompile at a specific address.
4. **Graph the app** — for an Electron or web target, `analyze-javascript-application` builds the source → bundle → runtime map; UI clicks are correlated with the binary/CDP layer underneath.
5. **Evidence bundle** — export the findings, diff across versions, and verify the reconstruction coverage.

Commands worth knowing (names vary slightly by version — enumerate with `--help`):

| Task | Surface |
| --- | --- |
| session / artifact open | `binary-session`, `open-binary`, `inspect-artifact` |
| metadata | `binary-overview`, `inspect-binary-layout`, `list-segments`, `list-names` |
| code | `decompile`, `analyze-function`, `search-procedures`, `get-call-graph` |
| strings | `list-strings`, `search-strings` |
| JS/Electron | `analyze-javascript-application`, `recover-javascript-sources`, `inspect-electron-page` |
| .NET | `inspect-managed-artifact`, `inspect-managed-members` |
| IDA bridge | `open_binary` / `decompile_function` / `search_procedures` against a live IDA session |
| verification | `compare-artifacts`, `compare-functions`, `diff-images`, `verify-reconstruction` |

## Agent patterns

- REA is a *tool server*, not a model. Drive it from a script or an MCP-capable agent; the value is the automation, not the answers.
- Feed it a target and let the agent iterate: orient → hypothesis → decompile the candidate → verify → next.
- For cross-version work, keep the artifact ids from each session and use `compare-*` rather than re-analyzing by hand.
- Playwright/CDP tools (`capture-web-screenshot`, `observe-web-session`, browser targets) make it usable on web apps and Electron shells, including the JS boundary.

## Pitfalls

- The CLI is a **Node 26** install; on Windows it lands under `%LOCALAPPDATA%\hermes\tools\node-26.7.0-win32-x64\`. Invoking `rea` from a shell that does not have that PATH entry fails with "command not found" — use the full path or add it.
- Global installs need the package manager's global bin on PATH; `npm i -g rea-agents` puts the shim next to `node`, not wherever `rea` was previously.
- IDA/Ghidra bridge tools require that instance **already running** with a matching plugin; they do not launch it.
- Long analyses stream progress — do not poll on stdout for a completion marker, wait for the process to exit.
- `--mcp` speaks stdio. Wrapping it in anything that also writes to stdout corrupts the JSON-RPC stream.
- Version drift: tool names in the 100+ set move between releases. Enumerate tools at start rather than hardcoding the call list.
