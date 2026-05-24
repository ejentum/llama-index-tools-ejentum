# llama-index-tools-ejentum

A [LlamaIndex](https://www.llamaindex.ai) tool spec that wraps the hosted [Ejentum](https://ejentum.com) MCP server and exposes its four cognitive-harness tools (`harness_reasoning`, `harness_code`, `harness_anti_deception`, `harness_memory`) as LlamaIndex `FunctionTool` objects. The agent calls one before generating; each call returns a structured scaffold the agent absorbs to harden its next response against a named failure mode.

Each operation in the Ejentum library (679 of them, organized across four harnesses) is engineered in **two layers**:

- a **natural-language procedure** the model can read, naming the steps to take and the failure pattern to refuse, and
- an **executable reasoning topology**: a graph-shaped plan over those steps. The plan names explicit decision points where the model branches, parallel branches that run and rejoin, bounded loops that run until convergence, named meta-cognitive moments where the model is asked to stop, look at its own working, and re-enter at a specific step, plus escape paths for when the prescribed plan stops fitting the task at hand.

The natural-language layer tells the model *what* to do. The topology layer pins down *how* those steps connect: where to decide, where to loop, where to stop and look at itself. Together they act as a persistent attention anchor that survives long context windows and multi-turn execution chains, which is precisely where a model's own reasoning template typically decays.

## Installation

```bash
pip install llama-index-tools-ejentum
```

## Configuration

Get an Ejentum API key at <https://ejentum.com/pricing> (free and paid tiers) and set it in your environment:

```bash
export EJENTUM_API_KEY="zpka_..."
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
# Only expose reasoning and code harnesses
spec = EjentumToolSpec(modes=["reasoning", "code"])
tools = spec.to_tool_list()
```

Valid mode names: `reasoning`, `code`, `anti_deception`, `memory`.

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

The agent routes to `harness_reasoning` based on the tool description. The returned scaffold is fed back into the agent's context for the next generation step.

## The four harnesses

| Tool | Best for | Library size |
|---|---|---|
| `harness_reasoning` | Analytical, diagnostic, planning, multi-step tasks spanning abstraction, time, causality, simulation, spatial, and metacognition | 311 operations |
| `harness_code` | Code generation, refactoring, review, and debugging across the software-engineering layer | 128 operations |
| `harness_anti_deception` | Prompts that pressure the agent to validate, certify, or soften an honest assessment, spanning sycophancy, hallucination, deception, adversarial framing, judgment, and executive control | 139 operations |
| `harness_memory` | Sharpening an observation already formed about cross-turn drift across the perception layer. Filter-oriented, not write-oriented. Format `query` as `"I noticed X. This might mean Y. Sharpen: Z."` | 101 operations |

## What an injection looks like

A real `reasoning` mode response on the query `investigate why our nightly ETL job has started failing intermittently over the past two weeks; nothing in the code or schema has changed`:

```
[NEGATIVE GATE]
The server's response time was accepted as average, despite a suspicious
rhythm break in its timing pattern.

[PROCEDURE]
Step 1: Establish baseline timing profiles by extracting historical
durations and intervals for each event type. Step 2: Compare each observed
timing against its baseline and compute deviation magnitude. Step 3:
Classify anomalies as too fast, too slow, too early, or too late, and rank
by severity. ... Step 5: If deviation exceeds two standard deviations,
probe root cause by tracing upstream dependencies. ...

[REASONING TOPOLOGY]
S1:durations -> FIXED_POINT[baselines] -> N{dismiss_timing_deviations_
without_investigation} -> for_each: S2:compare -> S3:deviation ->
G1{>2sigma?} --yes-> S4:classify -> S5:probe_cause -> FLAG -> continue --no->
S6:validate -> continue -> all_checked -> OUT:anomaly_report

[TARGET PATTERN]
Establish timing baselines by extracting historical response intervals.
Compare current server response time to this baseline. ...

[FALSIFICATION TEST]
If no event timing is flagged as suspiciously fast or slow relative to
baseline, temporal anomaly detection was not active.

Amplify: timing baseline comparison; anomaly classification; security
context elevation
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
| `modes` | `None` | Optional subset of harness modes to expose. Names without the `harness_` prefix, e.g. `["reasoning", "code"]`. Defaults to all four. |
| `api_url` | `https://api.ejentum.com/mcp` | Override only if you self-host the Ejentum MCP gateway. |
| `timeout` | `30` | HTTP timeout in seconds for the underlying MCP client. |

This class is a thin subclass of `llama_index.tools.mcp.McpToolSpec`, pre-configured with the hosted Ejentum endpoint and Bearer authentication. For raw MCP usage against arbitrary servers, use `McpToolSpec` directly.

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


## Measured effects

The Ejentum harness is benchmarked publicly under CC BY 4.0 at [github.com/ejentum/benchmarks](https://github.com/ejentum/benchmarks):

- **ELEPHANT** sycophancy: 5.8% composite on GPT-4o (40 real Reddit scenarios)
- **LiveCodeBench Hard**: 85.7% to 100% on Claude Opus (28 competitive programming tasks)
- **Memory retention**: 50% fewer stale facts served (20-turn implicit state changes)
- Plus per-harness numbers across BBH/CausalBench/MuSR, ARC-AGI-3, SciCode, and perception tasks

Methodology, scenarios, run scripts, and raw outputs are all in-repo.
