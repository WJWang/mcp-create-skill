# Design: Claude Code Plugin Distribution for mcp-create-skill

**Date:** 2026-04-05
**Status:** Approved

## Summary

Restructure the `mcp-create-skill` repo from a standalone markdown skill file into a proper Claude Code plugin, publicly distributable via a marketplace repo. The MCP scaffolding server is a separate future project (Approach B).

## Current State

```
mcp-create-skill/
├── README.md          # Project overview
└── mcp-create.md      # 570-line skill workflow
```

## Target State

```
mcp-create-skill/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── mcp-create/
│       └── SKILL.md             # Skill content (migrated from mcp-create.md)
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-04-05-plugin-distribution-design.md
├── README.md                    # Rewritten for plugin installation
└── LICENSE                      # MIT
```

## Components

### 1. Plugin Manifest (`.claude-plugin/plugin.json`)

```json
{
  "name": "mcp-create",
  "version": "1.0.0",
  "description": "Guided workflow for building production-ready MCP servers from the Asgard MCP Template",
  "author": { "name": "neo" },
  "homepage": "https://github.com/WJWang/mcp-create-skill",
  "repository": "https://github.com/WJWang/mcp-create-skill",
  "license": "MIT",
  "keywords": ["mcp", "template", "server", "workflow", "python"]
}
```

### 2. Skill File (`skills/mcp-create/SKILL.md`)

- Migrated from `mcp-create.md` with proper YAML frontmatter added
- Content unchanged — the 7-phase workflow stays as-is
- Frontmatter:
  ```yaml
  ---
  name: mcp-create
  description: Use when building a new MCP server from the Asgard MCP Template. Guides the complete workflow from reading service documentation through design, implementation, testing, and PyPI publishing.
  ---
  ```

### 3. README (rewritten)

Focused on plugin distribution:
- What this plugin is
- Installation (marketplace add + plugin install)
- Usage (how to invoke in Claude Code)
- Prerequisites (mcp-template, service docs)
- Brief workflow overview (not the full guide)
- Links to mcp-template, example project, Asgard ecosystem

### 4. Root `mcp-create.md` — deleted

Replaced by `skills/mcp-create/SKILL.md`.

## Distribution: Marketplace Repo (separate project)

A separate GitHub repo `WJWang/wjwang-marketplace` with:

```
wjwang-marketplace/
└── .claude-plugin/
    └── marketplace.json
```

```json
{
  "name": "wjwang-marketplace",
  "description": "Asgard AI Platform plugins for Claude Code",
  "owner": { "name": "neo" },
  "metadata": {
    "description": "Asgard AI Platform plugins for building MCP servers",
    "version": "1.0.0"
  },
  "plugins": [
    {
      "name": "mcp-create",
      "version": "1.0.0",
      "description": "Guided workflow for building MCP servers from the Asgard MCP Template",
      "source": {
        "source": "url",
        "url": "https://github.com/WJWang/mcp-create-skill.git"
      }
    }
  ]
}
```

User install flow:
```bash
/plugin marketplace add WJWang/wjwang-marketplace
/plugin install mcp-create@wjwang-marketplace
```

## Out of Scope

- MCP scaffolding server (separate future project)
- Creating the marketplace repo (user creates on GitHub; content provided here)
- Multi-platform support (.cursor-plugin, .opencode, .codex) — can be added later
- Hooks or agents — not needed for this skill
- CLAUDE.md — unnecessary for a single-skill plugin
- `.claude/settings.local.json` — local dev permissions, unrelated to plugin structure

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Distribution method | Custom marketplace | Full control, extensible for future plugins |
| Skill content changes | None | Already battle-tested and comprehensive |
| Plugin scope | Skill only | MCP server is separate project (Approach B) |
| Primary audience | Claude Code developers | Claude Desktop as nice-to-have |
