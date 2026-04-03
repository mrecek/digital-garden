# AGENTS.md

> **PUBLIC REPOSITORY** — All content in this vault is published to GitHub at [mrecek/digital-garden](https://github.com/mrecek/digital-garden) and visible to anyone on the internet.

This is Mark Recek's digital garden — a public collection of research, notes, and explorations. Content originates from private vaults and is published here after review.

## Prohibited Content

Never commit any of the following to this repository:

- Secrets, credentials, API keys, tokens
- Internal or corporate URLs, hostnames, IP addresses
- Employer-proprietary information or work product
- Personal or private data (addresses, phone numbers, account details)
- References to private vaults, private notes, or internal systems
- Wikilinks that reference notes not present in this vault

## Content Routing

See `README.md` for folder purposes. `Templates/` contains Obsidian templates — not content; do not modify without asking.

## Frontmatter Conventions

All notes use YAML frontmatter. Every field below is required unless marked optional.

```yaml
title: "Note Title"
tags:
  - topic
created: YYYY-MM-DD
stage: seedling | budding | evergreen
origin: ai-generated | ai-assisted | human
```

| Field | Values | Purpose |
|-------|--------|---------|
| `title` | free text | Display title |
| `tags` | list | Topic classification |
| `created` | `YYYY-MM-DD` | Date first published |
| `stage` | `seedling`, `budding`, `evergreen` | Content maturity — seedling (rough/early), budding (has substance, still evolving), evergreen (mature/stable) |
| `origin` | `ai-generated`, `ai-assisted`, `human` | How the content was produced — ai-generated (AI wrote the bulk), ai-assisted (human-written with AI help), human (all human) |

Optional fields: `source` (URL for content with an external origin), `updated` (date of last significant revision).

## Link Rules

- Wikilinks (`[[Note Name]]`) are fine within this vault
- Every wikilink must resolve to a file that exists in this repo
- No cross-vault links — if a link target doesn't exist here, replace it with plain text or remove it

## Publish Checklist

When copying content from a private vault:

1. Copy the file into the appropriate folder
2. Ensure frontmatter conforms to the standard above — set `stage`, `origin`, normalize `date` → `created`, remove vault-specific fields (`aliases`, `related`, `projects`)
3. Strip frontmatter links pointing to private notes
4. Scan body for wikilinks — replace any `[[Private Note]]` with plain text or remove
5. Remove work-sensitive, personal, or proprietary content (see Prohibited Content above)
6. Verify the note is self-contained and reads well standalone
7. Review the full diff before committing

## Change Boundaries

- Do not delete notes unless explicitly asked
- Do not change note taxonomy or frontmatter schema unless explicitly asked
- Do not create new top-level folders unless explicitly asked
