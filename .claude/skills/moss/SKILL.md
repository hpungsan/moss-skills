---
name: moss
description: Use when storing, fetching, or managing context capsules. Helps with session handoffs, multi-agent coordination, or preserving decisions across sessions.
---

# Moss: Context Capsules for AI Session Handoffs

Moss stores **distilled context snapshots** (capsules) for preserving working state across sessions, tools, or agents. Not a chat log—a structured summary of what matters.

## When NOT to Use Moss

- **Ephemeral debugging** — Use scratch files or logs, not capsules
- **Real-time state** — Moss is for handoffs, not live sync between agents
- **Large data** — 12K char limit; store references to files, not file contents
- **Chat transcripts** — Distill first; capsules are summaries, not logs
- **Temporary single-session notes** — Just keep them in context; no need to persist

## Quick Reference

| Task | Command |
|------|---------|
| Store new | `store(workspace, name, capsule_text)` |
| Overwrite | `store(name, mode: "replace", capsule_text)` |
| Get latest | `latest(workspace)` then `fetch(workspace, name)` |
| Browse | `list(workspace)` or `inventory()` |
| Multi-fetch | `fetch_many(items: [{workspace, name}, ...])` |
| Combine | `compose(items: [...], format: "markdown")` |
| Delete | `delete(workspace, name)` — soft delete, recoverable |

## Before Storing: Distill First

Before calling `store`, distill your context:

> "Distill into a Moss capsule under 12,000 chars. State not story. Must include: Objective, Current status, Decisions/Constraints, Next actions, Key locations, Open questions/Risks. Max 1-3 tiny code snippets only if critical. If too long, compress—do not omit decisions or next actions."

Moss stores the result. It doesn't summarize for you.

## Capsule Format

**6 required sections** (validated unless `allow_thin: true`):

```markdown
## Objective
What you're trying to accomplish (1-2 sentences)

## Status
What's done, in progress, or blocked

## Decisions
Key choices made and rationale

## Next actions
- Concrete next steps (bulleted)

## Key locations
- `src/auth/login.ts:45` - auth entry point

## Open questions
- Unresolved issues or uncertainties
```

See [reference.md](reference.md) for section synonyms and format variations.

## Addressing Modes

Pick ONE (mutually exclusive):

| Mode | Params | Use when |
|------|--------|----------|
| By ID | `id: "01HX..."` | Exact lookup, returned from store |
| By name | `workspace` + `name` | Human-friendly, memorable |

```
# WRONG - causes AMBIGUOUS_ADDRESSING error
fetch(id: "01HX...", workspace: "default", name: "auth")

# CORRECT
fetch(id: "01HX...")
fetch(workspace: "default", name: "auth")
```

## Output Bloat Rules

| Tool | Returns `capsule_text`? |
|------|------------------------|
| `fetch` | Yes (default), No with `include_text: false` |
| `fetch_many` | Yes (default), No with `include_text: false` |
| `compose` | **Always** (returns `bundle_text`) |
| `latest` | **No** (default), Yes with `include_text: true` |
| `list` | **Never** |
| `inventory` | **Never** |

**Pattern:** Use `list`/`inventory` to browse, then `fetch` to load specific capsules.

## When to Skip Validation

Use `allow_thin: true` **only** for:
- Quick scratch notes
- Temporary debugging context
- Notes you'll expand later

**Avoid** for anything persisted or shared between agents.

## Multi-Agent Orchestration

Use `run_id`, `phase`, `role` to coordinate:

```
store(name: "design", run_id: "task-1", phase: "design", role: "architect", ...)
latest(run_id: "task-1", phase: "review")
list(workspace: "default", run_id: "task-1")
```

See [examples.md](examples.md) for orchestration patterns.

## Error Quick Reference

| Error Code | Fix |
|------------|-----|
| `NOT_FOUND` | Check name/workspace spelling |
| `NAME_ALREADY_EXISTS` | Use `mode: "replace"` to overwrite |
| `AMBIGUOUS_ADDRESSING` | Use only id OR workspace+name |
| `CAPSULE_TOO_LARGE` | Distill further (limit: 12,000 chars) |
| `CAPSULE_TOO_THIN` | Add missing sections, or `allow_thin: true` |
| `COMPOSE_TOO_LARGE` | Fewer items, or trim source capsules |

**Note on `mode: "replace"`:** Only overwrites *active* capsules. If a capsule was soft-deleted, `replace` creates a new one (doesn't revive the deleted). To recover deleted capsules, use `export(include_deleted: true)` then `import`.

See [reference.md](reference.md) for full error details.

## Additional Resources

- [reference.md](reference.md) — Section synonyms, name normalization, backup/restore, limits, tips
- [examples.md](examples.md) — Store, fetch, update, orchestration examples
