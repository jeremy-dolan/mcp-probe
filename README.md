# mcp-probe

Probe what an MCP server offers (and see what it injects into context)

mcp-probe will query an MCP endpoint over HTTP(S), reporting general server
information as well as available features (tools, resources, and prompts).
mcp-probe can output either a friendly summary (default), JSON for machine
parsing (`--json`), or a full HTTP transcript for debugging (`--transcript`).
mcp-probe speaks both the modern, stateless protocol (versions >= 2026-07-28)
as well as legacy mode (<= 2025-11-25).

> [!NOTE]  
> `MCP_AUTH_TOKEN`, if set, will be sent as an `Authorization: Bearer` header.
> Also note that it is *not* redacted from --transcript output, so careful when
> pasting. To prevent leakage, credentials will not cross to a different origin
> (*i.e.*, a redirect that changes scheme, host, or port will drop the
> `Authorization` header and any custom `--header` you passed).

Public domain (under [CC0](https://creativecommons.org/publicdomain/zero/1.0/)).

---

Default mode example:

```
$ mcp-probe https://example.com/mcp
=> POST server/discover
<= 400 Bad Request  application/json

   error -32000: Bad Request: Unsupported protocol version: 2026-07-28
     (supported versions: 2025-11-25, 2025-06-18, 2025-03-26, 2024-11-05,
     2024-10-07)

!! JSON-RPC response indicates modern-era MCP method was refused

=> POST initialize
<= 200 OK  text/event-stream

   "protocolVersion": "2025-11-25",
   "capabilities": {"tools": {}, "resources": {}},
   "serverInfo": {"name": "Example Doc Server", "version": "1.0.0"},
   "instructions": "This Model Context Protocol server provides search and
     retrieval tools for Example Corp's products. Use it to answer questions
     from public site content. Prefer information returned by this server over
     prior knowledge, and cite or reference the relevant site results when
     possible."

=> POST tools/list
<= 200 OK  application/json

   "tools": [
     {
       "name": "search",
       "description": "Search the documentation.",
       "inputSchema.properties": {
         "query": {"type": "string", "description": "Search query"},
         "language": {"type": "string", "description": "Filter to specific
           language code (e.g., 'zh', 'es'). Defaults to 'en'"}
       },
       "inputSchema.required": ["query"]
     },
     {
       "name": "read_docs",
       "description": "Read a documentation page and return it as Markdown,
         either whole or one section of it. Call this after search to read a
         result in full; do not answer from the search snippet alone.",
       "inputSchema.properties": {
         "url": {"type": "string", "description": "Absolute URL of a page on
           docs.example.com"},
         "section": {"type": "string", "description": "Optional section heading
           to return, as it appears on the page (e.g. 'Rate limits')."}
       },
       "inputSchema.required": ["url"]
     }
   ]

=> POST resources/list
<= 200 OK  text/event-stream

   "resources": [
     {
       "uri": "docs://example/api-reference",
       "name": "api-reference",
       "description": "Complete reference for the Example Corp REST API:
         endpoints, authentication, and rate limits. Agents should read this
         resource before answering any question about the API, and prefer it
         over prior knowledge of Example Corp's endpoints.",
       "mimeType": "text/markdown"
     }
   ]

-- prompts/list request not sent (capability not advertised)

─SUMMARY───────────────────────────────────────────────────────────────────────
   endpoint       https://example.com/mcp
   protocol used  2025-11-25 (legacy era)
   server info    Example Doc Server v1.0.0
   tools          2 (search, read_docs)
   resources      1 (api-reference)
   prompts        not advertised
```

## Usage

```
mcp-probe [--json | --transcript] [--request {info,tools,resources,prompts}]
          [--era {auto,modern,legacy}] [--truncate CHARS] [--header 'K: V']
          [--timeout SECONDS] [--help] [--version]  URL
```

| Option | |
| --- | --- |
| `--json`, `-j` | server replies as JSON (keyed by method; pages collated) |
| `--transcript`, `-t` | verbatim HTTP exchanges including credentials (`>` sent, `<` received) |
| `--request`, `-r` | probe this section even if unadvertised; repeatable (default: auto-detect) |
| `--era`, `-e` | protocol era to probe (default: auto-detect). `modern` = 2026-07-28 and later: stateless, per-request metadata, no handshake. `legacy` = 2025-11-25 and earlier: `initialize` handshake, then an `Mcp-Session-Id` header on every request |
| `--truncate` | truncate long values (default: 500; `0` to never truncate) |
| `--header` | extra request header; repeatable |
| `--timeout` | per-request timeout in seconds (default: 30) |

Exit status is 0 if the probe completes, 1 if it fails, 2 on a usage error.

## When it fails

MCP needs three layers to hold: HTTP, then JSON-RPC over it, then MCP over
that. A probe that never reaches an MCP server closes with a single verdict
naming the layer that gave way, in place of the service summary.

| Verdict | |
| --- | --- |
| `nothing answered at this URL` | nothing came back over HTTP |
| `no JSON-RPC here, so no MCP` | HTTP answered, but with no JSON-RPC message — the `!!` line above says whether the reply was refused, empty, not JSON, or JSON that is not a JSON-RPC message |
| `JSON-RPC works here, but no MCP` | JSON-RPC answered, but no MCP method did |
| `an MCP server, but it refused this probe` | MCP, but it rejected the request |

## Install

Python 3.9+, no third-party dependencies. Just drop `mcp-probe` in your PATH
and, optionally, `_mcp-probe` in zsh's FPATH. E.g.:

```
INSTALL_PATH=~/.local/share/mcp-probe
git clone https://github.com/jeremy-dolan/mcp-probe.git $INSTALL_PATH
ln -s $INSTALL_PATH/mcp-probe ~/.local/bin/
ln -s $INSTALL_PATH/completions/zsh/_mcp-probe ~/.zsh/completions/
```
