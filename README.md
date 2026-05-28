# llama-index-tools-ejentum

A [LlamaIndex](https://www.llamaindex.ai) tool spec that wraps the hosted [Ejentum](https://ejentum.com) MCP server and exposes its eight cognitive-harness tools as LlamaIndex `FunctionTool` objects. The agent calls one before generating; each call returns a structured injection the agent absorbs to harden its next response against a named failure mode.

Four dynamic tools (`reasoning`, `code`, `anti-deception`, `memory`) are available on all tiers including the 30-day free trial. Four adaptive tools (`adaptive-reasoning`, `adaptive-code`, `adaptive-anti-deception`, `adaptive-memory`) additionally rewrite the procedure and topology DAG to fit your specific task via an adapter LLM, and require the Go or Super tier.

Each operation in the Ejentum library (679 of them, organized across four cognitive harnesses each with dynamic and adaptive variants) is engineered in **two layers**:

- a **natural-language procedure** the model can read, naming the steps to take and the failure pattern to refuse, and
- an **executable reasoning topology**: a graph-shaped plan over those steps. The plan names explicit decision points where the model branches, parallel branches that run and rejoin, bounded loops that run until convergence, named meta-cognitive moments where the model is asked to stop, look at its own working, and re-enter at a specific step, plus escape paths for when the prescribed plan stops fitting the task at hand.

The natural-language layer tells the model *what* to do. The topology layer pins down *how* those steps connect: where to decide, where to loop, where to stop and look at itself. Together they act as a persistent attention anchor that survives long context windows and multi-turn execution chains, which is precisely where a model's own reasoning template typically decays.

## Installation

```bash
pip install llama-index-tools-ejentum
```

## Configuration

Get an Ejentum API key at <https://ejentum.com/pricing>. The 30-day free trial (no card required) includes 1,000 dynamic reasoning calls; adaptive tools require Go or Super.

```bash
export EJENTUM_API_KEY="ej_..."
```

## Usage

### Minimal

```python
from llama_index.tools.ejentum import EjentumToolSpec

spec = EjentumToolSpec()  # reads EJENTUM_API_KEY from env
tools = spec.to_tool_list()
```

### Subset of modes

```python
# Only expose reasoning and code (dynamic + adaptive variants of those modes)
spec = EjentumToolSpec(modes=["reasoning", "code", "adaptive-reasoning", "adaptive-code"])
tools = spec.to_tool_list()
```

Valid mode names: `reasoning`, `code`, `anti-deception`, `memory`, `adaptive-reasoning`, `adaptive-code`, `adaptive-anti-deception`, `adaptive-memory`.

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

The agent routes to `reasoning` based on the tool description. The returned injection is fed back into the agent's context for the next generation step.

## The eight harnesses

### Dynamic (single retrieval, all tiers including the 30-day free trial)

| Tool | Best for | Library size |
|---|---|---|
| `reasoning` | Analytical, diagnostic, planning, multi-step tasks spanning abstraction, time, causality, simulation, spatial, and metacognition | 311 operations |
| `code` | Code generation, refactoring, review, and debugging across the software-engineering layer | 128 operations |
| `anti-deception` | Prompts that pressure the agent to validate, certify, or soften an honest assessment, spanning sycophancy, hallucination, deception, adversarial framing, judgment, and executive control | 139 operations |
| `memory` | Sharpening an observation already formed about cross-turn drift across the perception layer. Filter-oriented, not write-oriented. Format `query` as `"I noticed X. This might mean Y. Sharpen: Z."` | 101 operations |

### Adaptive (top-k retrieval + adapter LLM rewrites operation to fit the task; Go or Super tier required)

| Tool | When to prefer over the dynamic version |
|---|---|
| `adaptive-reasoning` | High-stakes analytical work where every DAG node should be mapped to your specifics before generation. Cost ~2-3s vs ~1s. |
| `adaptive-code` | Security-critical reviews, refactor-heavy diffs, or any code work where every verification step should be concretized. |
| `adaptive-anti-deception` | When the stakes of a soft or sycophantic answer are high. |
| `adaptive-memory` | When the dynamic memory tool's general scaffold is not sharp enough for the perception being formed. |

## What an injection looks like

A real `reasoning` mode response on the query `investigate why our nightly ETL job has started failing intermittently over the past two weeks; nothing in the code or schema has changed`:

```
[NEGATIVE GATE]
The server's response time was accepted as average, despite a suspicious
rhythm break in its timing pattern.

[PROCEDURE]
Step 1: Establish baseline timing profiles by extracting historical
durations and intervals for each event type. Step 2: Compare each observed
timing against its baseline and compute deviation magnitude. ...

[REASONING TOPOLOGY]
S1:durations -> FIXED_POINT[baselines] -> N{dismiss_timing_deviations_
without_investigation} -> for_each: S2:compare -> S3:deviation ->
G1{>2sigma?} --yes-> S4:classify -> S5:probe_cause -> FLAG -> continue --no->
S6:validate -> continue -> all_checked -> OUT:anomaly_report

[FALSIFICATION TEST]
If no event timing is flagged as suspiciously fast or slow relative to
baseline, temporal anomaly detection was not active.

Amplify: timing baseline comparison; anomaly classification
Suppress: average timing acceptance; outlier normalization
```

The agent reads both the natural-language `[PROCEDURE]` and the graph-logic `[REASONING TOPOLOGY]` before generating its user-facing answer. The bracketed labels are instructions to the agent, not content to display; the user sees a naturally-phrased answer shaped by the injection.

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
| `api_key` | `None` | If omitted, read from `EJENTUM_API_KEY`. Raises `ValueError` at construction time if neither is set. |
| `modes` | `None` | Optional subset of harness modes to expose. Use canonical names with hyphens, e.g. `["reasoning", "adaptive-anti-deception"]`. Defaults to all eight. |
| `api_url` | `https://api.ejentum.com/mcp` | Override only if you self-host the Ejentum MCP gateway. |
| `timeout` | `30` | HTTP timeout in seconds for the underlying MCP client. |

This class is a thin subclass of `llama_index.tools.mcp.McpToolSpec`, pre-configured with the hosted Ejentum endpoint and Bearer authentication. The tool names exposed to the LLM are whatever the upstream MCP server advertises (`reasoning`, `code`, `anti-deception`, `memory`, `adaptive-reasoning`, `adaptive-code`, `adaptive-anti-deception`, `adaptive-memory`), so this shim does not translate names. For raw MCP usage against arbitrary servers, use `McpToolSpec` directly.

## About the underlying MCP server

The same MCP server is available across other surfaces: stdio via `npx -y ejentum-mcp`, hosted at `https://api.ejentum.com/mcp` (Streamable HTTP), and listed on the [Official MCP Registry](https://registry.modelcontextprotocol.io/) as `io.github.ejentum/ejentum-mcp`.

## Compatibility

- Python 3.10+
- `llama-index-core>=0.13.0,<0.15`
- `llama-index-tools-mcp>=0.4.1,<0.5`

## Resources

- Ejentum homepage: <https://ejentum.com>
- Pricing: <https://ejentum.com/pricing>
- API reference: <https://ejentum.com/docs/api_reference>
- "Why LLM Agents Fail" essay: <https://ejentum.com/blog/why-llm-agents-fail>
- "Under Pressure" research paper: <https://doi.org/10.5281/zenodo.19392715>
- MCP source repository: <https://github.com/ejentum/ejentum-mcp>
- LlamaIndex documentation: <https://docs.llamaindex.ai>

## License

[MIT](./LICENSE)
