# Moss Reference

Detailed reference for Moss context capsules.

## Section Synonyms

Section names are case-insensitive. These synonyms are accepted:

| Canonical | Also accepted |
|-----------|---------------|
| Objective | Goal, Purpose |
| Status | Current status, State, Where we are |
| Decisions | Decisions/constraints, Constraints, Choices |
| Next actions | Next steps, Action items, Todo, Tasks |
| Key locations | Locations, Files, Paths, References |
| Open questions | Questions, Risks, Unknowns |

**Formats accepted:** Markdown headers (`## Section`), colon-style (`Section:`), or JSON keys.

## Name Normalization

Names are normalized: lowercased, whitespace collapsed.

```
"Auth System"     → "auth system"
"  My Project  "  → "my project"
"LOUD_NAME"       → "loud_name"
```

Fetch using either raw or normalized form—both work.

## Multi-Agent Orchestration Fields

| Field | Purpose | Example values |
|-------|---------|----------------|
| `run_id` | Groups capsules for one task | `"pr-review-123"`, `"feature-xyz"` |
| `phase` | Workflow stage | `"design"`, `"implement"`, `"review"`, `"verified"` |
| `role` | Agent type | `"architect"`, `"coder"`, `"qa-reviewer"` |

Query by orchestration:
```
latest(run_id: "feature-xyz", phase: "review", role: "qa")
list(workspace: "default", run_id: "feature-xyz")
inventory(phase: "review")  # All review-phase capsules globally
search(query: "JWT", run_id: "feature-xyz")  # FTS5 search within run
```

## Batch Operations

Several tools handle multiple capsules—they have different behaviors:

| Tool | Behavior | Use when |
|------|----------|----------|
| `search` | **Ranked results** — FTS5 full-text search with snippets | Find capsules by content |
| `fetch_many` | **Partial success** — returns found items + errors array | You need whatever's available |
| `compose` | **All-or-nothing** — fails on first missing | You need ALL items or none |
| `bulk_update` | **Filter-based** — updates all matching capsules | Batch metadata changes |
| `bulk_delete` | **Filter-based** — soft-deletes all matching | Batch cleanup |

**fetch_many** returns:
```json
{
  "items": [/* found capsules */],
  "errors": [{ "ref": {...}, "code": "NOT_FOUND", "message": "..." }]
}
```
Check `errors` array—empty means all found.

**compose** returns error on first missing:
```json
{ "error": { "code": "NOT_FOUND", "message": "capsule not found: ..." } }
```
No partial results. Fix the missing item and retry.

**bulk_update** uses filters + update fields:
```
bulk_update(workspace: "project", set_phase: "archived")
bulk_update(run_id: "task-1", set_role: "completed", set_tags: ["done"])
```
- Filters: `workspace`, `tag`, `name_prefix`, `run_id`, `phase`, `role` (AND semantics)
- Updates: `set_phase`, `set_role`, `set_tags` (prefixed with `set_` to distinguish from filters)
- Empty string (`""`) clears the field; empty array `[]` clears tags
- Requires at least one filter AND one update field

## Full-Text Search

**search** uses FTS5 for keyword search with relevance ranking:
```
search(query: "authentication")
search(query: "JWT OR OAuth", workspace: "project")
search(query: "auth*", run_id: "pr-123", phase: "review")
search(query: "JWT", include_deleted: true)
```

**Query syntax:**
- Simple: `authentication` (matches anywhere)
- Phrase: `"user authentication"` (exact match)
- Prefix: `auth*` (matches auth, authentication, authorize...)
- Boolean: `JWT OR OAuth`, `Redis AND cache`, `NOT deprecated`

**Filters:** `workspace`, `tag`, `run_id`, `phase`, `role` (AND semantics with query)

**Options:** `include_deleted: true` to include soft-deleted capsules (default: false)

**Results:**
- Ranked by relevance (BM25), title matches weighted 5x higher
- Returns `snippet` field with match context (~300 chars, `<b>` highlights)
- Pagination: `limit` (default 20, max 100), `offset`

## Backup & Restore

**Export:**
```
export()                                    # All → ~/.moss/exports/<timestamp>.jsonl
export(workspace: "frontend")               # One workspace
export(path: "/tmp/backup.jsonl")           # Custom path
export(include_deleted: true)               # Include soft-deleted
```

**Import:**
```
import(path: "/tmp/backup.jsonl")                    # mode: "error" (atomic, fail on collision)
import(path: "/tmp/backup.jsonl", mode: "replace")   # Overwrite existing
import(path: "/tmp/backup.jsonl", mode: "rename")    # Auto-suffix on collision
```

## Error Handling

| Error Code | Cause | Fix |
|------------|-------|-----|
| `NOT_FOUND` | Capsule doesn't exist | Check name/workspace spelling |
| `NAME_ALREADY_EXISTS` | Name collision on store | Use `mode: "replace"` to overwrite |
| `AMBIGUOUS_ADDRESSING` | Both id AND name provided | Use only one addressing mode |
| `CAPSULE_TOO_LARGE` | Exceeds 12,000 chars | Distill further, trim content |
| `CAPSULE_TOO_THIN` | Missing required sections | Add sections, or `allow_thin: true` |
| `FILE_TOO_LARGE` | Import file > 25MB | Split the export file |
| `COMPOSE_TOO_LARGE` | Composed bundle > 12,000 chars | Fewer items, or trim source capsules |
| `CANCELLED` | Context cancelled during long-running op | Retry the operation |

Error response format:
```json
{
  "error": {
    "code": "CAPSULE_TOO_THIN",
    "message": "Capsule missing required sections",
    "status": 422,
    "details": { "missing": ["Decisions", "Key locations"] }
  }
}
```

## Limits

| Limit | Default |
|-------|---------|
| Capsule size | 12,000 chars (~3k tokens) |
| Import file | 25 MB |
| `list` page | 100 max |
| `inventory` page | 500 max |
| `search` page | 100 max |
| `fetch_many` items | 50 max |
| `compose` items | 50 max |

## mode: "replace" Behavior

**Important:** `mode: "replace"` only targets **active** capsules.

- If an active capsule exists with that name → overwrites it
- If the capsule was soft-deleted → creates a **new** capsule (doesn't revive)
- If no capsule exists → creates new

To revive a soft-deleted capsule, use `export` with `include_deleted: true`, then `import`.

## Tips

1. **Distill, don't dump** — Capsules are summaries, not transcripts
2. **Use `mode: "replace"`** — For iterative updates to same capsule
3. **Use `allow_thin` sparingly** — Only for quick scratch notes
4. **Name meaningfully** — `auth-login-flow` not `capsule-1`
5. **Use `latest`** — Quickest way to resume; add `include_text: true` for full content
6. **Search by content** — `search(query: "JWT")` to find capsules by what they contain
7. **Browse first** — `inventory` or `list` to find, then `fetch` to load
8. **Soft deletes recover** — Use `include_deleted: true` to find them
