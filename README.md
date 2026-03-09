<p align="center">
  <img src="assets/hero.png" alt="mcp2cli — one CLI for every API" width="700">
</p>

<h1 align="center">mcp2cli</h1>

<p align="center">
  Turn any MCP server or OpenAPI spec into a CLI — at runtime, with zero codegen.
</p>

---

## The problem: tool sprawl is eating your tokens

If you've connected an LLM to more than a handful of tools, you've felt the pain. Every MCP server, every OpenAPI endpoint — their full schemas get injected into the system prompt on *every single turn*. Your 50-endpoint API costs 3,579 tokens of context *before the conversation even starts*, and that bill is paid again on every message, whether the model touches those tools or not.

This isn't a theoretical concern. [Kagan Yilmaz documented it well](https://kanyilmaz.me/2026/02/23/cli-vs-mcp.html) in his analysis of CLI vs MCP costs, showing that 6 MCP servers with 84 tools consume ~15,540 tokens at session start. His project [CLIHub](https://kanyilmaz.me/2026/02/23/cli-vs-mcp.html) demonstrated that converting MCP servers to CLIs and letting the LLM discover tools on-demand slashes that cost by 92-98%.

mcp2cli takes that insight and runs further with it.

## What mcp2cli adds

CLIHub showed the path: give the LLM a CLI instead of raw tool schemas, and let it `--list` and `--help` its way to what it needs. mcp2cli builds on that idea with a few key differences:

- **No codegen, no recompilation.** Point mcp2cli at a spec URL or MCP server and the CLI exists immediately. When the server adds new endpoints, they appear on the next invocation — no rebuild step, no generated code to commit.
- **OpenAPI support.** MCP isn't the only schema-rich protocol. mcp2cli handles OpenAPI specs (JSON or YAML, local or remote) with the same CLI interface, the same caching, and the same on-demand discovery. One tool for both worlds.
- **Spec caching with TTL control.** Fetched specs and MCP tool lists are cached locally with configurable TTL, so repeated invocations don't hit the network. `--refresh` bypasses the cache when you need it.
- **Single binary, zero dependencies on the server.** The server doesn't need to know mcp2cli exists. It works with any compliant OpenAPI spec or MCP server out of the box.

```bash
# OpenAPI
mcp2cli --spec https://api.example.com/openapi.json list-users --limit 10

# MCP over stdio
mcp2cli --mcp-stdio "npx @modelcontextprotocol/server-filesystem /tmp" list-directory --path /tmp

# MCP over HTTP/SSE
mcp2cli --mcp https://mcp.example.com/sse echo --message "hello"
```

## The numbers: how much context do you actually save?

We measured this. Not estimates — actual token counts using the cl100k_base tokenizer against real schemas, verified by [an automated test suite](tests/test_token_savings.py).

### Per-turn cost

Every API turn, the native approach injects all tool schemas. mcp2cli injects a single 67-token instruction.

| Scenario | Native (tokens/turn) | mcp2cli (tokens/turn) | Reduction |
|---|--:|--:|--:|
| Small MCP server (3 tools) | 203 | 67 | **67%** |
| Petstore API (5 endpoints) | 358 | 67 | **81%** |
| Medium API (20 endpoints) | 1,430 | 67 | **95%** |
| Large API (50 endpoints) | 3,579 | 67 | **98%** |
| Enterprise API (200 endpoints) | 14,316 | 67 | **>99%** |

Native cost scales at ~72 tokens per endpoint. mcp2cli's cost is fixed.

### Over a full conversation

The savings compound. Here's the total token cost across a realistic multi-turn conversation, including the one-time discovery cost (`--list`) and tool call outputs:

| Scenario | Turns | Tool calls | Native total | mcp2cli total | Saved |
|---|--:|--:|--:|--:|--:|
| Petstore (5 endpoints) | 10 | 5 | 3,730 | 839 | **77%** |
| Medium API (20 endpoints) | 15 | 8 | 21,720 | 1,305 | **94%** |
| Large API (50 endpoints) | 20 | 12 | 71,940 | 1,850 | **97%** |
| Enterprise API (200 endpoints) | 25 | 15 | 358,425 | 2,725 | **99%** |

A 200-endpoint enterprise API over 25 turns: **355,700 tokens saved**.

### Turn-by-turn: watching the gap widen

Here's a 50-endpoint API over 10 turns. The native approach bleeds tokens on every turn; mcp2cli's cost barely moves.

```
Turn   Native       mcp2cli      Savings
──────────────────────────────────────────
1      3,579        217          3,362       ← mcp2cli: discovery (--list)
2      7,158        284          6,874
3      10,767       381          10,386      ← tool call
4      14,346       448          13,898
5      17,955       545          17,410      ← tool call
6      21,534       612          20,922
7      25,143       709          24,434      ← tool call
8      28,722       776          27,946
9      32,331       873          31,458      ← tool call
10     35,910       940          34,970

Total: 34,970 tokens saved (97.4%)
```

### Why the gap is so large

**Native approach** — pay the full schema tax on every turn:
```
System prompt: "You have these 50 tools: [3,579 tokens of JSON schemas]"
  → 3,579 tokens consumed per turn, whether used or not
  → 10 turns = 35,910 tokens
```

**mcp2cli approach** — pay only for what you use:
```
System prompt: "Use mcp2cli --spec <url> <command> [--flags]"   (67 tokens/turn)
  → mcp2cli --spec <url> --list                                (65 tokens, once)
  → mcp2cli --spec <url> create-pet --help                     (78 tokens, once)
  → mcp2cli --spec <url> create-pet --name Rex                 (0 extra tokens)
  → 10 turns = 940 tokens
```

The LLM discovers what it needs, when it needs it. Everything else stays out of context.

## Install

```bash
pip install mcp2cli

# With MCP support
pip install mcp2cli[mcp]
```

## Usage

### OpenAPI mode

```bash
# List all commands from a remote spec
mcp2cli --spec https://petstore3.swagger.io/api/v3/openapi.json --list

# Call an endpoint
mcp2cli --spec ./openapi.json --base-url https://api.example.com list-pets --status available

# With auth
mcp2cli --spec ./spec.json --auth-header "Authorization:Bearer tok_..." create-item --name "Test"

# POST with JSON body from stdin
echo '{"name": "Fido", "tag": "dog"}' | mcp2cli --spec ./spec.json create-pet --stdin

# Local YAML spec
mcp2cli --spec ./api.yaml --base-url http://localhost:8000 --list
```

### MCP stdio mode

```bash
# List tools from an MCP server
mcp2cli --mcp-stdio "npx @modelcontextprotocol/server-filesystem /tmp" --list

# Call a tool
mcp2cli --mcp-stdio "npx @modelcontextprotocol/server-filesystem /tmp" \
  read-file --path /tmp/hello.txt

# Pass environment variables to the server process
mcp2cli --mcp-stdio "node server.js" --env API_KEY=sk-... --env DEBUG=1 \
  search --query "test"
```

### MCP HTTP/SSE mode

```bash
# Connect to an MCP server over HTTP
mcp2cli --mcp https://mcp.example.com/sse --list

# With auth header
mcp2cli --mcp https://mcp.example.com/sse --auth-header "x-api-key:sk-..." \
  query --sql "SELECT 1"
```

### Output control

```bash
# Pretty-print JSON (also auto-enabled for TTY)
mcp2cli --spec ./spec.json --pretty list-pets

# Raw response body (no JSON parsing)
mcp2cli --spec ./spec.json --raw get-data

# Pipe-friendly (compact JSON when not a TTY)
mcp2cli --spec ./spec.json list-pets | jq '.[] | .name'
```

### Caching

Specs and MCP tool lists are cached in `~/.cache/mcp2cli/` with a 1-hour TTL by default.

```bash
# Force refresh
mcp2cli --spec https://api.example.com/spec.json --refresh --list

# Custom TTL (seconds)
mcp2cli --spec https://api.example.com/spec.json --cache-ttl 86400 --list

# Custom cache key
mcp2cli --spec https://api.example.com/spec.json --cache-key my-api --list

# Override cache directory
MCP2CLI_CACHE_DIR=/tmp/my-cache mcp2cli --spec ./spec.json --list
```

Local file specs are never cached.

## CLI reference

```
mcp2cli [global options] <subcommand> [command options]

Source (mutually exclusive, one required):
  --spec URL|FILE       OpenAPI spec (JSON or YAML, local or remote)
  --mcp URL             MCP server URL (HTTP/SSE)
  --mcp-stdio CMD       MCP server command (stdio transport)

Options:
  --auth-header K:V     HTTP header (repeatable)
  --base-url URL        Override base URL from spec
  --env KEY=VALUE       Env var for MCP stdio server (repeatable)
  --cache-key KEY       Custom cache key
  --cache-ttl SECONDS   Cache TTL (default: 3600)
  --refresh             Bypass cache
  --list                List available subcommands
  --pretty              Pretty-print JSON output
  --raw                 Print raw response body
  --version             Show version
```

Subcommands and their flags are generated dynamically from the spec or MCP server tool definitions. Run `<subcommand> --help` for details.

## How it works

1. **Load** -- Fetch the OpenAPI spec or connect to the MCP server. Resolve `$ref`s. Cache for reuse.
2. **Extract** -- Walk the spec paths/tools and produce a uniform list of command definitions with typed parameters.
3. **Build** -- Generate an argparse parser with subcommands, flags, types, choices, and help text.
4. **Execute** -- Dispatch the parsed args as an HTTP request (OpenAPI) or tool call (MCP).

Both adapters produce the same internal `CommandDef` structure, so the CLI builder and output handling are shared.

## Development

```bash
# Install with test + MCP deps
uv sync --extra test --extra mcp

# Run tests (96 tests covering OpenAPI, MCP stdio, MCP HTTP, caching, and token savings)
uv run pytest tests/ -v

# Run just the token savings tests
uv run pytest tests/test_token_savings.py -v -s
```

## Acknowledgments

This project was inspired by [Kagan Yilmaz's analysis of CLI vs MCP token costs](https://kanyilmaz.me/2026/02/23/cli-vs-mcp.html) and his work on [CLIHub](https://kanyilmaz.me/2026/02/23/cli-vs-mcp.html). His observation that CLI-based tool access is dramatically more token-efficient than native MCP injection was the spark for mcp2cli. Where CLIHub generates static CLIs from MCP servers, mcp2cli takes a different approach: it reads schemas at runtime, so there's no codegen step and no rebuild when the server adds or changes tools. It also extends the pattern to OpenAPI specs — any REST API with a spec file gets the same treatment.

## License

MIT
