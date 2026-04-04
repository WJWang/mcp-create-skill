# MCP Server Creation Skill

A structured AI skill for building production-ready [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers -- from reading an API spec to publishing on PyPI, in one session.

## What This Is

This is a **skill file** (prompt/workflow guide) designed to be used by AI coding assistants (Claude Code, etc.) when building MCP servers from the [Asgard MCP Server Template](https://github.com/asgard-ai-platform/mcp-template). It encodes the complete process, common pitfalls, and hard-won lessons from building real MCP servers.

Think of it as a runbook that an AI assistant follows to turn your API documentation into a working, published MCP server.

## The Problem It Solves

Building an MCP server involves many steps that are easy to get wrong:
- Reading API docs and deciding on architecture
- Choosing the right auth module and connector from the template
- Handling Pydantic `FieldInfo` gotchas in tool parameter defaults
- Knowing whether an API expects empty strings or omitted fields
- Setting up PyPI publishing with trusted publishers
- Writing proper README with install methods for Claude Desktop, Claude Code, Cursor, etc.

This skill captures all of these lessons so the AI doesn't have to rediscover them each time.

## How It Works

The skill follows 7 phases:

```
Phase 0: Understand the Template   ->  Know what's available
Phase 1: Read & Understand Ref     ->  Read API docs / service docs
Phase 2: Design                    ->  Architecture decisions
Phase 3: Plan                      ->  Implementation tasks
Phase 4: Implement                 ->  Code, tests, docs
Phase 5: Verify & Fix              ->  Tests passing, cleanup
Phase 6: Publish                   ->  PyPI + GitHub release
```

## Prerequisites

- [Asgard MCP Server Template](https://github.com/asgard-ai-platform/mcp-template) -- the template this skill is designed to work with
- API documentation or service reference material for the target service
- Test credentials for the target service (if applicable)

## Usage

### With Claude Code

Place `mcp-create.md` where your AI assistant can access it (e.g., in a skills directory or reference it directly), then:

> "Use the mcp-create skill to build an MCP server from this template. The API docs are in the reference/ folder."

The AI will follow the 7-phase workflow: read the docs, design the architecture, plan tasks, implement, verify, and publish.

### What It Produces

A complete MCP server project with:
- Working tools that connect to the target service's API
- AES encryption, OAuth, or whatever auth the service requires
- Unit tests and E2E tests passing against real API
- README in English and Traditional Chinese with usage examples
- PyPI package published via GitHub Actions
- Shield badges, GitHub topics, and repo metadata configured

## Template Compatibility

This skill is designed for the [Asgard MCP Server Template](https://github.com/asgard-ai-platform/mcp-template), which provides:

| Component | What the template provides |
|-----------|---------------------------|
| **Auth modules** | Bearer token, API key, OAuth 2.0, No auth |
| **Connectors** | REST, RSS, Scraper, MQTT, GraphQL |
| **Tools** | Sample tools with `_val()` helper for FieldInfo handling |
| **Publishing** | GitHub Actions workflow for PyPI |
| **Config** | `pyproject.toml` with PyPI metadata placeholders |
| **Docs** | README templates with commented-out shield badges |

The skill guides you through choosing which modules to keep, which to delete, and when to create custom ones.

## Real-World Example

This skill was developed and validated by building [mcp-ezpay-einvoice](https://github.com/asgard-ai-platform/mcp-ezpay-einvoice) -- a 7-tool MCP server for Taiwan's ezPay e-invoice API, published to [PyPI](https://pypi.org/project/mcp-ezpay-einvoice/). The entire project was completed in a single session using this workflow.

## Key Lessons Encoded

Pitfalls discovered during real builds that this skill prevents:

| Pitfall | What happens | Skill's guidance |
|---------|-------------|-----------------|
| `MCPServer` import | `ModuleNotFoundError` -- class doesn't exist | Use `FastMCP` from `mcp.server.fastmcp` |
| Pydantic `FieldInfo` | Optional params serialize as `FieldInfo(...)` instead of `None` | Copy `_val()` helper from template's `sample_tools.py` |
| Empty vs omitted fields | API rejects request with cryptic field format errors | Test with real API; template documents 3 patterns |
| Legacy build backend | `python -m build` fails, can't publish to PyPI | Template uses `setuptools.build_meta` |
| Template leftovers | `{service}` placeholders, wrong imports in CONTRIBUTING.md | Phase 5 includes grep audit commands |
| Test account setup | API returns "no contract" even with credentials | Skill reminds to verify account configuration |

## File Structure

```
mcp-create-skill/
├── README.md          # This file
└── mcp-create.md      # The skill (7-phase workflow)
```

## License

MIT

## Part of the Asgard Ecosystem

- [MCP Server Template](https://github.com/asgard-ai-platform/mcp-template) -- the template this skill works with
- [mcp-ezpay-einvoice](https://github.com/asgard-ai-platform/mcp-ezpay-einvoice) -- example project built with this skill
- [Asgard AI Platform](https://github.com/asgard-ai-platform) -- full ecosystem
