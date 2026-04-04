# MCP Server Creation Skill

A structured workflow for building production-ready MCP (Model Context Protocol) servers from a template project to a published PyPI package. Based on real-world experience building the Asgard MCP server ecosystem.

---

## When to Use

- Building a new MCP server from the Asgard MCP server template
- The project has a `reference/` folder containing the service's documentation (API specs, SDK docs, protocol guides, or any third-party service documentation)
- You need to transform a template into a working MCP server tailored to a specific service

## Inputs

Every MCP server project starts with two things:

1. **The template** -- An Asgard MCP server template project (already cloned/set up as the working directory). Contains pluggable auth modules, connectors, sample tools, and project scaffolding.

2. **The reference material** -- Located in the project's `reference/` folder. This is NOT limited to REST API docs. It can be:
   - REST API specification (PDF, OpenAPI, Swagger)
   - SOAP/XML web service documentation
   - SDK or library documentation
   - Protocol specification (MQTT, WebSocket, gRPC, etc.)
   - Third-party platform guides (CRM docs, ERP manuals, government data portals)
   - Any document describing how to interact with the target service

The reference material determines the entire architecture. Read it first, then decide how to adapt the template.

## Overview

The process follows 7 phases:

```
Phase 0: Understand the Template   -->  Know what's available to keep, modify, or delete
Phase 1: Read & Understand Ref     -->  Complete knowledge of the service's interface
Phase 2: Design                    -->  Architecture decisions, design spec
Phase 3: Plan                      -->  Implementation plan with tasks
Phase 4: Implement                 -->  Working code, tests, docs
Phase 5: Verify & Fix              -->  All tests passing
Phase 6: Publish                   -->  PyPI package + GitHub release
```

---

## Phase 0: Understand the Template

**Goal:** Know what the template provides so you can make informed decisions about what to keep, delete, or replace.

### Steps

1. **Read the template's CLAUDE.md** -- It describes the architecture and conventions
2. **Map the template structure:**

```
Template Structure:
├── app.py                    # MCPServer/FastMCP singleton (ALWAYS modify)
├── mcp_server.py             # Entry point, tool imports (ALWAYS modify)
├── config/settings.py        # Endpoints, URL builder, auth delegation (ALWAYS rewrite)
├── auth/                     # Pluggable auth modules (PICK ONE or create custom)
│   ├── bearer.py             #   Bearer token
│   ├── api_key.py            #   API key (header or query)
│   ├── oauth2.py             #   OAuth 2.0 client credentials
│   └── none.py               #   No auth (public APIs)
├── connectors/               # Data source connectors (PICK ONE or create custom)
│   ├── rest_client.py        #   HTTP REST with retry, pagination
│   ├── rss_client.py         #   RSS/Atom feed parser
│   ├── scraper_client.py     #   Web scraper with BeautifulSoup
│   ├── mqtt_client.py        #   MQTT for IoT/industrial
│   └── graphql_client.py     #   GraphQL with cursor pagination
├── tools/sample_tools.py     # Example tools (ALWAYS delete & replace)
├── pyproject.toml            # Package metadata (ALWAYS modify)
├── .env.example              # Env var template (ALWAYS rewrite)
├── .mcp.json                 # Claude Code config (ALWAYS rewrite)
├── scripts/auth/test_connection.py  # Connection test (ALWAYS rewrite)
└── tests/test_all_tools.py   # E2E tests (ALWAYS rewrite)
```

3. **Understand the pluggable patterns:**
   - **Auth:** Each module exports `get_auth_headers() -> dict`. Pick one or write a custom one.
   - **Connectors:** Each provides helpers (e.g., `api_get()`, `api_post()`, `fetch_all_pages()`). Pick one, customize, or write new.
   - **Tools:** `@mcp.tool()` decorated functions with Pydantic `Field()` for params. All return `dict`.

### Deliverable

A mental inventory of the template: what exists, what patterns to follow, what to delete.

---

## Phase 1: Read & Understand the Reference

**Goal:** Build a complete mental model of the service before writing any code.

### Steps

1. **Locate reference material** in the project's `reference/` folder
2. **Read EVERYTHING** -- Do not skim. For PDFs, read in 20-page batches. For web docs, follow all linked pages.
3. **Classify the service type** -- This determines your architecture:

| Service Type | Template Modules to Use | Example |
|-------------|------------------------|---------|
| Standard REST API | `rest_client.py` + existing auth module | Shopline, Stripe |
| REST with custom encryption | Custom connector + custom auth | ezPay (AES-256-CBC) |
| SOAP/XML web service | Custom connector | Government services, legacy ERP |
| RSS/Atom feeds | `rss_client.py` + `none.py` | News aggregation |
| Web scraping | `scraper_client.py` + `none.py` | Sites without APIs |
| MQTT/IoT | `mqtt_client.py` | Sensor data, industrial |
| GraphQL | `graphql_client.py` + auth module | Modern platforms |
| SDK/Library wrapper | Custom connector (call SDK) | Cloud services with Python SDK |
| File/Data processing | No connector needed | Local file transformation |

4. **Extract key details based on service type:**

**For APIs (REST, SOAP, GraphQL):**

| Detail | What to capture |
|--------|----------------|
| **Endpoints** | URL paths, HTTP methods, test vs production base URLs |
| **Authentication** | Bearer, API key, OAuth, custom encryption, certificates |
| **Request format** | JSON, form POST, XML, multipart, custom encoding |
| **Parameters** | Per-endpoint: name, type, required/optional, constraints |
| **Response format** | Success/error shapes, nested parsing |
| **Error codes** | All codes and meanings |
| **Rate limits** | Requests per second/minute, pagination |
| **Encryption/signing** | AES, HMAC, signatures, checksums |

**For non-API services (SDK, protocol, scraping):**

| Detail | What to capture |
|--------|----------------|
| **Connection method** | How to connect (TCP, WebSocket, SDK init, URL pattern) |
| **Authentication** | Credentials, certificates, tokens |
| **Operations** | What actions are available (read, write, subscribe) |
| **Data format** | How data is structured (JSON, XML, binary, HTML) |
| **Error handling** | How errors are reported (exceptions, status codes, error events) |
| **Dependencies** | Required libraries or SDKs |

5. **Identify the tricky parts** -- Things that are unique or non-obvious:
   - Custom encryption protocols
   - Fields that must be empty strings vs omitted
   - Multi-step workflows (auth then request, polling, callbacks)
   - Character encoding requirements
   - Date/time format quirks
   - Business logic validation rules

### Deliverable

A clear understanding of:
- How many tools to create (1 per logical operation, NOT necessarily 1 per endpoint)
- Which template modules to keep/delete/create
- What dependencies are needed
- What credentials/config the user needs

---

## Phase 2: Design

**Goal:** Decide architecture and get user approval before coding.

### Steps

1. **Map reference findings to template structure:**
   - Match the service's auth to a template auth module (or decide to create custom)
   - Match the service's protocol to a template connector (or decide to create custom)
   - Define which tools to expose

2. **Propose 2-3 approaches** with trade-offs:

   | Approach | When to use |
   |----------|-------------|
   | **Keep template connector** | Service is standard REST with standard auth |
   | **Minimal custom connector** | Service has unique protocol (encryption, non-REST, SDK) |
   | **Hybrid** | Standard REST but with custom auth or pre/post processing |

3. **Present the design** covering:
   - Files to delete (unused template modules)
   - Files to create (new auth, connector, tools)
   - Files to modify (app.py, mcp_server.py, config, pyproject.toml)
   - Data flow diagram
   - Tool list with key parameters
   - Error handling strategy

4. **Write design spec** to `docs/superpowers/specs/YYYY-MM-DD-<name>-design.md`

### Key Architecture Decisions

**Connector choice decision tree:**

```
Is it a standard REST API with JSON body?
├── Yes → Does it use standard auth (Bearer/API key/OAuth)?
│   ├── Yes → Keep rest_client.py + matching auth module
│   └── No  → Keep rest_client.py + create custom auth module
└── No  → What is it?
    ├── REST with custom encoding (form POST, encryption) → Custom connector
    ├── GraphQL → Keep graphql_client.py
    ├── RSS/Atom → Keep rss_client.py
    ├── Web scraping → Keep scraper_client.py
    ├── MQTT/IoT → Keep mqtt_client.py
    └── SDK/Protocol/Other → Custom connector
```

**Tool granularity:** Generally 1 tool per user-facing operation. Group related API calls into one tool if they always happen together (e.g., "get token then fetch data" = 1 tool).

**Optional field handling:** Some APIs expect ALL fields present (empty string for unused ones). Others want unused fields omitted. Check sample code in the reference docs.

---

## Phase 3: Plan

**Goal:** Break implementation into small, independent tasks with exact file paths and code.

### Task Structure

A typical MCP server has these tasks:

```
Task 1: Clean up template (delete unused files)
Task 2: Update project config (pyproject.toml, app.py, mcp_server.py, .env.example, .mcp.json)
Task 3: Implement auth module (if custom auth needed)
Task 4: Implement config/settings.py (endpoints/config, env vars)
Task 5: Implement connector (service client)
Task 6: Implement MCP tools (@mcp.tool() functions)
Task 7: Test connection script
Task 8: E2E tests for all tools
Task 9: Documentation (README.md, README.zh-TW.md, CLAUDE.md, CHANGELOG.md)
Task 10: Final verification
```

### Task Dependencies

```
Task 1 ──┐
Task 2 ──┤ (independent, can run in parallel)
Task 4 ──┘
           ↓
Task 3 ──→ Task 5 ──→ Task 6 ──→ Task 7 + Task 8 (parallel)
                                        ↓
                                   Task 9 ──→ Task 10
```

### Plan Document

Save to `docs/superpowers/plans/YYYY-MM-DD-<name>.md` with:
- File structure table (create/modify/delete)
- Each task with exact file paths, complete code, and test commands
- Commit messages for each task

---

## Phase 4: Implement

**Goal:** Execute the plan task by task, testing as you go.

### Execution Strategy

**Parallel-safe tasks** (dispatch simultaneously):
- Task 1 (cleanup) + Task 2 (config) + Task 4 (settings)
- Task 3 (auth) + Task 5 (connector) -- if auth doesn't depend on connector
- Task 7 (connection test) + Task 8 (E2E tests)

**Sequential tasks:**
- Task 6 (tools) depends on Task 5 (connector)
- Task 7-8 (tests) depend on Task 6 (tools)
- Task 9 (docs) after tests pass

### Common Pitfalls

1. **FastMCP vs MCPServer** -- The `mcp` package uses `FastMCP` not `MCPServer`:
   ```python
   # WRONG
   from mcp.server.mcpserver import MCPServer
   mcp = MCPServer("name")
   
   # CORRECT
   from mcp.server.fastmcp import FastMCP
   mcp = FastMCP("name")
   ```

2. **Pydantic FieldInfo in direct calls** -- When MCP tool functions use `Field(default=None)` and are called directly (not via MCP protocol), defaults remain as `FieldInfo` objects, not `None`. Fix with a helper:
   ```python
   from pydantic.fields import FieldInfo
   
   def _val(v, default=""):
       """Resolve a parameter value, handling FieldInfo defaults from direct calls."""
       if v is None or isinstance(v, FieldInfo):
           return default
       return v
   ```

3. **Empty string vs omitted fields** -- Some APIs require ALL fields present as empty strings. Others reject empty strings. Test with the actual service to determine behavior. Check sample code in the reference docs.

4. **pyproject.toml build backend** -- Use `setuptools.build_meta` for PyPI publishing:
   ```toml
   [build-system]
   requires = ["setuptools>=68.0"]
   build-backend = "setuptools.build_meta"
   ```

5. **Python bytecode cache** -- After editing files, clear `__pycache__` if tests show stale behavior:
   ```bash
   find . -path ./.venv -prune -o -name "__pycache__" -type d -exec rm -rf {} +
   ```

### Test Against Live Service

Always test against the real service (test environment) during development. Real responses reveal:
- Validation requirements the docs don't mention
- Whether empty strings or missing fields are expected
- Actual error behavior
- Auth/encryption correctness

---

## Phase 5: Verify & Fix

**Goal:** All tests pass, all edge cases handled.

### Verification Checklist

```
[ ] Auth/crypto unit tests pass (if applicable)
[ ] Connection test passes (credentials + connectivity)
[ ] All tool E2E tests pass (no Python exceptions)
[ ] All tools get valid responses (even errors prove connectivity)
[ ] At least one tool returns SUCCESS with real data
[ ] MCP server starts without errors
[ ] Git working tree is clean
```

### Common Fixes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `ModuleNotFoundError` | Dependency not installed | `uv pip install <package>` |
| `MCPServer not found` | Wrong import | Use `FastMCP` from `mcp.server.fastmcp` |
| API field format errors | Sending FieldInfo objects | Add `_val()` helper |
| API rejects empty fields | API wants fields omitted | Don't include in payload |
| API rejects missing fields | API wants empty strings | Include as `""` |
| Stale code after edits | Python bytecode cache | Delete `__pycache__` dirs |

---

## Phase 6: Publish

**Goal:** Package on PyPI with automated release workflow.

### Step 1: Update pyproject.toml for PyPI

```toml
[project]
name = "mcp-your-service"
version = "0.1.0"
description = "MCP server wrapping ... into N AI Agent-callable tools for ..."
readme = "README.md"
license = "MIT"
requires-python = ">=3.10"
authors = [{ name = "Your Name" }]
keywords = ["mcp", "your-service", "ai-agent", "claude"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Libraries",
]
dependencies = ["requests", "mcp", ...]

[project.urls]
Homepage = "https://github.com/org/mcp-your-service"
Repository = "https://github.com/org/mcp-your-service"
Issues = "https://github.com/org/mcp-your-service/issues"

[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"

[project.scripts]
mcp-your-service = "mcp_server:main"
```

### Step 2: Create GitHub Actions Workflow

Create `.github/workflows/publish.yml`:

```yaml
name: Publish to PyPI

on:
  release:
    types: [published]

permissions:
  contents: read

jobs:
  build:
    name: Build distribution
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install build dependencies
        run: pip install build
      - name: Build package
        run: python -m build
      - name: Store distribution packages
        uses: actions/upload-artifact@v4
        with:
          name: python-package-distributions
          path: dist/

  publish-to-pypi:
    name: Publish to PyPI
    needs: [build]
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/mcp-your-service
    permissions:
      id-token: write
    steps:
      - name: Download distributions
        uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
```

### Step 3: Set Up PyPI Trusted Publisher

On https://pypi.org/manage/account/publishing/:
- **PyPI project name:** `mcp-your-service`
- **Owner:** your GitHub org
- **Repository:** `mcp-your-service`
- **Workflow name:** `publish.yml`
- **Environment name:** `pypi`

### Step 4: Create GitHub Environment

Repo Settings > Environments > New environment named `pypi`.

### Step 5: Update GitHub Repo Metadata

```bash
gh repo edit org/mcp-your-service \
  --description "MCP Server for ... — N AI-callable tools for ..." \
  --add-topic "mcp,mcp-server,ai-tools,..." \
  --homepage "https://..."
```

### Step 6: Update README with All Install Methods

Include these in README:

1. **PyPI install** -- `pip install mcp-your-service`
2. **uvx** -- `uvx mcp-your-service` (no install needed)
3. **Source install** -- `git clone` + `uv pip install -e .`
4. **Claude Desktop** -- JSON config (uvx variant + pip variant)
5. **Claude Code** -- Auto-discovery + `claude mcp add`
6. **Cursor / Windsurf** -- MCP settings config
7. **Direct run** -- `python mcp_server.py` or `uvx`

### Step 7: Add Shield Badges

```markdown
[![PyPI version](https://img.shields.io/pypi/v/mcp-your-service)](https://pypi.org/project/mcp-your-service/)
[![Python](https://img.shields.io/pypi/pyversions/mcp-your-service)](https://pypi.org/project/mcp-your-service/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![MCP](https://img.shields.io/badge/MCP-compatible-blue)](https://modelcontextprotocol.io/)
[![GitHub stars](https://img.shields.io/github/stars/org/mcp-your-service)](https://github.com/org/mcp-your-service/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/org/mcp-your-service)](https://github.com/org/mcp-your-service/issues)
[![GitHub last commit](https://img.shields.io/github/last-commit/org/mcp-your-service)](https://github.com/org/mcp-your-service/commits/main)
```

### Step 8: Push, Release, Verify

```bash
git push origin main
gh release create v0.1.0 --title "v0.1.0 - Initial Release" --notes "..."
gh run watch  # watch the publish workflow
```

---

## README Template

A good MCP server README has these sections:

```
1. Title + shield badges
2. Language toggle (English / 繁體中文)
3. Overview (1 paragraph)
4. Features (tool list + technical highlights)
5. Prerequisites
6. Installation (PyPI, uvx, source)
7. Configuration (env vars table)
8. Usage (Claude Desktop, Claude Code, Cursor, direct run, testing)
9. Usage Examples (conversational format with real results)
10. Tools Reference (table)
11. Error Codes Reference
12. Architecture (text diagram + file tree)
13. Contributing
14. License
```

### Usage Examples Format

Follow this conversational pattern for each example:

```markdown
### "Description of what the user wants to do"

> **You:** 用自然語言描述需求

**AI calls:**

\```
tool_name(
  param1 = "value1",
  param2 = "value2",
)
\```

**Result:** `SUCCESS` -- Description of what happened and key response data.
```

Include 6-8 examples covering main workflows. Use real responses when possible.

---

## Checklist

### Before starting
- [ ] Template project is set up as working directory
- [ ] Reference material is in `reference/` folder
- [ ] Test credentials are obtained (if applicable)

### After Phase 1 (Read Reference)
- [ ] Service type identified (REST, SDK, protocol, etc.)
- [ ] Know which template modules to keep/delete
- [ ] Know what custom modules to create
- [ ] Dependencies identified

### After Phase 2 (Design)
- [ ] User approved the design approach
- [ ] Design spec committed

### After Phase 4 (Implement)
- [ ] Template cleanup done
- [ ] Auth module works (if applicable)
- [ ] Config has all endpoints/settings and env vars
- [ ] Connector handles the service protocol correctly
- [ ] All tools implemented with proper Field() descriptions
- [ ] Connection test script works
- [ ] E2E tests pass

### After Phase 5 (Verify)
- [ ] At least one tool returns SUCCESS with real data
- [ ] No field format or protocol errors
- [ ] FieldInfo handling is correct for direct calls
- [ ] Git is clean

### After Phase 6 (Publish)
- [ ] pyproject.toml has PyPI metadata
- [ ] GitHub Actions workflow created
- [ ] PyPI trusted publisher configured
- [ ] Package published to PyPI
- [ ] GitHub repo metadata set (description, topics, homepage)
- [ ] README has all install methods
- [ ] README has shield badges
- [ ] README has conversational usage examples
- [ ] Both English and Chinese READMEs updated
