# LotLine — Manufacturing Traceability and Containment Prototype

LotLine is a synthetic-data manufacturing traceability prototype built around a FalkorDB knowledge graph. It helps users investigate a source lot, follow recorded manufacturing genealogy, inspect related quality and shipment information, retrieve source-backed context, and exercise a limited set of controlled graph updates.

This repository describes a demo application. It is **not** connected to a real factory, supplier, customer, carrier, ERP, MES, QMS, or logistics platform, and it must not be used to make operational decisions about real products.

## Contents

- [Purpose and scope](#purpose-and-scope)
- [Implemented workflows](#implemented-workflows)
- [Architecture](#architecture)
- [Read and write boundaries](#read-and-write-boundaries)
- [Technology stack](#technology-stack)
- [Graph and data assumptions](#graph-and-data-assumptions)
- [Configuration](#configuration)
- [Run locally](#run-locally)
- [Validation and tests](#validation-and-tests)
- [Known limitations](#known-limitations)
- [Project layout](#project-layout)
- [Security and data handling](#security-and-data-handling)

## Purpose and scope

Manufacturing traceability questions often require following links between material lots, parts, suppliers, work orders, machines, quality events, customer orders, and shipments. LotLine uses recorded graph relationships to make these connections inspectable in one interface.

The project uses synthetic records and a pre-existing graph. Runtime writes affect that configured demo graph only. LotLine does not provision, seed, reset, migrate, back up, or restore the database on startup.

## Implemented workflows

The Streamlit application contains five pages.

### 1. Overview

- Reads the configured graph through the read-only MCP interface.
- Displays node counts by label and relationship counts by type.
- Shows selected demo records and a graph-integrity observation.
- Treats query failures as failures; a failed query is not displayed as a count of zero.
- Checks the intentionally disconnected machine fixture `MC-CNC-01` instead of inventing a relationship for it.

### 2. Investigate

- Inspects a source lot and displays related part, supplier, and quality-event information.
- Traces downstream lot genealogy using recorded `INPUT_TO`, `PRODUCES`, and `SPLIT_INTO` relationships. A bill-of-materials `REQUIRES` relationship is not treated as proof of actual material consumption.
- Lists relevant work orders, downstream shipments/orders, and applicable source records.
- Distinguishes already-dispatched shipments from planned shipments when presenting containment options.
- Saves an investigation case to the synthetic graph.
- Provides a containment approval workflow for eligible planned shipments.
- Includes an agent interface that can select from the application's allowlisted investigation tools, inspect results, and produce a natural-language response. The agent has a bounded iteration budget; if it reports reaching that limit, its answer may be incomplete.

Saving an investigation case records the case and its graph relationship. It does not itself change a lot or shipment status.

### 3. Factory Operations

Explicit forms support a limited set of synthetic business events:

- Register a synthetic supplier and link it to a part.
- Receive a material lot from a supplier that has the required current `APPROVED_FOR` relationship for the part.
- Split a fully available lot into available and scrapped portions, subject to the implemented validation rules.

These are bounded demonstration operations, not a general-purpose factory integration. Supplier creation marks the supplier as approved for the selected part; there is no separate real supplier qualification or approval workflow.

### 4. Company Brain

- Finds applicable source records and relevant expertise using graph relationships.
- Inspects recorded decisions and their provenance.
- Shows project scope, covered work orders, decisions, and owner teams.
- Accepts a natural-language graph question through the Text-to-Cypher workflow.
- Displays the generated query and information about schema terms used.

The Text-to-Cypher service generates a query only. LotLine validates the generated Cypher against its read-only policy and the live schema, then executes it using the read-only MCP query tool. Unsupported query shapes, including procedure calls, are rejected by the current application policy.

### 5. Scenario Lab

- Runs repeatable, graph-grounded read-only acceptance scenarios.
- Shows expected and actual results for implemented checks.
- Includes a connectivity audit that attempts to use the algorithm capability available in the configured FalkorDB environment.
- Does not fabricate a result if a database algorithm or procedure is unavailable.

Run the read-only scenarios against the expected baseline before performing demo writes.

## Architecture

```text
Streamlit UI
  ├── Overview / Investigate / Company Brain / Scenario Lab
  │     └── GraphService and application services
  │           ├── ReadOnlyGraphAdapter
  │           │     └── FalkorDB MCP server over stdio
  │           │           ├── list_graphs
  │           │           ├── get_schema
  │           │           ├── query_read
  │           │           └── explain
  │           └── Text-to-Cypher REST service
  │                 └── generates Cypher only; LotLine validates it;
  │                     execution returns through MCP query_read
  └── Explicit write forms / approved containment action
        └── Deterministic Python validation and WriteRepository
            using the FalkorDB Python client
```

The Investigate agent uses the OpenAI Python SDK against Google's OpenAI-compatible Generative Language endpoint. The current model default is `gemini-3.5-flash-lite`, and the API key/model are configurable. The agent can call only the tools declared in LotLine's application code.

The MCP read session is launched by the application as a subprocess over stdio. LotLine sets `FALKORDB_MCP_ALLOW_WRITES=0` and independently allowlists read tools. Explicit application writes use a separate Python repository and are not exposed as general-purpose write tools to the agent.

## Read and write boundaries

### Read path

1. Company Brain asks Text-to-Cypher to generate a query with `cypher_only=true`.
2. LotLine validates the query shape and checks labels and relationship types against the schema retrieved through MCP.
3. The application rejects unsafe or unsupported clauses and enforces a row limit.
4. The query executes through MCP's `query_read` tool.

The query guard is an application-level defense-in-depth measure, not a general-purpose Cypher parser. The read-only MCP execution path is an additional enforcement boundary.

### Write path

Writes occur only through explicit application workflows implemented in Python, including saving an investigation case, approving an eligible planned-shipment hold, registering a synthetic supplier, receiving a lot, or splitting an available lot. These operations use deterministic application validation and reviewed query templates. The natural-language agent does not receive a general-purpose graph-write tool.

Writes affect the configured synthetic graph and may persist across restarts depending on the FalkorDB deployment's persistence settings.

## Technology stack

- Python — application and business logic
- Streamlit — multipage user interface
- FalkorDB — graph database
- FalkorDB MCP server — read-only graph query interface
- FalkorDB Text-to-Cypher — separate REST service for query generation
- Google Gemini API — natural-language agent model through the OpenAI-compatible endpoint
- OpenAI Python SDK — compatible chat-completions client
- Pydantic — data validation
- pytest — automated tests

See `requirements.txt` and `requirements-dev.txt` for Python dependency ranges.

## Graph and data assumptions

- The configured graph defaults to `LOTLINE_DEV`.
- The application expects an existing, preloaded graph; it does not create, seed, reset, or restore it.
- The augmented demo baseline is expected to contain **116 nodes and 241 relationships before runtime cases, containment actions, or factory-operation records are added**. This is a baseline expectation, not a guarantee about a graph that has already been changed.
- `MC-CNC-01` is intentionally disconnected in the expected baseline. Its lack of relationships is a deliberate test condition.
- `lotline/knowledge/BUSINESS_DATA_DICTIONARY.md` describes the business graph context.
- Seed data, graph-loading scripts, and backup/restore procedures are maintained separately from the Streamlit app.

Do not reset or overwrite an existing graph merely to make its counts match the baseline. Inspect the current data first and account for demo operations that may already have added records.

## Configuration

LotLine reads `.env` with `python-dotenv`. The current code reads these variables:

| Variable | Default / purpose |
|---|---|
| `GEMINI_API_KEY` | Required for Investigate agent requests. |
| `GEMINI_MODEL` | Defaults to `gemini-3.5-flash-lite`. |
| `FALKORDB_HOST` | `localhost` |
| `FALKORDB_PORT` | `6379` |
| `FALKORDB_USERNAME` | `default` |
| `FALKORDB_PASSWORD` | Empty if database authentication is not used. |
| `FALKORDB_GRAPH` | `LOTLINE_DEV` |
| `FALKORDB_MCP_COMMAND` | `falkordb-mcp`; use an absolute executable path on Windows if it is not on `PATH`. |
| `FALKORDB_MCP_MAX_ROWS` | `200`, bounded by application configuration. |
| `TEXT_TO_CYPHER_URL` | `http://127.0.0.1:8080` |
| `TEXT_TO_CYPHER_ENABLED` | `true` |
| `MAX_AGENT_ITERATIONS` | `6`, clamped to 1–12. This limits model/tool rounds, not a guaranteed number of individual tool calls or successful completion. |
| `MAX_QUERY_ROWS` | `100`, clamped to 1–200. |
| `QUERY_TIMEOUT_SECONDS` | `30`, clamped to 5–120 seconds. |
| `LOTLINE_DEMO_MODE` | `true`. |

Text-to-Cypher has its own model/provider credentials and configuration. Configure it according to the official repository version installed locally. LotLine sends it a query-generation request and does not pass the agent's API key through this client.

Use the variable names actually read by `lotline/config.py`. If a checked-in example file differs from this list, reconcile that example before using it as the source for a local `.env`.

## Run locally

For the complete Windows setup—including FalkorDB, the MCP server, Text-to-Cypher, environment configuration, and verification—follow [`SETUP_INSTRUCTIONS.md`](SETUP_INSTRUCTIONS.md).

Once prerequisites and configuration are ready, run from the project root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements-dev.txt
python -m compileall -q .
python -m pytest -q
python scripts\verify_integrations.py
streamlit run app.py
```

Streamlit prints the local URL, normally `http://localhost:8501`.

## Validation and tests

Run from the project root:

```powershell
python -m compileall -q .
python -m pytest -q
python scripts\verify_integrations.py
```

- `compileall` checks Python syntax/bytecode compilation.
- `pytest` runs the repository's automated unit tests.
- `verify_integrations.py` checks selected live graph/service assumptions and integration capabilities. It requires the relevant services and graph to be available.

A passing unit-test suite alone does not prove that live FalkorDB, MCP, Text-to-Cypher, Gemini, or graph-algorithm paths work in a particular environment. Report the actual command output when describing validation. For the demo sequence and scenario-specific expected results, see `HOW_TO_TEST_PROJECT.md`; verify its assumptions against the current configuration before following it.

## Known limitations

- This is a synthetic-data prototype, not a production manufacturing system.
- The application does not provision, seed, migrate, reset, back up, or restore the graph.
- The natural-language agent has a bounded iteration budget; complex questions can reach the limit before a complete answer is produced.
- Text-to-Cypher output is restricted by LotLine's current read-only policy. Procedure calls and other unsupported query shapes can be rejected even if a given FalkorDB deployment supports them.
- The schema/query guard uses application-level checks and regular-expression-based inspection; it is not a full Cypher parser.
- Connectivity algorithms depend on the installed FalkorDB image's available procedures/capabilities. Failed checks must be diagnosed rather than replaced with assumed results.
- Write workflows are deliberately narrow and are not a complete approval, inventory, supplier qualification, or shipment-management system.
- No external carrier, supplier, ERP, MES, QMS, or customer system is contacted.
- Authentication, user management, and production-grade deployment controls are outside the current prototype's scope.

## Project layout

```text
lotline-app/
├── app.py
├── pages/
│   ├── 1_Overview.py
│   ├── 2_Investigate.py
│   ├── 3_Factory_Operations.py
│   ├── 4_Company_Brain.py
│   └── 5_Scenario_Lab.py
├── lotline/
│   ├── agent/          # Agent prompts, tool schemas, dispatcher and orchestration
│   ├── graph/          # Query guard, graph helpers and controlled write repository
│   ├── knowledge/      # Business data dictionary
│   ├── mcp/            # MCP client and read-only adapter
│   ├── services/       # Investigation, containment, factory and graph services
│   ├── ui/             # Shared Streamlit UI helpers
│   ├── config.py
│   ├── runtime.py
│   └── text_to_cypher.py
├── scripts/verify_integrations.py
├── tests/
├── requirements.txt
├── requirements-dev.txt
├── .env.example
├── HOW_TO_TEST_PROJECT.md
├── README.md
└── SETUP_INSTRUCTIONS.md
```

## Security and data handling

- Keep `.env` local and never commit API keys or database credentials.
- Use synthetic data and dedicated demo credentials, not production data or credentials.
- Keep graph reads on the read-only MCP path and writes limited to explicit application workflows on the synthetic graph.
- Review generated queries and returned rows when debugging Company Brain results.
- Back up the graph before testing write workflows if its current state matters.
- Rotate any credential exposed in a screenshot, log, or repository.

## Detailed setup

See [`SETUP_INSTRUCTIONS.md`](SETUP_INSTRUCTIONS.md) for end-to-end local installation and [`HOW_TO_TEST_PROJECT.md`](HOW_TO_TEST_PROJECT.md) for the test/demo runbook.
