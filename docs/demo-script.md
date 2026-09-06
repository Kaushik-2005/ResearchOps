# ResearchOps MCP Demo Script

## Goal

Record a 3-5 minute walkthrough that demonstrates ResearchOps MCP as a production-style MCP server, not just a collection of Python functions.

## Demo Setup

Install dependencies:

```powershell
python -m pip install -e .[dev]
```

Run quick verification:

```powershell
pytest
python -m researchops_mcp.evals --fail-on-thresholds
```

## Demo Flow

### 1. Explain The Project

Say:

ResearchOps MCP exposes a research workflow through the Model Context Protocol. It supports tools for actions, resources for stable context, prompts for reusable reasoning templates, and both local and HTTP transports.

Show:

```powershell
python client/cli.py discover
python client/cli.py list-tools
```

### 2. Show Read-Only Research Tools

Show bounded search and stable paper lookup:

```powershell
python client/cli.py call-tool search_papers --arg "query=Model Context Protocol" --arg "limit=2"
python client/cli.py call-tool get_paper --arg "paper_id=W7129030749"
python client/cli.py call-tool export_bibtex --arg "paper_id=W7129030749"
```

Point out:

- stable OpenAlex IDs
- bounded results
- structured outputs
- separate citation-export tool

### 3. Show Resources And Prompts

Show resource reads and reusable prompt templates:

```powershell
python client/cli.py read-resource paper://W7129030749
python client/cli.py get-prompt compare_papers --arg "paper_id_a=W7129030749" --arg "paper_id_b=W4417069007" --arg "focus=security trade-offs"
```

Point out:

- resources expose reusable context by URI
- prompts provide reusable scaffolding
- untrusted external content is labeled

### 4. Show Safe Writes

Show write approval and idempotency:

```powershell
python client/cli.py --yes call-tool create_reading_list --arg "name=Demo List" --arg "idempotency_key=demo-list-1"
```

Point out:

- write tools are separate from read tools
- client-side approval is explicit
- server-side authorization and idempotency still matter

### 5. Show HTTP, Auth, And Observability

Start HTTP server:

```powershell
python src/server.py --transport streamable-http --host 127.0.0.1 --port 8765 --stateless-http
```

From another terminal:

```powershell
Invoke-RestMethod http://127.0.0.1:8765/healthz
Invoke-RestMethod http://127.0.0.1:8765/readyz
Invoke-RestMethod http://127.0.0.1:8765/metrics
```

Point out:

- Streamable HTTP is available for remote hosting
- request IDs and operation metrics make failures diagnosable
- the Docker and Render setup make this deployable

### 6. Show Quality Gates

Show:

```powershell
pytest
python -m researchops_mcp.evals --fail-on-thresholds
docker build -t researchops-mcp:demo .
```

Point out:

- unit tests verify business logic
- integration tests verify MCP flows
- metadata regression tests protect tool schemas and descriptions
- evals catch model-facing interface regressions
- Docker build proves package metadata is complete enough for clean deployment

## Closing Statement

Say:

The project is portfolio-ready as a local and staging MCP server. The remaining production work is explicit: real OAuth provider integration, durable managed database hosting, external metrics/tracing export, distributed rate limiting, and optional Tasks or Apps extensions.