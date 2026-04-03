---
title: "AI Agent Config File Standards: AGENTS.md vs CLAUDE.md"
tags:
  - ai-agents
  - developer-tools
  - standards
  - claude-code
created: 2026-04-03
stage: evergreen
origin: ai-generated
---

# AI Agent Config File Standards: AGENTS.md vs CLAUDE.md

Research into best practices for managing AI coding agent configuration files across multiple tool harnesses (Claude Code, Codex, Cursor, Windsurf, GitHub Copilot, Aider).

## AGENTS.md as the Universal Standard

AGENTS.md has emerged as the dominant standard for AI coding agent instructions. It was formalized by OpenAI for Codex CLI in August 2025, then donated to the Linux Foundation's [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation) in December 2025. Anthropic, OpenAI, and Block are co-founders of AAIF. As of early 2026, 60,000+ open-source repos contain AGENTS.md.

## Tool Support Matrix

| Tool | Reads AGENTS.md? | Native config file |
|---|---|---|
| OpenAI Codex | Yes (originator) | `AGENTS.md` |
| Cursor | Yes | `.cursor/rules/*.mdc` |
| Windsurf | Yes | `.windsurfrules` |
| GitHub Copilot | Yes | `.github/copilot-instructions.md` |
| Aider | Yes | `CONVENTIONS.md` |
| **Claude Code** | **No** | `CLAUDE.md` |

> [!note]
> Claude Code is the only major tool that does not natively read AGENTS.md, despite Anthropic co-founding the foundation that governs it. This is the [#1 requested feature](https://github.com/anthropics/claude-code/issues/6235) (3,000+ upvotes).

## Approaches: Symlink vs Reference

### Symlink (`ln -s AGENTS.md CLAUDE.md`)

- True single source of truth with zero content drift
- Git tracks symlinks natively
- Recommended by [SSW Rules](https://www.ssw.com.au/rules/symlink-agents-to-claude) and community guides
- **Downsides**: loses ability to have Claude-specific instructions; potential double-loading if Claude Code adds native AGENTS.md support; Windows requires `core.symlinks=true`

### Reference (`CLAUDE.md` says "Read AGENTS.md")

- Allows Claude-specific overrides alongside shared AGENTS.md content
- More explicit about architecture for developers reading the repo
- Trivial maintenance cost (CLAUDE.md is ~2 lines)
- **Downsides**: CLAUDE.md still exists as a file; relies on agent interpreting the instruction correctly (which it does reliably)

### Recommendation

**Reference approach is preferred** when you may need Claude-specific instructions or when you want a clean migration path. When Claude Code eventually adds native AGENTS.md support, you can delete CLAUDE.md with zero effort. A symlink would require cleanup to avoid double-loading.

**Symlink approach is preferred** for teams with heavy multi-tool usage and no need for tool-specific overrides.

## Cross-Agent Directory Layout

The emerging standard for agent assets:

```text
repo/
├── AGENTS.md                  # Universal agent instructions
├── CLAUDE.md                  # Thin shim: "Read AGENTS.md"
├── .agents/                   # Cross-agent standard
│   ├── skills/                # Reusable agent skills
│   └── commands/              # Agent commands
├── .claude/                   # Claude Code auto-discovery
│   ├── skills/ → ../.agents/skills/
│   └── commands/ → ../.agents/commands/
```

The `.agents/skills/` directory is defined by the [Agent Skills standard](https://agentskills.io), created by Anthropic. Tools supporting it include Codex CLI, Gemini CLI, Cursor, Kiro, and OpenCode. Claude Code still uses `.claude/skills/`, hence the symlink bridge.

## Notable Tools

- **[Ruler](https://github.com/intellectronica/ruler)**: Define instructions once in `.ruler/`, run `ruler apply` to distribute to all agent config files. Supports AGENTS.md, CLAUDE.md, `.cursorrules`, `.windsurfrules`, `copilot-instructions.md`, and more.
- **[0xdevalias tracking gist](https://gist.github.com/0xdevalias/f40bc5a6f84c4c5ad862e314894b2fa6)**: Living document tracking all AI agent rule/instruction file formats.

## Sources

- [AGENTS.md official site](https://agents.md/)
- [AGENTS.md spec repo](https://github.com/agentsmd/agents.md)
- [OpenAI Codex AGENTS.md docs](https://developers.openai.com/codex/guides/agents-md)
- [Claude Code feature request #6235](https://github.com/anthropics/claude-code/issues/6235)
- [GitHub Copilot AGENTS.md support](https://github.blog/changelog/2025-08-28-copilot-coding-agent-now-supports-agents-md-custom-instructions/)
- [Linux Foundation AAIF announcement](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [SSW: Symlink AGENTS.md to .claude](https://www.ssw.com.au/rules/symlink-agents-to-claude)
