---
id: P001
name: rfc-process
date: 2026-06-21
author: Julio Capote
used_in: ['github.com/boldblackai/harness']
---

# RFC Process

A lightweight Request-for-Comments workflow for proposing, tracking, and archiving significant changes. It gives every meaningful decision a durable, reviewable home — and, crucially, gives an agent a clear place to look to understand *why* something exists and *whether* it's still current.

## When to use

- Any project where significant changes, architectural decisions, or new features should be written down and agreed on before they're built.
- Agent-driven workflows especially benefit: the AGENTS.md section tells the agent what counts as "significant" and the exact lifecycle to follow, so it can create, advance, and close out RFCs without further prompting.

## How it works

1. Drop the **AGENTS.md** section below into your repo's agent instructions. This is the contract the agent follows.
2. Seed the `rfcs/` directory with the **template** so new proposals start from a known shape.
3. Authors (human or agent) create a new RFC per the naming convention, fill it in, and move it through the status lifecycle: `Proposed` → `Accepted` → `Implemented` (or `Rejected`).
4. When an RFC reaches `Implemented`, the agent updates `AGENTS.md` and `README.md` to reflect any new infrastructure/commands/workflows and swaps the implementation checklist for implementation notes.

## The AGENTS.md section

This is the authoritative description of the process. Place it in `AGENTS.md` (or wherever your agent reads instructions).

```markdown title="AGENTS.md"
## RFCs

Significant changes, architectural decisions, and new features should be proposed as RFCs in the `rfcs/` directory. RFCs use the format `rfcs/YYYY-MM-DD_short_title.md` with the following structure:

- `# Title` — short descriptive title
- `**Date:**` — proposal date (ISO format)
- `**Status:**` — `Proposed`, `Accepted`, `Implemented`, or `Rejected`
- `## Goal` — what the RFC aims to accomplish
- Remaining sections are free-form but typically include motivation, technical details, migration notes, and an implementation checklist
- When moving an RFC to `Implemented`, update `AGENTS.md` and `README.md` to reflect any new infrastructure, commands, or workflows introduced by the RFC. Also, replace the implementation checklist with implementation notes.
```

## RFC template

Copy this into `rfcs/` and rename to `YYYY-MM-DD_short_title.md` to start a new proposal.

```markdown title="rfcs/YYYY-MM-DD_short_title.md"
# Title

**Date:** YYYY-MM-DD
**Status:** Proposed

## Goal

What this RFC aims to accomplish.

## Motivation

Why this change is needed.

## Technical Details

How it will work.

## Migration Notes

Anything required to roll this out safely.

## Implementation Checklist

- [ ] Task one
- [ ] Task two
```
