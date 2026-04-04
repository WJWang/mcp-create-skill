# MCP Server Creation Skill

A structured workflow for building production-ready MCP (Model Context Protocol) servers from an API specification to a published PyPI package. Based on real-world experience building Taiwan's ezPay e-invoice MCP server.

---

## When to Use

- Building a new MCP server that wraps an external API
- Converting an API specification (PDF, OpenAPI, docs) into AI-callable tools
- Using the Asgard MCP server template as a starting point

## Overview

The process follows 6 phases, each with clear deliverables:

```
Phase 1: Read & Understand API  -->  API knowledge (endpoints, auth, params, errors)
Phase 2: Design                 -->  Design spec document
Phase 3: Plan                   -->  Implementation plan with tasks
Phase 4: Implement              -->  Working code, tests, docs
Phase 5: Verify & Fix           -->  All tests passing against live API
Phase 6: Publish                -->  PyPI package + GitHub release
```

---

## Phase 1: Read & Understand the API

**Goal:** Build complete mental model of the API before writing any code.

### Steps

1. **Locate the API documentation** -- PDF, web docs, OpenAPI spec, or reference files in the project
2. **Read the ENTIRE spec** -- Do not skim. Read every page. For PDFs, read in page batches (20 pages at a time)
3. **Extract these key details:**

| Detail | What to capture |
|--------|----------------|
| **Endpoints** | URL paths, HTTP methods, test vs production base URLs |
| **Authentication** | Auth mechanism (Bearer, API key, OAuth, custom encryption, etc.) |
| **Request format** | JSON body, form POST, multipart, custom encoding |
| **Parameters per endpoint** | Name, type, required/optional, valid values, constraints |
| **Response format** | Success/error shapes, nested result parsing |
| **Error codes** | All documented error codes and their meanings |
| **Versioning** | API version per endpoint if they differ |
| **Encryption/signing** | Any custom crypto (AES, HMAC, signatures, checksums) |
| **Business rules** | Date periods, calculation formulas, validation rules |
| **Code samples** | Reference implementations in any language (PHP, C#, etc.) |

4. **Identify the unique/tricky parts** -- What makes this API different from standard REST? Examples:
   - Custom encryption protocols (AES-256-CBC encrypted form POST)
   - Non-standard auth (not Bearer/API key)
   - Fields that must be present as empty strings vs omitted
   - URL-encoded query string payloads
   - CheckCode/signature verification on responses

### Deliverable

A clear understanding of:
- How many tools/endpoints to expose (typically 1 tool per API endpoint)
- What connector pattern to use (REST, custom encrypted POST, GraphQL, etc.)
- What auth module to use or create
- What dependencies are needed (e.g., pycryptodome for AES)

---

## Phase 2: Design

**Goal:** Decide architecture and get user approval before coding.

### Steps

1. **Assess the template** -- Determine which template modules to keep, delete, or create:
   - Which auth module? (bearer, api_key, oauth2, none, or custom)
   - Which connector? (rest_client, or custom)
   - What tool modules to create?

2. **Propose 2-3 approaches** with trade-offs. Common patterns:

   | Approach | When to use |
   |----------|-------------|
   | **Minimal custom connector** | API has unique protocol (encryption, custom auth, non-REST) |
   | **Shim existing REST connector** | API is standard REST with minor tweaks |
   | **Single-file monolith** | Very simple API with 1-3 endpoints |

3. **Present the design** covering:
   - Files to delete (unused template modules)
   - Files to create (new auth, connector, tools)
   - Files to modify (app.py, mcp_server.py, config, pyproject.toml)
   - Data flow diagram (request -> encrypt -> POST -> parse response)
   - Tool list with key parameters per tool
   - Error handling strategy

4. **Write design spec** to `docs/superpowers/specs/YYYY-MM-DD-<name>-design.md`

### Key Architecture Decisions

**Connector choice:** If the API uses ANY of these, create a custom connector instead of using rest_client.py:
- Custom encryption/signing of request body
- Form POST instead of JSON
- Non-standard response parsing
- No standard auth headers

**Tool granularity:** Generally 1 tool per API endpoint. Don't combine multiple endpoints into one tool (violates single responsibility). Don't split one endpoint into multiple tools unless the parameter sets are radically different.

**Optional field handling:** Some APIs expect ALL fields present (empty string for unused ones). Others want unused fields omitted. Check the API docs and sample code carefully. This is a common source of bugs.

---

## Phase 3: Plan

**Goal:** Break implementation into small, independent tasks with exact file paths and code.

### Task Structure

A typical MCP server has these tasks in order:

```
Task 1: Clean up template (delete unused files)
Task 2: Update project config (pyproject.toml, app.py, mcp_server.py, .env.example, .mcp.json)
Task 3: Implement auth/crypto module (if custom auth needed)
Task 4: Implement config/settings.py (endpoints, env vars)
Task 5: Implement connector (API client)
Task 6: Implement MCP tools (the actual @mcp.tool() functions)
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
- Task 1 (cleanup) + Task 2 (config) + Task 4 (settings.py)
- Task 3 (crypto) + Task 5 (connector) -- if crypto doesn't depend on connector
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

3. **Empty string vs omitted fields** -- Some APIs (like ezPay) require ALL fields present as empty strings. Others reject empty strings. Test with the actual API to determine behavior. Check the API's sample code in PHP/C# for guidance.

4. **pyproject.toml build backend** -- Use `setuptools.build_meta` (not `setuptools.backends._legacy:_Backend`) for PyPI publishing:
   ```toml
   [build-system]
   requires = ["setuptools>=68.0"]
   build-backend = "setuptools.build_meta"
   ```

5. **Python bytecode cache** -- After editing files, clear `__pycache__` if tests show stale behavior:
   ```bash
   find . -path ./.venv -prune -o -name "__pycache__" -type d -exec rm -rf {} +
   ```

### Test Against Live API

Always test against the real API (test environment) during development. API responses reveal:
- Field validation requirements the docs don't mention
- Whether empty strings or missing fields are expected
- Actual error code behavior
- Encryption/signing correctness

---

## Phase 5: Verify & Fix

**Goal:** All tests pass, all edge cases handled.

### Verification Checklist

```
[ ] Crypto/auth unit tests pass
[ ] Connection test passes (env vars + encryption + API connectivity)
[ ] All tool E2E tests pass (no Python exceptions)
[ ] All tools get valid API responses (even error codes prove connectivity)
[ ] At least one tool returns SUCCESS with real data
[ ] MCP server starts without errors
[ ] Git working tree is clean
```

### Common Fixes Needed

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `ModuleNotFoundError: Crypto` | pycryptodome not installed | `uv pip install pycryptodome` |
| `MCPServer not found` | Wrong import | Use `FastMCP` from `mcp.server.fastmcp` |
| API field format errors | Sending FieldInfo objects | Add `_val()` helper for all params |
| API rejects empty optional fields | API wants fields omitted | Don't include in post_data |
| API rejects missing optional fields | API wants empty strings | Include as `""` |
| Stale code after edits | Python bytecode cache | Delete `__pycache__` dirs |

---

## Phase 6: Publish

**Goal:** Package on PyPI with automated release workflow.

### Step 1: Update pyproject.toml for PyPI

Add these fields:

```toml
[project]
name = "mcp-your-service"
version = "0.1.0"
description = "MCP server wrapping the ... API into N AI Agent-callable tools for ..."
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

Include these sections in README:

1. **PyPI install** -- `pip install mcp-your-service`
2. **uvx** -- `uvx mcp-your-service` (no install needed)
3. **Source install** -- `git clone` + `uv pip install -e .`
4. **Claude Desktop** -- JSON config (uvx variant + pip variant)
5. **Claude Code** -- Auto-discovery + `claude mcp add`
6. **Cursor / Windsurf** -- `.cursor/mcp.json` config
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

A good MCP server README has these sections in order:

```
1. Title + shield badges
2. Language toggle link (English / 繁體中文)
3. Overview (1 paragraph)
4. Features (tool list + technical highlights)
5. Prerequisites
6. Installation (PyPI, uvx, source)
7. Configuration (env vars table)
8. Usage (Claude Desktop, Claude Code, Cursor, direct run, testing)
9. Usage Examples (conversational format with real API results)
10. Tools Reference (table with key params)
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

Include 6-8 examples covering the main workflows. Use real API responses when possible.

---

## Checklist

### Before starting
- [ ] API documentation is available and readable
- [ ] Test credentials are obtained
- [ ] Template project is set up

### After Phase 2 (Design)
- [ ] User approved the design approach
- [ ] Design spec is committed

### After Phase 4 (Implement)
- [ ] All template cleanup done
- [ ] Auth/crypto module works (unit tests pass)
- [ ] Config has all endpoints and env vars
- [ ] Connector handles the API protocol correctly
- [ ] All tools implemented with proper Field() descriptions
- [ ] Connection test script works
- [ ] E2E tests pass

### After Phase 5 (Verify)
- [ ] At least one tool returns SUCCESS from live API
- [ ] No field format errors from API
- [ ] FieldInfo handling is correct for direct calls
- [ ] Git is clean

### After Phase 6 (Publish)
- [ ] pyproject.toml has PyPI metadata (keywords, classifiers, urls)
- [ ] GitHub Actions workflow created
- [ ] PyPI trusted publisher configured
- [ ] GitHub environment "pypi" created
- [ ] Package published to PyPI
- [ ] GitHub repo description, topics, homepage set
- [ ] README has all install methods (PyPI, uvx, Claude Desktop, Claude Code, Cursor)
- [ ] README has shield badges
- [ ] README has conversational usage examples
- [ ] Both English and Chinese READMEs updated
