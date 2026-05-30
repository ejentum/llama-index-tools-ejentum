# llama-index-tools-ejentum

[LlamaIndex](https://www.llamaindex.ai) tool spec that subclasses `McpToolSpec` and points at the hosted Ejentum MCP server. Exposes the eight cognitive-harness tools as LlamaIndex `FunctionTool` objects ready for any LlamaIndex agent or query engine.

Use the harness before the agent generates on complex, multi-step, or multi-constraint tasks where the model's default reasoning template would miss a constraint, take a shortcut, or drift across turns. Each call returns a *cognitive operation*: a structured procedure (numbered steps with a failure pattern to refuse and a falsification test) paired with an executable reasoning topology (a DAG of those steps with decision gates, parallel branches, bounded loops, and meta-cognitive exit nodes). The agent reads both layers before producing its response.

Four dynamic tools (`reasoning`, `code`, `anti-deception`, `memory`) are available on all tiers including the 30-day free trial. Four adaptive tools (`adaptive-reasoning`, `adaptive-code`, `adaptive-anti-deception`, `adaptive-memory`) additionally run an adapter LLM that rewrites the matched operation with task-specific identifiers; they require the Go or Super tier.

Tool names exposed to the LLM are whatever the upstream MCP server advertises (canonical hyphenated strings: `reasoning`, `code`, `anti-deception`, `memory`, `adaptive-reasoning`, `adaptive-code`, `adaptive-anti-deception`, `adaptive-memory`). This shim does not rename them.

## Install

```bash
pip install llama-index-tools-ejentum
```

## Configuration

```bash
export EJENTUM_API_KEY="ej_..."
```

Or pass `api_key=` to `EjentumToolSpec(...)`. Get a key at [ejentum.com/pricing](https://ejentum.com/pricing).

## Usage

### Minimal

```python
from llama_index.tools.ejentum import EjentumToolSpec

spec = EjentumToolSpec()
tools = spec.to_tool_list()
```

### Subset of modes

```python
spec = EjentumToolSpec(modes=["reasoning", "code", "adaptive-reasoning", "adaptive-code"])
tools = spec.to_tool_list()
```

Valid mode names (use canonical hyphenated form): `reasoning`, `code`, `anti-deception`, `memory`, `adaptive-reasoning`, `adaptive-code`, `adaptive-anti-deception`, `adaptive-memory`.

### With a ReActAgent

```python
from llama_index.core.agent import ReActAgent
from llama_index.llms.openai import OpenAI
from llama_index.tools.ejentum import EjentumToolSpec

tools = EjentumToolSpec().to_tool_list()
agent = ReActAgent.from_tools(tools, llm=OpenAI(model="gpt-4o-mini"))

response = await agent.achat(
    "Why might our microservice return 503s only under specific load patterns?"
)
```

## Tool inventory

### Dynamic (all tiers)

| Tool | Library size |
|---|---:|
| `reasoning` | 311 |
| `code` | 128 |
| `anti-deception` | 139 |
| `memory` | 101 |

### Adaptive (Go or Super tier)

| Tool |
|---|
| `adaptive-reasoning` |
| `adaptive-code` |
| `adaptive-anti-deception` |
| `adaptive-memory` |

Each tool takes a single `query: str` argument. Returns the injection as a string.

## API reference

```python
EjentumToolSpec(
    api_key: str | None = None,
    modes: list[str] | None = None,
    api_url: str = "https://api.ejentum.com/mcp",
    timeout: int = 30,
)
```

| Field | Default | Description |
|---|---|---|
| `api_key` | `None` | If unset, read from `EJENTUM_API_KEY`. Raises `ValueError` at construction if neither is set. |
| `modes` | `None` | Optional subset of harness modes to expose. Defaults to all eight. |
| `api_url` | `https://api.ejentum.com/mcp` | Override for self-hosted MCP gateway. |
| `timeout` | `30` | HTTP timeout in seconds for the underlying MCP client. |

The class is a thin subclass of `llama_index.tools.mcp.McpToolSpec`, pre-configured with the hosted Ejentum endpoint and Bearer authentication.

## Wire contract

This shim talks to the **MCP** endpoint (`/mcp`), not the direct-REST endpoint (`/harness/`). For the direct-REST contract used by every other Ejentum shim, see the [ejentum-mcp README](https://github.com/ejentum/ejentum-mcp#wire-contract); for the MCP-over-streamable-HTTP contract, see the [MCP specification](https://modelcontextprotocol.io).

Field structure of an injection and a canonical dynamic-vs-adaptive comparison on the same query are documented in the [ejentum-mcp README](https://github.com/ejentum/ejentum-mcp#canonical-example-dynamic-vs-adaptive-on-the-same-query).

## The underlying MCP server

The same MCP server is available on three additional surfaces:

- Stdio via `npx -y ejentum-mcp`
- Hosted Streamable HTTP at `https://api.ejentum.com/mcp`
- Listed on the [Official MCP Registry](https://registry.modelcontextprotocol.io/) as `io.github.ejentum/ejentum-mcp`

## Compatibility

- Python 3.10+
- `llama-index-core>=0.13.0,<0.15`
- `llama-index-tools-mcp>=0.4.1,<0.5`

## License

[MIT](./LICENSE)
