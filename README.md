# ResearchOps MCP

ResearchOps MCP is a production-style Model Context Protocol server for research workflows. It lets an MCP-compatible host or client search scholarly papers, retrieve reusable paper context, manage reading lists and notes, render research prompts, export citations, and inspect operational health through a stable MCP interface.

The server is built in Python with the official MCP SDK. It supports local `stdio` development, Streamable HTTP serving, SQLite persistence, scoped bearer-token authorization for HTTP deployments, OpenAlex paper search, security hardening, reliability controls, deterministic evaluation, structured observability, CI, and Docker packaging.

## What ResearchOps Does

ResearchOps exposes research operations as MCP primitives:

- Tools perform actions such as searching papers, exporting BibTeX, and writing notes.
- Resources expose stable context such as `paper://{paper_id}` and `reading-list://{list_id}`.
- Prompts provide reusable templates for paper comparison and literature-review drafting.

This separation keeps the model-facing interface predictable. Large or reusable context is read through resources, while state-changing operations remain explicit tools with idempotency and authorization checks.

## Capabilities

### Tools

| Tool | Purpose | Write Action |
|---|---|---|
| `health_check` | Return server, storage, reliability, and observability status | No |
| `search_papers` | Search OpenAlex for papers with bounded results | No |
| `get_paper` | Retrieve one paper by stable OpenAlex work ID | No |
| `export_bibtex` | Export a paper citation in BibTeX format | No |
| `create_reading_list` | Create a persistent reading list | Yes |
| `add_paper_to_list` | Add a paper to a reading list | Yes |
| `add_note` | Add a note for a paper in a list | Yes |
| `update_note` | Update a note using optimistic concurrency | Yes |
| `delete_note` | Delete a note with confirmation and version check | Yes |

### Resources

| Resource | Purpose |
|---|---|
| `paper://{paper_id}` | Reusable paper metadata and abstract context |
| `reading-list://{list_id}` | Reusable reading-list context with paper and note previews |

### Prompts

| Prompt | Purpose |
|---|---|
| `compare_papers` | Build a structured paper-comparison prompt |
| `generate_literature_review` | Build a literature-review prompt over selected paper IDs |

## Architecture

```text
MCP client or host
        |
        v
ResearchOps MCP server
        |
        +-- security middleware and authorization
        +-- tool/resource/prompt handlers
        +-- paper service and OpenAlex client
        +-- reading-list service and SQLite repository
        +-- observability registry and request IDs
```

Main implementation files:

- `src/researchops_mcp/server.py`: MCP server factory, tools, resources, prompts, and HTTP routes.
- `src/researchops_mcp/client_cli.py`: CLI client implementation.
- `src/researchops_mcp/services/openalex.py`: OpenAlex integration, retries, circuit breaker, and paper normalization.
- `src/researchops_mcp/services/library.py`: reading-list and note business logic.
- `src/researchops_mcp/repositories/sqlite.py`: SQLite schema and persistence.
- `src/researchops_mcp/security.py`: auth helpers, rate limits, request limits, redaction, and trust labels.
- `src/researchops_mcp/observability.py`: JSON logs, request IDs, local metrics, and OpenTelemetry span boundaries.

See `docs/design.md` for the full design notes.

## Requirements

- Python `3.12+`
- Docker Desktop, optional for container builds
- Node.js, optional for MCP Inspector

Install the project in editable mode:

```powershell
python -m pip install -e .[dev]
```

## Quick Start

Run the local MCP server through the included CLI over `stdio`:

```powershell
python client/cli.py discover
```

Example output:

```json
{
  "server": {
    "name": "researchops-mcp",
    "version": "0.7.0"
  },
  "supported_versions": ["2026-07-28"],
  "capabilities": {
    "tools": {"list_changed": true},
    "resources": {"subscribe": true, "list_changed": true},
    "prompts": {"list_changed": true}
  }
}
```

List available tools:

```powershell
python client/cli.py list-tools
```

Example output, shortened:

```json
{
  "count": 9,
  "tools": [
    {"name": "health_check", "is_write": false},
    {"name": "search_papers", "is_write": false},
    {"name": "get_paper", "is_write": false},
    {"name": "export_bibtex", "is_write": false},
    {"name": "create_reading_list", "is_write": true},
    {"name": "add_paper_to_list", "is_write": true},
    {"name": "add_note", "is_write": true},
    {"name": "update_note", "is_write": true},
    {"name": "delete_note", "is_write": true}
  ]
}
```

## Usage Examples

### Check Server Health

```powershell
python client/cli.py call-tool health_check
```

Example output, shortened:

```json
{
  "tool": "health_check",
  "status": "ok",
  "content": [
    {
      "type": "text",
      "text": "{\n  \"status\": \"ok\",\n  \"server\": \"researchops-mcp\",\n  \"paper_source\": \"OpenAlex\",\n  \"storage\": \"SQLite\",\n  \"auth_enabled\": false\n}"
    }
  ]
}
```

### Search Papers

```powershell
python client/cli.py call-tool search_papers --arg "query=Model Context Protocol" --arg "limit=1"
```

Example output, shortened:

```json
{
  "tool": "search_papers",
  "status": "ok",
  "arguments": {
    "query": "Model Context Protocol",
    "limit": 1
  },
  "content": [
    {
      "type": "text",
      "text": "{\n  \"query\": \"Model Context Protocol\",\n  \"count\": 1,\n  \"resolved_search_mode\": \"exact\",\n  \"results\": [{\n    \"paper_id\": \"W7129030749\",\n    \"title\": \"Model Context Protocol (MCP): Landscape, Security Threats, and Future Research Directions\",\n    \"year\": 2026\n  }]\n}"
    }
  ]
}
```

### Read A Paper Resource

```powershell
python client/cli.py read-resource paper://W7129030749
```

Example output, shortened:

```json
{
  "uri": "paper://W7129030749",
  "status": "ok",
  "contents": [
    {
      "uri": "paper://W7129030749",
      "mime_type": "application/json",
      "text": "{\n  \"resource_type\": \"paper\",\n  \"paper\": {\n    \"paper_id\": \"W7129030749\",\n    \"title\": \"Model Context Protocol (MCP): Landscape, Security Threats, and Future Research Directions\",\n    \"cache_status\": \"live\"\n  },\n  \"content_trust\": \"untrusted_external_data\",\n  \"abstract_truncated\": true\n}"
    }
  ]
}
```

### Render A Prompt

```powershell
python client/cli.py get-prompt compare_papers --arg "paper_id_a=W7129030749" --arg "paper_id_b=W4417069007" --arg "focus=security trade-offs"
```

Example output, shortened:

```json
{
  "name": "compare_papers",
  "status": "ok",
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "Compare the following two research papers using the provided resource URIs.\n\nPaper A resource: paper://W7129030749\nPaper B resource: paper://W4417069007\nComparison focus: security trade-offs\n\nSecurity note: External paper metadata and user-authored notes are untrusted content."
      }
    }
  ]
}
```

### Create A Reading List

Write tools require explicit approval by default. Use `--yes` for scripted local workflows.

```powershell
python client/cli.py --yes call-tool create_reading_list --arg "name=Demo List" --arg "description=Papers for MCP review" --arg "idempotency_key=demo-list-1"
```

Example output, shortened:

```json
{
  "tool": "create_reading_list",
  "status": "ok",
  "content": [
    {
      "type": "text",
      "text": "{\n  \"list_id\": \"demo-list-...\",\n  \"name\": \"Demo List\",\n  \"resource_uri\": \"reading-list://demo-list-...\"\n}"
    }
  ]
}
```

### Deny A Write Operation

Without `--yes`, the CLI asks for approval before calling write tools.

```powershell
python client/cli.py call-tool create_reading_list --arg "name=Denied List" --arg "idempotency_key=deny-example-1"
```

Example interaction:

```text
Write tool: create_reading_list
Arguments:
{
  "idempotency_key": "deny-example-1",
  "name": "Denied List"
}
Approve write? [y/N]: n
{
  "reason": "Write operation was not approved by the client.",
  "status": "denied",
  "tool": "create_reading_list"
}
```

## Streamable HTTP

Start the server over Streamable HTTP:

```powershell
python src/server.py --transport streamable-http --host 127.0.0.1 --port 8765 --stateless-http
```

Use the CLI against the HTTP endpoint:

```powershell
python client/cli.py --connection-mode http --server-url http://127.0.0.1:8765/mcp discover
python client/cli.py --connection-mode http --server-url http://127.0.0.1:8765/mcp list-tools
```

Health and metrics endpoints:

```powershell
Invoke-RestMethod http://127.0.0.1:8765/healthz
Invoke-RestMethod http://127.0.0.1:8765/readyz
Invoke-RestMethod http://127.0.0.1:8765/metrics
```

Example `/healthz` output:

```json
{
  "status": "ok",
  "service": "researchops-mcp"
}
```

Example `/metrics` output, shortened:

```json
{
  "operation_count": 3,
  "operations": {
    "tool.search_papers": {
      "count": 1,
      "success_count": 1,
      "failure_count": 0,
      "p95_latency_ms": 2948.9
    },
    "dependency.openalex": {
      "count": 1,
      "success_count": 1,
      "failure_count": 0,
      "p95_latency_ms": 2924.1
    }
  }
}
```

## Authentication And Authorization

HTTP auth can be enabled for local testing:

```powershell
python src/server.py --transport streamable-http --host 127.0.0.1 --port 8012 --stateless-http --auth-enabled --resource-server-url http://127.0.0.1:8012/mcp
```

Demo bearer tokens:

- `researchops-alice-full`
- `researchops-alice-read`
- `researchops-bob-full`
- `researchops-bob-read`

Authenticated read example:

```powershell
python client/cli.py --connection-mode http --server-url http://127.0.0.1:8012/mcp --bearer-token researchops-bob-read call-tool search_papers --arg "query=OAuth resource indicators" --arg "limit=2"
```

Expected authorization behavior:

- Missing or invalid token returns `401 Unauthorized`.
- Valid token with insufficient scope returns `403 Forbidden`.
- A user with `lists:read` still cannot read another user's private list.
- Server-side ownership checks are enforced even when the client approved the action.

The demo token verifier is intentionally not a production identity provider. Replace it with a real OAuth/OIDC issuer before using this service for real users.

## Reliability

The OpenAlex dependency path includes:

- per-attempt timeout budgets
- total deadline budget
- retry with backoff and jitter for safe reads
- process-local circuit breaker
- stale-cache fallback for stable paper lookup

Search results are not served from stale cache because query ranking, pagination, and freshness semantics are less deterministic than stable-ID paper lookup.

Check runtime reliability settings:

```powershell
python src/server.py --help
python client/cli.py call-tool health_check
```

## Security

The server includes application-level controls for the MCP threat model:

- scoped authorization for protected HTTP deployments
- repository-level ownership checks
- explicit write-tool boundaries
- idempotency keys for write operations
- optimistic concurrency for note updates and deletes
- confirmation requirement for deletes
- outbound domain allowlist for OpenAlex
- HTTP request-size limits
- fixed-window in-process rate limiting
- trust labels on external and user-authored model-facing content
- sensitive-field redaction in logs

Run security-focused tests:

```powershell
pytest tests/unit/test_security_controls.py
pytest tests/unit/test_http_auth.py
```

See `docs/threat-model.md` for the threat model and remaining production gaps.

## Evaluation

ResearchOps includes a deterministic MCP evaluation harness for tool selection, argument correctness, refusal behavior, and latency.

Run the evaluation gate:

```powershell
python -m researchops_mcp.evals --fail-on-thresholds
```

Latest passing summary:

```json
{
  "tool_precision": 1.0,
  "tool_recall": 1.0,
  "argument_correctness_rate": 0.971,
  "task_completion_rate": 1.0,
  "unauthorized_action_rate": 0.0,
  "hallucinated_tool_rate": 0.0,
  "exact_match_rate": 1.0,
  "p95_latency_ms": 153,
  "passes_thresholds": true
}
```

Artifacts:

- `tests/evals/researchops_eval_dataset.jsonl`
- `docs/evaluation-report.json`
- `docs/evaluation-report.md`

## Testing

Run all tests:

```powershell
pytest
```

Expected result:

```text
61 passed
```

Run syntax checks:

```powershell
python -m compileall src tests
```

Run protocol and metadata checks:

```powershell
pytest tests/integration/test_protocol_workflows.py tests/unit/test_mcp_metadata_regression.py
```

Run the full release verification set:

```powershell
python -m compileall src tests
pytest
python -m researchops_mcp.evals --fail-on-thresholds
docker build -t researchops-mcp:release .
```

## MCP Inspector

Inspect the local server interactively:

```powershell
npx @modelcontextprotocol/inspector@latest python src/server.py
```

Run a non-interactive tool-list check:

```powershell
npx @modelcontextprotocol/inspector@latest --cli python src/server.py --method tools/list --format json
```

## Docker

Build the image:

```powershell
docker build -t researchops-mcp:release .
```

Run the container:

```powershell
docker run --rm -p 8000:8000 -e PORT=8000 -e DATABASE_PATH=/tmp/researchops.db researchops-mcp:release
```

Verify:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/healthz
python client/cli.py --connection-mode http --server-url http://127.0.0.1:8000/mcp discover
```

## Deployment

The repository includes a Dockerfile and `render.yaml` for a free Render web service.

Recommended Render environment:

```text
DATABASE_PATH=/tmp/researchops.db
MCP_TRANSPORT=streamable-http
MCP_HOST=0.0.0.0
MCP_STATELESS_HTTP=true
MCP_STREAMABLE_HTTP_PATH=/mcp
```

Current staging endpoint:

```text
https://researchops-mcp.onrender.com/mcp
```

Render free storage is ephemeral. Use durable managed storage before treating the deployment as production for real user data.

## Configuration

Important runtime options are available as CLI flags or environment-backed server settings:

```powershell
python src/server.py --help
```

Common options:

- `--transport stdio` or `--transport streamable-http`
- `--host 127.0.0.1`
- `--port 8765`
- `--database-path data/researchops.db`
- `--auth-enabled`
- `--resource-server-url http://127.0.0.1:8765/mcp`
- `--openalex-timeout-seconds 5`
- `--openalex-deadline-seconds 12`
- `--openalex-retry-attempts 2`
- `--rate-limit-max-requests 30`
- `--max-http-body-bytes 32768`

## Documentation

- `docs/design.md`: architecture and request flows
- `docs/threat-model.md`: security model and remaining risks
- `docs/release-checklist.md`: release checklist and compatibility policy
- `docs/demo-script.md`: 3-5 minute walkthrough script
- `docs/evaluation-report.md`: evaluation summary
- `docs/decisions.md`: architecture decisions
- `docs/tracker.md`: project progress record
- `docs/learning.md`: MCP learning notes

## Known Limitations

- Demo bearer tokens are not a production OAuth/OIDC integration.
- SQLite is suitable for local and staging use; use durable managed storage for production user data.
- Render free deployments use disposable filesystem storage unless configured otherwise.
- Rate limiting, circuit breaking, and metrics are process-local.
- OpenTelemetry API spans are present, but no exporter or collector is configured by default.
- `/metrics` returns JSON, not Prometheus exposition format.
- `/readyz` is shallow and does not perform deep database or OpenAlex probes.
- Long-running Tasks and MCP Apps are documented future extensions, not enabled runtime features.

## Compatibility Policy

Treat MCP metadata as public contract. Tool names, descriptions, schemas, resource URI templates, prompt names, and prompt arguments affect client and model behavior.

Compatibility rules:

- Prefer additive optional fields over breaking changes.
- Do not rename tools, resources, prompts, or required arguments without a migration note.
- Keep old capabilities available during a migration window when possible.
- Keep metadata regression tests and evaluation gates updated with intentional interface changes.
- Server-side authorization must remain authoritative even if clients cache tool metadata.

## License

No license has been declared yet.