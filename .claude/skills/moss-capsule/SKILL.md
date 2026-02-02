---
name: moss-capsule
description: Use when storing, fetching, searching, or managing context capsules. Helps with session handoffs, multi-agent coordination, or preserving decisions across sessions.
---

# Capsules: Context Snapshots for AI Session Handoffs

> **Full examples:** [examples.md](examples.md) | **Reference:** [reference.md](reference.md)

Capsules are **distilled context snapshots** for preserving working state across sessions, tools, or agents. Not a chat log—a structured summary of what matters.

## When NOT to Use Capsules

- **Ephemeral debugging** — Use scratch files or logs, not capsules
- **Real-time state** — Capsules are for handoffs, not live sync between agents
- **Large data** — 12K char limit; store references to files, not file contents
- **Chat transcripts** — Distill first; capsules are summaries, not logs
- **Temporary single-session notes** — Just keep them in context; no need to persist

## Quick Reference

| Task | Command |
|------|---------|
| Store new | `capsule_store(workspace, name, capsule_text)` |
| Overwrite | `capsule_store(name, mode: "replace", capsule_text)` |
| Get latest | `capsule_latest(workspace)` then `capsule_fetch(workspace, name)` |
| Browse | `capsule_list(workspace)` or `capsule_inventory()` |
| Search | `capsule_search(query: "JWT OR auth*")` — full-text search |
| Multi-fetch | `capsule_fetch_many(items: [{workspace, name}, ...])` |
| Combine | `capsule_compose(items: [...], format: "markdown")` |
| Bulk update | `capsule_bulk_update(workspace, set_phase: "archived")` — update metadata by filter |
| Bulk delete | `capsule_bulk_delete(workspace, tag, ...)` — filter-based soft delete |
| Delete | `capsule_delete(workspace, name)` — soft delete, recoverable |

## Before Storing: Distill First

Before calling `capsule_store`, distill your context:

> "Distill into a capsule under 12,000 chars. State not story. Must include: Objective, Current status, Decisions/Constraints, Next actions, Key locations, Open questions/Risks. Max 1-3 tiny code snippets only if critical. If too long, compress—do not omit decisions or next actions."

`capsule_store` saves the result. It doesn't summarize for you.

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
capsule_fetch(id: "01HX...", workspace: "default", name: "auth")

# CORRECT
capsule_fetch(id: "01HX...")
capsule_fetch(workspace: "default", name: "auth")
```

## Output Bloat Rules

| Tool | Returns `capsule_text`? |
|------|------------------------|
| `capsule_fetch` | Yes (default), No with `include_text: false` |
| `capsule_fetch_many` | Yes (default), No with `include_text: false` |
| `capsule_compose` | **Always** (returns `bundle_text`) |
| `capsule_latest` | **No** (default), Yes with `include_text: true` |
| `capsule_list` | **Never** |
| `capsule_inventory` | **Never** |

**Pattern:** Use `capsule_list`/`capsule_inventory` to browse, then `capsule_fetch` to load specific capsules.

## When to Skip Validation

Use `allow_thin: true` **only** for:
- Quick scratch notes
- Temporary debugging context
- Notes you'll expand later

**Avoid** for anything persisted or shared between agents.

## Multi-Agent Orchestration

Use `run_id`, `phase`, `role` to coordinate:

```
capsule_store(name: "design", run_id: "task-1", phase: "design", role: "architect", ...)
capsule_latest(run_id: "task-1", phase: "review")
capsule_list(workspace: "default", run_id: "task-1")
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
| `CANCELLED` | Operation cancelled (context timeout) |

**Note on `mode: "replace"`:** Only overwrites *active* capsules. If a capsule was soft-deleted, `replace` creates a new one (doesn't revive the deleted). To recover deleted capsules, use `capsule_export(include_deleted: true)` then `capsule_import`.

See [reference.md](reference.md) for full error details.

## Additional Resources

- [reference.md](reference.md) — Section synonyms, name normalization, backup/restore, limits, tips
- [examples.md](examples.md) — Store, fetch, update, orchestration examples
