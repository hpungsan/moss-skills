# Capsule Examples

Practical examples for common capsule operations.

## Store a New Capsule

```
capsule_store(
  workspace: "default",
  name: "auth-refactor",
  capsule_text: "## Objective
Refactor authentication to use JWT tokens.

## Status
Design complete. Implementation starting.

## Decisions
- Use RS256 for token signing (security requirement)
- 15-minute token expiry with refresh tokens
- Store refresh tokens in httpOnly cookies

## Next actions
- Implement token generation in auth/jwt.go
- Add middleware for token validation
- Update login endpoint to return tokens

## Key locations
- `internal/auth/jwt.go` - new token logic
- `internal/middleware/auth.go` - validation middleware
- `cmd/api/routes.go` - endpoint registration

## Open questions
- Should we support multiple signing keys for rotation?"
)
```

Returns:
```json
{
  "id": "01HX7YGBK3...",
  "fetch_key": { "workspace": "default", "name": "auth-refactor" }
}
```

## Update Existing Capsule

```
capsule_store(
  workspace: "default",
  name: "auth-refactor",
  mode: "replace",
  capsule_text: "## Objective
Refactor authentication to use JWT tokens.

## Status
Token generation complete. Middleware in progress.

## Decisions
- Use RS256 for token signing (security requirement)
- 15-minute token expiry with refresh tokens
- Store refresh tokens in httpOnly cookies
- Key rotation: support 2 active keys during rotation period

## Next actions
- Complete middleware for token validation
- Add refresh token endpoint
- Write integration tests

## Key locations
- `internal/auth/jwt.go` - token logic (done)
- `internal/middleware/auth.go` - validation (in progress)
- `internal/auth/refresh.go` - refresh logic (new)

## Open questions
- None currently"
)
```

## Append to Section

Add content to a specific section without rewriting the full capsule:

```
capsule_append(
  workspace: "default",
  name: "auth-refactor",
  section: "Decisions",
  content: "- Key rotation: support 2 active keys during rotation period"
)
```

Returns:
```json
{
  "id": "01HX7YGBK3...",
  "fetch_key": { "moss_capsule": "auth-refactor", "moss_workspace": "default" },
  "section_hit": "## Decisions",
  "replaced": false
}
```

**Accumulate design reviews:**
```
capsule_append(
  workspace: "default",
  name: "pr-123-design",
  section: "Design Reviews",
  content: "Round 2: APPROVE\n- Security concerns addressed\n- Added rate limiting"
)
```

**Replace placeholder:**
```
# If section contains only "(pending)" or "TBD", replaces it entirely
capsule_append(
  workspace: "default",
  name: "auth-refactor",
  section: "Open questions",
  content: "- How to handle token refresh during deploy?"
)
# Returns: { "replaced": true, ... }
```

**Use synonyms:**
```
# "Status" matches "## Current status"
capsule_append(name: "auth", section: "Status", content: "Middleware complete")

# "goal" matches "## Objective"
capsule_append(name: "auth", section: "goal", content: "Added: support SSO")
```

## Resume Work

**Pattern 1: Quick check then load**
```
# See what's most recent (no text, fast)
capsule_latest(workspace: "default")
# Returns: { name: "auth-refactor", updated_at: ..., phase: ... }

# Load full content
capsule_fetch(workspace: "default", name: "auth-refactor")
```

**Pattern 2: Direct load with text**
```
capsule_latest(workspace: "default", include_text: true)
```

## Browse Capsules

**List in workspace:**
```
capsule_list(workspace: "default")
capsule_list(workspace: "default", limit: 50, offset: 0)
```

**Global inventory:**
```
capsule_inventory()
capsule_inventory(workspace: "frontend", tag: "urgent")
capsule_inventory(run_id: "pr-123", phase: "review")
```

**Find deleted capsules:**
```
capsule_list(workspace: "default", include_deleted: true)
```

## Search Capsules

**Simple keyword search:**
```
capsule_search(query: "authentication")
```

Returns results ranked by relevance with match snippets:
```json
{
  "items": [
    {
      "id": "01HX...",
      "workspace": "default",
      "name": "auth-refactor",
      "snippet": "...using <b>authentication</b> tokens...",
      "fetch_key": { "moss_capsule": "auth-refactor", "moss_workspace": "default" }
    }
  ],
  "pagination": { "limit": 20, "offset": 0, "has_more": false, "total": 1 },
  "sort": "relevance"
}
```

**Phrase search (exact match):**
```
capsule_search(query: "\"user authentication\"")
```

**Prefix search:**
```
capsule_search(query: "auth*")  # Matches auth, authentication, authorize, etc.
```

**Boolean search:**
```
capsule_search(query: "JWT OR OAuth")
capsule_search(query: "Redis AND cache")
capsule_search(query: "authentication NOT deprecated")
```

**Search with filters:**
```
capsule_search(query: "security", workspace: "project")
capsule_search(query: "JWT", run_id: "pr-review-123", phase: "review")
capsule_search(query: "bug", tag: "urgent")
capsule_search(query: "JWT", include_deleted: true)  # Include soft-deleted capsules
```

**Paginated search:**
```
capsule_search(query: "authentication", limit: 10, offset: 0)  # First page
capsule_search(query: "authentication", limit: 10, offset: 10) # Second page
```

## Multi-Agent Orchestration

**Orchestrator sets up base context:**
```
capsule_store(
  workspace: "default",
  name: "pr-123-base",
  run_id: "pr-review-123",
  phase: "setup",
  role: "orchestrator",
  capsule_text: "## Objective
Review PR #123 for security, quality, and documentation.

## Status
Base context created. Spawning review agents.

## Decisions
- Three parallel reviewers: security, qa, docs
- Each stores findings with same run_id

## Next actions
- Security reviewer: check for vulnerabilities
- QA reviewer: check for bugs and test coverage
- Docs reviewer: check for documentation gaps

## Key locations
- PR diff: `gh pr diff 123`
- Changed files: src/auth/*.go, docs/API.md

## Open questions
- None"
)
```

**Agent A (security reviewer) stores findings:**
```
capsule_store(
  workspace: "default",
  name: "pr-123-security",
  run_id: "pr-review-123",
  phase: "review",
  role: "security-reviewer",
  capsule_text: "## Objective
Security review of PR #123.

## Status
Review complete. 1 blocker found.

## Decisions
- Marked SQL injection as blocker
- Input validation warning is non-blocking

## Next actions
- Orchestrator to aggregate findings

## Key locations
- `src/auth/login.go:45` - SQL injection
- `src/auth/validate.go:23` - weak validation

## Open questions
- None"
)
```

**Orchestrator collects all findings:**
```
capsule_fetch_many(items: [
  { workspace: "default", name: "pr-123-security" },
  { workspace: "default", name: "pr-123-qa" },
  { workspace: "default", name: "pr-123-docs" }
])
```

**Or query by run_id:**
```
capsule_list(workspace: "default", run_id: "pr-review-123")
```

**Compose all findings into one bundle:**
```
capsule_compose(
  items: [
    { workspace: "default", name: "pr-123-security" },
    { workspace: "default", name: "pr-123-qa" },
    { workspace: "default", name: "pr-123-docs" }
  ]
)
```

Returns markdown bundle:
```markdown
## pr-123-security

## Objective
Security review of PR #123.
...

---

## pr-123-qa

## Objective
QA review of PR #123.
...
```

**Compose and store as new capsule:**
```
capsule_compose(
  items: [
    { workspace: "default", name: "pr-123-security" },
    { workspace: "default", name: "pr-123-qa" },
    { workspace: "default", name: "pr-123-docs" }
  ],
  store_as: {
    workspace: "default",
    name: "pr-123-combined",
    mode: "replace"
  }
)
```

**Note:** `store_as` requires markdown format (default). JSON format + `store_as` returns an error.

**JSON format for programmatic use:**
```
capsule_compose(
  items: [...],
  format: "json"
)
```

Returns structured data (cannot be stored as capsule):
```json
{
  "parts": [
    { "id": "01HX...", "workspace": "default", "name": "pr-123-security", "title": "...", "text": "...", "chars": 1234 },
    ...
  ]
}
```

## Bulk Update by Filter

Update metadata (phase, role, tags) on all capsules matching filters. Requires at least one filter AND one update field.

**Archive all capsules in a workspace:**
```
capsule_bulk_update(workspace: "project", set_phase: "archived")
```

**Update role for completed run:**
```
capsule_bulk_update(run_id: "pr-review-123", set_role: "completed")
```

**Replace tags for capsules matching filters:**
```
capsule_bulk_update(workspace: "project", phase: "review", set_tags: ["reviewed", "approved"])
```

**Clear a field (empty string = null):**
```
capsule_bulk_update(workspace: "scratch", set_phase: "")  # Clears phase field
capsule_bulk_update(run_id: "old-run", set_tags: [])      # Clears all tags
```

**Multiple update fields:**
```
capsule_bulk_update(
  workspace: "project",
  tag: "completed",
  set_phase: "archived",
  set_role: "done"
)
```

Returns:
```json
{
  "updated": 5,
  "message": "Updated 5 capsules matching workspace=\"project\", tag=\"completed\"; set phase=\"archived\", role=\"done\""
}
```

**Note:** Only targets active capsules. Filters use AND semantics (all must match).

## Bulk Delete by Filter

Delete all capsules matching filters (AND semantics). At least one filter required.

**Delete all capsules in a workspace:**
```
capsule_bulk_delete(workspace: "scratch")
```

**Delete stale research capsules in a project:**
```
capsule_bulk_delete(workspace: "myproject", phase: "research", tag: "stale")
```

**Delete all capsules from a completed run:**
```
capsule_bulk_delete(run_id: "pr-review-123")
```

Returns:
```json
{
  "deleted": 3,
  "message": "Soft-deleted 3 capsules matching run_id=\"pr-review-123\""
}
```

**Note:** Only targets active capsules. Already soft-deleted capsules are unaffected. Use `capsule_purge` to permanently remove soft-deleted capsules.

## Quick Scratch Note

For temporary notes that don't need full structure:

```
capsule_store(
  workspace: "scratch",
  name: "debug-notes",
  allow_thin: true,
  capsule_text: "Issue: Login fails with empty password. Check validate.go line 45."
)
```

## Error Recovery

**Name collision:**
```
# First attempt
capsule_store(name: "auth", capsule_text: "...")
# Error: NAME_ALREADY_EXISTS

# Fix: use replace mode
capsule_store(name: "auth", mode: "replace", capsule_text: "...")
```

**Capsule too large:**
```
# Error: CAPSULE_TOO_LARGE (limit 12,000 chars)

# Fix: distill further
# - Remove verbose explanations
# - Keep only key file:line references
# - Summarize decisions, don't explain
# - Max 1-3 small code snippets
```

**Missing sections:**
```
# Error: CAPSULE_TOO_THIN, missing: ["Decisions", "Key locations"]

# Fix: add missing sections
# Or for scratch notes:
capsule_store(name: "note", allow_thin: true, capsule_text: "...")
```
