---
name: GitHub API Efficiency
description: Bundle, cache, batch — survive the 5000 pts/hour GraphQL limit and 5000/hour REST limit during board manipulation, issue grooming, and orchestrator work.
category: process
applicable_phases: [orchestration, planning]
applicable_stacks: [universal]
version: 1
---

GitHub's GraphQL API has a 5000 points/hour budget. REST has a separate 5000 calls/hour. Sloppy per-item lookup-then-edit patterns burn through GraphQL in ~10 minutes of board grooming. This skill is the rule to follow whenever an agent touches the GitHub API at scale.

## The single rule

**Bundle. Cache. Batch.** Pull state once, keep IDs in a local cache, batch mutations through GraphQL aliases.

## Anti-pattern (don't do this)

```bash
# For each of 50 issues:
gh issue view 384 --json projectItems  # call 1: returns ALL 200 board items
gh project item-edit ...               # call 2: mutate one
# Repeat 50 times → 100 API calls, ~3000 GraphQL points
```

The first call returns the entire board for every issue. Quadratic cost.

## Pattern (do this)

### 1. Pre-flight rate check

Start every board session with:

```bash
gh api graphql -f query='query { rateLimit { remaining limit cost resetAt } }'
```

If `remaining < 500`, switch to REST-only operations or wait for reset.

### 2. Pull board state ONCE per session

Single paginated query that fetches all items + relevant fields:

```graphql
query GetBoard($cursor: String) {
  organization(login: "your-org") {
    projectV2(number: 3) {
      items(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          content { ... on Issue { number title state } }
          fieldValueByName(name: "Release") {
            ... on ProjectV2ItemFieldSingleSelectValue { name optionId }
          }
          # add Priority, Status, etc.
        }
      }
    }
  }
}
```

Cache the result locally as `.tmp/board-cache.json`. Re-query only after structural changes.

### 3. Cache stable IDs

Project ID, field IDs, option IDs change rarely. Hardcode them in the session or `.tmp` cache. Never re-query "what's the Release field ID" 20 times.

```javascript
// Stable IDs for one project, kept in code or .tmp:
const PROJECT_ID = "PVT_kwDOXXXX";
const RELEASE_FIELD = "PVTSSF_lADOXXXX";
const OPTIONS = { "R1.5": "babe3c64", "R1.6": "ab198e47", ... };
```

### 4. Batch mutations via aliases

N item updates = 1 GraphQL call with N aliases, NOT N separate calls:

```graphql
mutation BulkMove {
  m1: updateProjectV2ItemFieldValue(input: {projectId: "...", itemId: "PVTI_a", fieldId: "...", value: {singleSelectOptionId: "..."}}) { projectV2Item { id } }
  m2: updateProjectV2ItemFieldValue(input: {projectId: "...", itemId: "PVTI_b", fieldId: "...", value: {singleSelectOptionId: "..."}}) { projectV2Item { id } }
  m3: addSubIssue(input: {issueId: "I_a", subIssueId: "I_b"}) { issue { number } }
  m4: addBlockedBy(input: {issueId: "I_a", blockingIssueId: "I_c"}) { issue { number } }
}
```

GraphQL charges by complexity, not call count. 16 mutations in one call ≈ 16 points. 16 separate calls ≈ 16+ points each (because each carries fixed overhead).

### 5. Use REST when it's cheaper

Plain issue lists, label edits, and assignee changes are often cheaper via REST:

```bash
gh api 'repos/owner/repo/issues?state=open&per_page=100&labels=lane:claude'
```

Save GraphQL for project-board specifics: field values, sub-issues, project items, blocked-by edges.

### 6. Adding a single-select option recreates ALL option IDs

`updateProjectV2Field` on a single-select field with a new option **regenerates IDs for every option**. This blanks the field value on every item that had it set. See the related feedback memory `feedback_project_field_options_recreate_ids` for the snapshot+restore routine.

If GitHub adds an option-only mutation in the future, prefer it.

## Quick budget reference

| Operation | Approx GraphQL cost |
|---|---|
| `rateLimit` check | 1 |
| Single issue read | 1 |
| Issue with projectItems(first:5) | 1-2 |
| Board pull, 100 items + 4 fields each | ~10-15 |
| Single mutation | 1 |
| Batched mutation, 16 aliases | ~16 |
| Sub-issue link / blocked-by edge | 1-2 |
| Issue creation | 1-2 |

A typical "groom 30 issues" session: ~50 points if batched, ~3000 if not.

## When this skill applies

- Migrating items between project fields (release, priority, status)
- Bulk creating issues + linking them as sub-issues / dependencies
- Renaming issues across the board
- Wave-review post-merge audits
- Orchestrator dispatch + status sync

## When it doesn't

- Single ad-hoc lookups (just `gh issue view` is fine)
- Workflows that only touch one or two issues
- Short tasks where rate-limit isn't a concern

## See also

- [GitHub GraphQL rate limit docs](https://docs.github.com/en/graphql/overview/resource-limitations) — official accounting model
- `feedback_project_field_options_recreate_ids` — option-ID regeneration gotcha
