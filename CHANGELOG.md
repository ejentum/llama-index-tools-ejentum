# Changelog

All notable changes to `llama-index-tools-ejentum` are documented here. This project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-05-22

### Added

- Initial release.
- `EjentumToolSpec` subclasses `llama_index.tools.mcp.McpToolSpec` and pre-configures the hosted Ejentum MCP server at `https://api.ejentum.com/mcp` with Bearer authentication via the `EJENTUM_API_KEY` environment variable.
- Four harness tools exposed: `harness_reasoning`, `harness_code`, `harness_anti_deception`, `harness_memory`.
- Mode-subset shorthand via the `modes=` constructor argument (e.g. `modes=["reasoning", "code"]` limits the agent to two of the four harnesses).
- `api_url` and `timeout` constructor arguments for self-hosted deployments.
- Construction-time validation: raises `ValueError` if `EJENTUM_API_KEY` is unset, raises `ValueError` for unknown mode names.
- Eight unit tests cover the public surface: subclass identity, explicit-key construction, env-var fallback, missing-key error, mode subset filtering, unknown-mode error, custom URL + timeout, and the SUPPORTED_MODES constant.
- Published to PyPI with OIDC trusted-publisher provenance attestation via GitHub Actions.

### Background

This package is the standalone-PyPI replacement for the closed [run-llama/llama_index#21757](https://github.com/run-llama/llama_index/pull/21757) PR. LlamaHub stopped accepting new integration packages into the monorepo; PyPI remains the canonical distribution surface, and the LlamaHub naming convention `llama-index-tools-<name>` is preserved so the import path `from llama_index.tools.ejentum import EjentumToolSpec` matches every other LlamaIndex tool.
