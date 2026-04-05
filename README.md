# mcp-create

A Claude Code plugin that guides you through building production-ready [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers -- from reading service documentation to publishing on PyPI, in one session.

## Installation

### 1. Add the Asgard marketplace

```
/plugin marketplace add WJWang/wjwang-marketplace
```

### 2. Install the plugin

```
/plugin install mcp-create@wjwang-marketplace
```

## Usage

1. Clone the [Asgard MCP Server Template](https://github.com/asgard-ai-platform/mcp-template) as your working directory
2. Place your service documentation in the `reference/` folder (API specs, SDK docs, protocol guides, etc.)
3. In Claude Code:

```
Use the mcp-create skill to build an MCP server from this template.
The service docs are in the reference/ folder.
```

The skill walks through 7 phases:

| Phase | What happens |
|-------|-------------|
| 0. Understand the Template | Learn what the template provides |
| 1. Read & Understand Reference | Study your service documentation |
| 2. Design | Architecture decisions with your approval |
| 3. Plan | Break implementation into tasks |
| 4. Implement | Code, tests, documentation |
| 5. Verify & Fix | All tests passing, cleanup |
| 6. Publish | PyPI package + GitHub release |

## Prerequisites

- [Asgard MCP Server Template](https://github.com/asgard-ai-platform/mcp-template)
- Service reference material (API specs, SDK docs, protocol guides, platform manuals, etc.)
- Test credentials for the target service (if applicable)

## What It Produces

A complete MCP server project with:
- Working tools connected to the target service
- Appropriate auth and connector modules
- Unit tests and E2E tests passing against real API
- README in English and Traditional Chinese
- PyPI package published via GitHub Actions

## Real-World Example

This skill was validated by building [mcp-ezpay-einvoice](https://github.com/asgard-ai-platform/mcp-ezpay-einvoice) -- a 7-tool MCP server for Taiwan's ezPay e-invoice API, published to [PyPI](https://pypi.org/project/mcp-ezpay-einvoice/).

## Part of the Asgard Ecosystem

- [MCP Server Template](https://github.com/asgard-ai-platform/mcp-template) -- the template this skill works with
- [mcp-ezpay-einvoice](https://github.com/asgard-ai-platform/mcp-ezpay-einvoice) -- example project built with this skill
- [Asgard AI Platform](https://github.com/asgard-ai-platform) -- full ecosystem

## License

MIT
