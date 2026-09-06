# ResearchOps MCP Release Checklist

## Release Scope

Date: 2026-09-06

Release target: portfolio-ready ResearchOps MCP project.

This release packages the completed 14-day roadmap into a demonstrable MCP server with local and remote transports, persistence, authentication, security controls, reliability behavior, evaluation gates, observability, CI, Docker, and deployment notes.

## Supported MCP Surface

- Tools: `health_check`, `search_papers`, `get_paper`, `export_bibtex`, `create_reading_list`, `add_paper_to_list`, `add_note`, `update_note`, `delete_note`.
- Resources: `paper://{paper_id}`, `reading-list://{list_id}`.
- Prompts: `compare_papers`, `generate_literature_review`.
- Transports: local `stdio` and Streamable HTTP.
- Auth mode: demo bearer-token auth with scopes and ownership enforcement for protected HTTP deployments.
- Evaluation: deterministic 42-case MCP selection and argument regression harness.
- Observability: JSON operation logs, request IDs, local metrics, and OpenTelemetry API spans.

## Final Verification Commands

Run these before tagging or deploying a release:

```powershell
python -m compileall src tests
pytest
python -m researchops_mcp.evals --fail-on-thresholds
docker build -t researchops-mcp:release .
```

Optional MCP-visible checks:

```powershell
python client/cli.py discover
python client/cli.py list-tools
python client/cli.py call-tool health_check
npx @modelcontextprotocol/inspector@latest --cli python src/server.py --method tools/list --format json
```

## Security Review

- Authentication is available for Streamable HTTP through demo bearer tokens.
- Scopes are enforced for paper reads, list reads/writes, and note writes.
- Ownership checks are enforced below the MCP handler at the repository layer.
- Write tools require explicit idempotency keys and use audit records.
- Note updates and deletes use optimistic concurrency.
- Delete requires explicit confirmation.
- External OpenAlex calls are domain-allowlisted.
- HTTP request bodies are size-limited.
- Authenticated HTTP requests are rate-limited in process.
- Model-facing resources and prompts label external or user-provided content as untrusted.
- Logs redact sensitive fields before serialization.

## Known Limitations

- Demo bearer tokens are not a production OAuth provider.
- Render staging currently uses disposable `/tmp` SQLite storage.
- Rate limiting, circuit breaking, and observability metrics are process-local.
- OpenTelemetry spans are instrumented through the API, but no SDK exporter or collector is configured.
- `/metrics` returns JSON rather than Prometheus exposition format.
- `/readyz` is a shallow readiness endpoint and does not perform deep dependency probes.
- Search-result caching is intentionally absent; only stable paper lookups have stale-cache fallback.
- Long-running Tasks are documented as a future extension rather than implemented in this release.
- MCP Apps/UI resources are documented as a future extension rather than implemented in this release.

## Compatibility Policy

- Treat `tools/list`, resource templates, prompt names, descriptions, and input schemas as public contract.
- Do not rename a tool, resource URI pattern, prompt, required argument, or output field without a migration note.
- Additive optional fields are preferred over breaking schema changes.
- Breaking changes should use a new tool name or versioned capability while the older interface remains available for a migration window.
- Tool-list cache TTLs should be conservative when metadata is still evolving.
- Deprecated capabilities should remain documented with replacement guidance until removed.

## Release Decision

ResearchOps MCP is portfolio-ready as a local/staging production candidate, not a fully managed enterprise service.

The final release should present the project honestly: the MCP interface, tests, security posture, reliability controls, evals, observability, and deployment path are demonstrable, while full OAuth provider integration, durable production database hosting, external telemetry backends, and Tasks/Apps extensions remain clearly documented future work.