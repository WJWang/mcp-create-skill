---
name: mcp-create
description: Use when building a new MCP server from the Asgard MCP Template. Guides the complete workflow from reading service documentation through design, implementation, testing, and PyPI publishing.
---

# MCP Server Creation Skill

A structured workflow for building production-ready MCP (Model Context Protocol) servers from a template project to a published PyPI package. Based on real-world experience building the Asgard MCP server ecosystem.

---

## When to Use

- Building a new MCP server from the Asgard MCP server template
- The project has a `reference/` folder containing the service's documentation (API specs, SDK docs, protocol guides, or any third-party service documentation)
- You need to transform a template into a working MCP server tailored to a specific service

## Inputs

Every MCP server project starts with two things:

1. **The template** -- An [Asgard MCP server template](https://github.com/asgard-ai-platform/mcp-template) project (already cloned/set up as the working directory). Contains pluggable auth modules, connectors, sample tools, project scaffolding, PyPI publish workflow, and commented-out shield badges.

2. **The reference material** -- Located in the project's `reference/` folder (gitignored -- not committed to the repo). This is NOT limited to REST API docs. It can be:
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
Phase 5: Verify & Fix              -->  All tests passing, template leftovers cleaned
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
├── app.py                    # FastMCP singleton (ALWAYS modify: rename service)
├── mcp_server.py             # Entry point, tool imports (ALWAYS modify: change import)
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
├── tools/sample_tools.py     # Example tools + _val() helper (ALWAYS delete & replace)
├── pyproject.toml            # Package metadata with PyPI fields (ALWAYS modify)
├── .env.example              # Env var template (ALWAYS rewrite)
├── .mcp.json                 # Claude Code config (ALWAYS rewrite)
├── .github/workflows/publish.yml  # PyPI publish workflow (update package name)
├── CONTRIBUTING.md           # Contribution guide (update service name + connector import)
├── scripts/auth/test_connection.py  # Connection test (ALWAYS rewrite)
├── tests/test_all_tools.py   # E2E tests (ALWAYS rewrite)
└── reference/                # Service docs (gitignored, NOT committed)
```

3. **Know what the template already provides for publishing:**
   - `pyproject.toml` has `keywords`, `classifiers`, `[project.urls]` placeholders -- just fill them in
   - `.github/workflows/publish.yml` is ready -- just update the package name on the PyPI URL line
   - `README.md` has commented-out shield badges -- uncomment and replace `{service}` after publishing
   - `README.md` includes install methods (PyPI, uvx, source) and client configs (Claude Desktop, Claude Code, Cursor)

4. **Know the `_val()` helper pattern:**
   - `tools/sample_tools.py` includes the `_val()` helper function and documentation
   - Copy this helper to your tools file -- it handles Pydantic FieldInfo defaults in direct test calls
   - See "Common Pitfalls" in Phase 4 for details

### Deliverable

A mental inventory of the template: what exists, what patterns to follow, what to delete.

---

## Phase 1: Read & Understand the Reference

**Goal:** Build a complete mental model of the service before writing any code.

### Steps

1. **Locate reference material** in the project's `reference/` folder
   - Note: `reference/` is in `.gitignore` -- these files stay local, not committed
2. **Read EVERYTHING** -- Do not skim. For PDFs, read in 20-page batches. For web docs, follow all linked pages.
3. **Ask for test credentials** -- After reading the docs, ask the user for test credentials immediately. You'll need them for connection testing in Phase 4-5. Common credentials:
   - API keys / tokens
   - Merchant IDs
   - Hash keys / secrets
   - OAuth client ID + secret
   - Test environment URLs

4. **Classify the service type** -- This determines your architecture:

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

5. **Extract key details based on service type:**

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

6. **Identify the tricky parts** -- Things that are unique or non-obvious:
   - Custom encryption protocols
   - Fields that must be empty strings vs omitted
   - Multi-step workflows (auth then request, polling, callbacks)
   - Character encoding requirements
   - Date/time format quirks
   - Business logic validation rules
   - Test account setup requirements (track config, contract activation, etc.)

### Deliverable

A clear understanding of:
- How many tools to create (1 per logical operation, NOT necessarily 1 per endpoint)
- Which template modules to keep/delete/create
- What dependencies are needed
- What credentials/config the user needs
- Test credentials in hand (or know what to ask for)

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
├── Yes -> Does it use standard auth (Bearer/API key/OAuth)?
│   ├── Yes -> Keep rest_client.py + matching auth module
│   └── No  -> Keep rest_client.py + create custom auth module
└── No  -> What is it?
    ├── REST with custom encoding (form POST, encryption) -> Custom connector
    ├── GraphQL -> Keep graphql_client.py
    ├── RSS/Atom -> Keep rss_client.py
    ├── Web scraping -> Keep scraper_client.py
    ├── MQTT/IoT -> Keep mqtt_client.py
    └── SDK/Protocol/Other -> Custom connector
```

**Tool granularity:** Generally 1 tool per user-facing operation. Group related API calls into one tool if they always happen together (e.g., "get token then fetch data" = 1 tool).

**Optional field handling:** Some APIs expect ALL fields present (empty string for unused ones). Others want unused fields omitted. Check sample code in the reference docs. The template's `sample_tools.py` documents three patterns (A/B/C) for this.

---

## Phase 3: Plan

**Goal:** Break implementation into small, independent tasks with exact file paths and code.

### Task Structure

A typical MCP server has these tasks:

```
Task 1:  Clean up template (delete unused files)
Task 2:  Update project config (pyproject.toml, app.py, mcp_server.py, .env.example, .mcp.json)
Task 3:  Implement auth module (if custom auth needed)
Task 4:  Implement config/settings.py (endpoints/config, env vars)
Task 5:  Implement connector (service client)
Task 6:  Implement MCP tools (@mcp.tool() functions)
Task 7:  Test connection script
Task 8:  E2E tests for all tools
Task 9:  Documentation (README.md, README.zh-TW.md, CLAUDE.md, CONTRIBUTING.md, CHANGELOG.md)
Task 10: Final cleanup & verification
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

### Task 9: Documentation Details

Update ALL of these (not just READMEs):
- `README.md` -- Uncomment badges, fill in service details, add usage examples
- `README.zh-TW.md` -- Same in Traditional Chinese
- `CLAUDE.md` -- Update architecture diagram, patterns, conventions
- `CONTRIBUTING.md` -- Replace `{service}`, update connector import example
- `CHANGELOG.md` -- Reset with 0.1.0 initial release

### Task 10: Final Cleanup & Verification

This is critical. After implementation, audit the project for:
- Template `{service}` placeholders left in any file
- References to deleted modules (e.g., `rest_client`, `bearer`)
- Unused exception classes or functions (defined but never called)
- Stale TODO comments from the template
- `CONTRIBUTING.md` still showing template example code
- Any file with "example.com" or generic template content

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

1. **FastMCP not MCPServer** -- The template's `app.py` uses `FastMCP`:
   ```python
   from mcp.server.fastmcp import FastMCP
   mcp = FastMCP("mcp-your-service")
   ```
   If you see `MCPServer` anywhere, it's wrong. The `mcp` package only exports `FastMCP`.

2. **Pydantic FieldInfo in direct calls** -- When MCP tool functions use `Field(default=None)` and are called directly (not via MCP protocol), defaults remain as `FieldInfo` objects, not `None`. The template provides `_val()` in `sample_tools.py` -- **copy it to your tools file**:
   ```python
   from pydantic.fields import FieldInfo
   
   def _val(v, default=""):
       """Resolve a parameter value, handling FieldInfo defaults from direct calls."""
       if v is None or isinstance(v, FieldInfo):
           return default
       return v
   ```
   Use it everywhere you build parameter dicts: `"Field": _val(param)`.

3. **Empty string vs omitted fields** -- Some APIs require ALL fields present as empty strings. Others reject empty strings. Test with the actual service to determine behavior. The template's `sample_tools.py` documents three patterns:
   - **Pattern A:** `"Field": _val(param)` -- sends `""` if None (API wants all fields)
   - **Pattern B:** Only add to dict if not None (API rejects empty strings)
   - **Pattern C:** Conditionally include based on mode/flag

4. **pyproject.toml build backend** -- Template already has the correct one (`setuptools.build_meta`). Don't change it to the legacy backend.

5. **Python bytecode cache** -- After editing files, clear `__pycache__` if tests show stale behavior:
   ```bash
   find . -path ./.venv -prune -o -name "__pycache__" -type d -exec rm -rf {} +
   ```

6. **Test account setup** -- Some services need more than just credentials:
   - Invoice track configuration (e-invoice services)
   - Contract/agreement activation
   - Sandbox data seeding
   - Webhook URL registration
   
   If tests return unexpected errors, ask the user to verify their test account is fully configured.

### Test Against Live Service

Always test against the real service (test environment) during development. Real responses reveal:
- Validation requirements the docs don't mention
- Whether empty strings or missing fields are expected
- Actual error behavior
- Auth/encryption correctness

---

## Phase 5: Verify & Fix

**Goal:** All tests pass, all edge cases handled, no template leftovers.

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

### Template Cleanup Audit

After all tests pass, search the entire project for leftovers:

```bash
# Search for template placeholders
grep -r "{service}" --include="*.py" --include="*.md" --include="*.json" --include="*.toml" .

# Search for references to deleted modules
grep -r "rest_client\|rss_client\|scraper_client\|mqtt_client\|graphql_client" --include="*.py" --include="*.md" .
grep -r "bearer\|api_key\|oauth2\|from auth.none" --include="*.py" --include="*.md" .

# Search for template TODOs
grep -r "TODO" --include="*.py" --include="*.md" .

# Search for example.com
grep -r "example.com" --include="*.py" --include="*.md" .
```

Fix any findings. Common leftover locations:
- `CONTRIBUTING.md` -- still showing `{service}` or wrong connector import
- Docstrings mentioning exceptions that are never raised
- Functions defined but never called (dead code)

### Common Fixes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `ModuleNotFoundError` | Dependency not installed | `uv pip install <package>` |
| API field format errors | Sending FieldInfo objects | Add `_val()` helper |
| API rejects empty fields | API wants fields omitted | Don't include in payload |
| API rejects missing fields | API wants empty strings | Include as `""` |
| Stale code after edits | Python bytecode cache | Delete `__pycache__` dirs |
| Test account errors | Account not fully configured | Ask user to verify setup |

---

## Phase 6: Publish

**Goal:** Package on PyPI with automated release workflow.

The template already provides most of the publishing infrastructure. You mostly need to fill in placeholders.

### Step 1: Update pyproject.toml

The template already has the structure. Fill in:
- `name` -- your package name (e.g., `mcp-ezpay-einvoice`)
- `description` -- one-line description
- `keywords` -- relevant keywords
- `dependencies` -- add service-specific deps (e.g., `pycryptodome`)
- `[project.urls]` -- update GitHub URLs
- `[project.scripts]` -- update entry point name

### Step 2: Update publish.yml

The template already has `.github/workflows/publish.yml`. Just update the PyPI URL on the `url:` line:
```yaml
url: https://pypi.org/p/mcp-your-service
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

### Step 6: Uncomment Shield Badges

The template README has commented-out badges. Uncomment them and replace `{service}`:

```markdown
[![PyPI version](https://img.shields.io/pypi/v/mcp-your-service)](https://pypi.org/project/mcp-your-service/)
[![Python](https://img.shields.io/pypi/pyversions/mcp-your-service)](https://pypi.org/project/mcp-your-service/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![MCP](https://img.shields.io/badge/MCP-compatible-blue)](https://modelcontextprotocol.io/)
[![GitHub stars](https://img.shields.io/github/stars/org/mcp-your-service)](https://github.com/org/mcp-your-service/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/org/mcp-your-service)](https://github.com/org/mcp-your-service/issues)
[![GitHub last commit](https://img.shields.io/github/last-commit/org/mcp-your-service)](https://github.com/org/mcp-your-service/commits/main)
```

### Step 7: Push, Release, Verify

```bash
git push origin main
gh release create v0.1.0 --title "v0.1.0 - Initial Release" --notes "..."
gh run watch  # watch the publish workflow
```

---

## README Template

The template README already includes most sections. Customize these:

```
1. Title + shield badges (uncomment and fill in)
2. Language toggle (English / 繁體中文)
3. Overview (1 paragraph about YOUR service)
4. Features (YOUR tool list + technical highlights)
5. Prerequisites (YOUR service's credential requirements)
6. Installation (PyPI, uvx, source -- update {service} placeholders)
7. Configuration (YOUR env vars table)
8. Usage (update {service} in Claude Desktop/Code/Cursor configs)
9. Usage Examples (conversational format with real results)
10. Tools Reference (table with YOUR tools)
11. Error Codes Reference (from YOUR service's docs)
12. Architecture (update diagram for YOUR project)
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

Include 6-8 examples covering main workflows. Use real responses from your test runs.

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
- [ ] Test credentials in hand

### After Phase 2 (Design)
- [ ] User approved the design approach
- [ ] Design spec committed

### After Phase 4 (Implement)
- [ ] Template cleanup done (unused modules deleted)
- [ ] Auth module works (if applicable)
- [ ] Config has all endpoints/settings and env vars
- [ ] Connector handles the service protocol correctly
- [ ] All tools implemented with `_val()` helper and proper `Field()` descriptions
- [ ] Connection test script works
- [ ] E2E tests pass

### After Phase 5 (Verify)
- [ ] At least one tool returns SUCCESS with real data
- [ ] No field format or protocol errors
- [ ] FieldInfo handling is correct for direct calls
- [ ] No template `{service}` placeholders remaining in code
- [ ] No references to deleted template modules
- [ ] CONTRIBUTING.md updated with correct connector import
- [ ] No unused exception classes or dead code
- [ ] Git is clean

### After Phase 6 (Publish)
- [ ] pyproject.toml has correct package name, keywords, classifiers, urls
- [ ] publish.yml updated with correct PyPI package name
- [ ] PyPI trusted publisher configured
- [ ] GitHub environment "pypi" created
- [ ] Package published to PyPI
- [ ] GitHub repo metadata set (description, topics, homepage)
- [ ] README shield badges uncommented and updated
- [ ] README has conversational usage examples
- [ ] Both English and Chinese READMEs updated
- [ ] CONTRIBUTING.md updated
