# Capsule Reference

Detailed reference for capsules.

## Section Names (Lint Validation)

When storing a capsule, these section names satisfy the 6 required sections (case-insensitive):

| Canonical | Also accepted |
|-----------|---------------|
| Objective | Goal, Purpose |
| Status | Current status, State, Where we are |
| Decisions | Decisions/constraints, Constraints, Choices |
| Next actions | Next steps, Action items, Todo, Tasks |
| Key locations | Locations, Files, Paths, References |
| Open questions | Questions, Risks, Unknowns |

**Formats accepted:** Markdown headers (`## Section`), colon-style (`Section:`), or JSON keys.

**Note:** These synonyms apply to lint validation only. For `capsule_append`, use the exact header name as it appears in the capsule (see [Section Append](#section-append)).

## Name Normalization

Names are normalized: lowercased, whitespace collapsed.

```
"Auth System"     → "auth system"
"  My Project  "  → "my project"
"LOUD_NAME"       → "loud_name"
```

Fetch using either raw or normalized form—both work.

## Capsule Schema

### Input Fields (store/update)

Fields you provide when creating or updating capsules.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workspace` | string | No | Namespace for organizing capsules. Default: `"default"` |
| `name` | string | No | Unique handle within workspace. Omit for unnamed capsules. |
| `title` | string | No | Human-readable title. Defaults to `name` if not provided. |
| `capsule_text` | string | **Yes** | Main content. Must have 6 sections unless `allow_thin: true`. |
| `tags` | string[] | No | Categorization tags (e.g., `["urgent", "frontend"]`) |
| `source` | string | No | Origin identifier (e.g., session ID, file path) |
| `run_id` | string | No | Orchestration run identifier for multi-agent workflows |
| `phase` | string | No | Workflow phase (e.g., `"design"`, `"implement"`, `"review"`) |
| `role` | string | No | Agent role (e.g., `"architect"`, `"qa-reviewer"`) |

**Store-only options:**
| Option | Type | Description |
|--------|------|-------------|
| `mode` | string | `"error"` (default, fail on collision) or `"replace"` (overwrite) |
| `allow_thin` | bool | Skip section validation for quick notes. Default: `false` |

### Output Fields (responses)

Fields returned in API responses. Includes all input fields plus:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | ULID, auto-generated. Use for precise lookups. |
| `workspace_norm` | string | Normalized workspace (lowercased, trimmed) |
| `name_norm` | string? | Normalized name (lowercased, trimmed) |
| `capsule_chars` | int | Character count (runes, not bytes). Computed on store. |
| `tokens_estimate` | int | Estimated LLM tokens (~chars/4). Computed on store. |
| `created_at` | int | Unix timestamp when capsule was created |
| `updated_at` | int | Unix timestamp of last modification |
| `deleted_at` | int? | Unix timestamp when soft-deleted (null if active) |
| `fetch_key` | object | Convenience object for re-fetching (see below) |

**fetch_key structure:**
```json
{
  "moss_capsule": "auth-refactor",
  "moss_workspace": "default"
}
```
Returned by `capsule_store`, `capsule_fetch`, `capsule_fetch_many`, and `capsule_search`. Pass to `capsule_fetch` for easy retrieval.

### Response Variations by Tool

| Tool | Returns | Notes |
|------|---------|-------|
| `capsule_fetch` | Full capsule | All fields including `capsule_text` |
| `capsule_fetch_many` | Full capsules + errors | `items` array + `errors` array for missing |
| `capsule_latest` | Summary only | Add `include_text: true` for full content |
| `capsule_list` | Summaries | No `capsule_text`, includes `fetch_key` |
| `capsule_inventory` | Summaries | No `capsule_text`, includes `fetch_key` |
| `capsule_search` | Summaries + snippets | `snippet` field with `<b>` match highlights |
| `capsule_compose` | Bundle | `bundle_text` (markdown) or `parts` array (JSON) |
| `capsule_append` | Confirmation | `id`, `section_hit`, `replaced`, `fetch_key` |

**Summary fields:** `id`, `workspace`, `workspace_norm`, `name`, `name_norm`, `title`, `capsule_chars`, `tokens_estimate`, `tags`, `source`, `run_id`, `phase`, `role`, `created_at`, `updated_at`, `deleted_at`, `fetch_key`

## Multi-Agent Orchestration Fields

| Field | Purpose | Example values |
|-------|---------|----------------|
| `run_id` | Groups capsules for one task | `"pr-review-123"`, `"feature-xyz"` |
| `phase` | Workflow stage | `"design"`, `"implement"`, `"review"`, `"verified"` |
| `role` | Agent type | `"architect"`, `"coder"`, `"qa-reviewer"` |

Query by orchestration:
```
capsule_latest(run_id: "feature-xyz", phase: "review", role: "qa")
capsule_list(workspace: "default", run_id: "feature-xyz")
capsule_inventory(phase: "review")  # All review-phase capsules globally
capsule_search(query: "JWT", run_id: "feature-xyz")  # FTS5 search within run
```

## Batch Operations

Several tools handle multiple capsules—they have different behaviors:

| Tool | Behavior | Use when |
|------|----------|----------|
| `capsule_search` | **Ranked results** — FTS5 full-text search with snippets | Find capsules by content |
| `capsule_fetch_many` | **Partial success** — returns found items + errors array | You need whatever's available |
| `capsule_compose` | **All-or-nothing** — fails on first missing | You need ALL items or none |
| `capsule_bulk_update` | **Filter-based** — updates all matching capsules | Batch metadata changes |
| `capsule_bulk_delete` | **Filter-based** — soft-deletes all matching | Batch cleanup |

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
capsule_bulk_update(workspace: "project", set_phase: "archived")
capsule_bulk_update(run_id: "task-1", set_role: "completed", set_tags: ["done"])
```
- Filters: `workspace`, `tag`, `name_prefix`, `run_id`, `phase`, `role` (AND semantics)
- Updates: `set_phase`, `set_role`, `set_tags` (prefixed with `set_` to distinguish from filters)
- Empty string (`""`) clears the field; empty array `[]` clears tags
- Requires at least one filter AND one update field

## Full-Text Search

**search** uses FTS5 for keyword search with relevance ranking:
```
capsule_search(query: "authentication")
capsule_search(query: "JWT OR OAuth", workspace: "project")
capsule_search(query: "auth*", run_id: "pr-123", phase: "review")
capsule_search(query: "JWT", include_deleted: true)
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

## Section Append

Append content to a specific section without rewriting the full capsule:

```
capsule_append(workspace: "default", name: "auth", section: "Decisions", content: "Round 2: Approved")
capsule_append(id: "01HX...", section: "Current status", content: "Implementation complete")
```

**Section matching:**
- Exact header name match (case-insensitive)
- No synonym resolution — use the header as written (e.g., `## Current status` → `"Current status"`)
- Custom sections work: `## Design Reviews` → `"Design Reviews"`
- Error message lists available sections if not found

**Placeholder handling:** If section contains only placeholder text (`(pending)`, `TBD`, `N/A`, `-`, `none`), replaces it entirely. Otherwise appends after existing content with blank line separator.

**Response:**
```json
{
  "id": "01HX...",
  "fetch_key": { "moss_capsule": "auth", "moss_workspace": "default" },
  "section_hit": "## Decisions",
  "replaced": false
}
```
- `section_hit`: Actual header that matched
- `replaced`: `true` if placeholder was replaced, `false` if content was appended
- `fetch_key`: Omitted for unnamed capsules

**Errors:**
- Section not found → `INVALID_REQUEST` (422)
- Result exceeds size limit → `CAPSULE_TOO_LARGE` (413)
- JSON format capsule (no markdown sections) → `INVALID_REQUEST` (422)

## Backup & Restore

**Export:**
```
capsule_export()                                    # All → ~/.moss/exports/<timestamp>.jsonl
capsule_export(workspace: "frontend")               # One workspace
capsule_export(path: "/tmp/backup.jsonl")           # Custom path
capsule_export(include_deleted: true)               # Include soft-deleted
```

**Import:**
```
capsule_import(path: "/tmp/backup.jsonl")                    # mode: "error" (atomic, fail on collision)
capsule_import(path: "/tmp/backup.jsonl", mode: "replace")   # Overwrite existing
capsule_import(path: "/tmp/backup.jsonl", mode: "rename")    # Auto-suffix on collision
```

## Error Handling

| Error Code | Cause | Fix |
|------------|-------|-----|
| `NOT_FOUND` | Capsule doesn't exist | Check name/workspace spelling |
| `NAME_ALREADY_EXISTS` | Name collision on store | Use `mode: "replace"` to overwrite |
| `AMBIGUOUS_ADDRESSING` | Both id AND name provided | Use only one addressing mode |
| `INVALID_REQUEST` | Invalid request parameters | Check parameter types and values |
| `CONFLICT` | Concurrent modification conflict (reserved) | Retry with fresh data |
| `CAPSULE_TOO_LARGE` | Exceeds 12,000 chars | Distill further, trim content |
| `CAPSULE_TOO_THIN` | Missing required sections | Add sections, or `allow_thin: true` |
| `FILE_TOO_LARGE` | Import file > 25MB | Split the export file |
| `COMPOSE_TOO_LARGE` | Composed bundle > 12,000 chars | Fewer items, or trim source capsules |
| `CANCELLED` | Context cancelled during long-running op | Retry the operation |
| `INTERNAL` | Unexpected server error | Report bug if reproducible |

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
| `capsule_list` page | 100 max |
| `capsule_inventory` page | 500 max |
| `capsule_search` page | 100 max |
| `capsule_fetch_many` items | 50 max |
| `capsule_compose` items | 50 max |

## mode: "replace" Behavior

**Important:** `mode: "replace"` only targets **active** capsules.

- If an active capsule exists with that name → overwrites it
- If the capsule was soft-deleted → creates a **new** capsule (doesn't revive)
- If no capsule exists → creates new

To revive a soft-deleted capsule, use `capsule_export` with `include_deleted: true`, then `capsule_import`.

## Tips

1. **Distill, don't dump** — Capsules are summaries, not transcripts
2. **Use `mode: "replace"`** — For iterative updates to same capsule
3. **Use `capsule_append` for history** — Add to sections without rewriting (reviews, decisions, status updates)
4. **Use `allow_thin` sparingly** — Only for quick scratch notes
5. **Name meaningfully** — `auth-login-flow` not `capsule-1`
6. **Use `capsule_latest`** — Quickest way to resume; add `include_text: true` for full content
7. **Search by content** — `capsule_search(query: "JWT")` to find capsules by what they contain
8. **Browse first** — `capsule_inventory` or `capsule_list` to find, then `capsule_fetch` to load
9. **Soft deletes recover** — Use `include_deleted: true` to find them
