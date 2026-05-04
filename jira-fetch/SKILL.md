---
name: jira-fetch
description: Fetch a Jira ticket via ACLI return JSON bundle (ticket body, comments, labels, linked issues, keyword-matched similar tickets) for downstream skills (triage, draft-prd, to-issues) to consume in fresh sessions. Use whenever the user wants to pull a Jira ticket into the workspace, prepare a ticket for triage, or names a ticket key (e.g. "ABC-123") and asks to look at, fetch, ingest, or work on it. Also use when the user says things like "let's triage ABC-123", "pull in ticket X", or "what's in ABC-123".
---

# jira-fetch

Pull a Jira ticket via `acli` and write a normalized JSON bundle to disk for downstream skills (`/triage`, `/draft-prd`, `/to-issues`) that run in fresh sessions. The on-disk JSON is the contract — anything not in it is invisible to them.

Read-only. Never writes to Jira.

## Output

Write to `${WHETSTONE_RUNS_DIR:-./runs}/<KEY>/jira.json` relative to the user's cwd. Pretty-print with 2-space indent.

If the file already exists and `--force` was not passed, print "already fetched" with the relative age (from `fetched_at`) plus the `Next: /triage <KEY>` hint, and stop.

## Critical rule: always slim `acli` output with `jq`

Never read raw `acli` JSON. Every response is padded with avatar URLs, `self` links, `statusCategory` subtrees, and `expand` strings — 30+ noise lines per ticket. Always pipe through `jq` and project only the fields you need. A 17-ticket pool went from ~60k tokens raw to a few hundred slimmed.

This applies to *every* `acli` call in this skill, including exploratory ones. Don't `head` the raw output to inspect shape — go straight to the slim filter below.

## Pre-flight

1. `which acli`. Missing → exit: *"`acli` not installed. See https://developer.atlassian.com/cloud/acli/."*
2. `acli jira auth status`. Not authenticated → exit: *"Run `acli jira auth login`."*
3. If the first real API call returns `unauthorized: use 'acli [product] auth login' to authenticate`, that's a token-scope mismatch (the stored profile exists but lacks Jira scope) — surface it and tell the user to re-run `acli jira auth login`. Single most common first-run failure.

## `acli` command surface (v1.3.18, Jira Cloud)

- View ticket: `acli jira workitem view <KEY> --fields '*all' --json` (note `workitem`, not `issue`; the default field set omits labels/priority/comments/issuelinks, so `*all` is required).
- JQL search: `acli jira workitem search --jql '<jql>' --fields 'key,summary,status,issuetype,priority' --limit 20 --json`. `*all` is rejected here; the search subcommand has a constrained allowlist (`resolution`, `updated`, `labels` will error).
- Flag is `--json`, not `--output json`.

## Step 1 — Fetch the main ticket

```
acli jira workitem view <KEY> --fields '*all' --json | jq '{
  key,
  summary: .fields.summary,
  status: .fields.status.name,
  priority: .fields.priority.name,
  issuetype: .fields.issuetype.name,
  labels: .fields.labels,
  project: .fields.project.key,
  created: .fields.created,
  updated: .fields.updated,
  reporter: {name: .fields.reporter.displayName, email: .fields.reporter.emailAddress},
  assignee: (if .fields.assignee then {name: .fields.assignee.displayName, email: .fields.assignee.emailAddress} else null end),
  description: .fields.description,
  comments: .fields.comment.comments,
  issuelinks: .fields.issuelinks
}'
```

If this call fails — abort. Don't write a partial `jira.json`; downstream skills assume the file is well-formed and partial is worse than absent. Distinguish "not found / no permission" from generic failures (look at exit code + stderr) and include `acli`'s stderr for the latter.

### ADF flattening

`description` and comment bodies come back as Atlassian Document Format — a nested tree of `type` nodes. Render to plain text (drop marks like bold/italic/links — triage cares about words, not formatting):

- `text` → `node.text`
- `paragraph` → recurse, append `\n\n`
- `hardBreak` → `\n`
- `bulletList` / `orderedList` → recurse over children
- `listItem` → `"- " + recurse(content).strip() + "\n"`
- `heading` → `"#"*level + " " + recurse(content) + "\n\n"`
- `codeBlock` → fenced
- anything else with a `content` array → recurse

## Step 2 — Linked issues

`fields.issuelinks` from Step 1 has each link's `type.inward`/`outward` strings and an `inwardIssue`/`outwardIssue` with `key`, summary, and status usually inlined — use those without extra calls.

If summary or status is missing, fetch with another slimmed `acli jira workitem view <LINKED-KEY>` per Step 1. Cap at 10 linked issues. Best-effort: on failure, set that entry's summary/status to `null` and continue.

## Step 3 — Similarity hint

A *hint* via cheap keyword JQL, not a ranked judgment. Downstream skills (and the human) decide what's actually relevant.

**Pick one keyword** from the source `summary`: the most distinctive content word (typically the most specific noun). Lowercase. Skip stopwords and Jira-generic verbs (`report`, `request`, `issue`, `update`, `fix`). If the summary is only stopwords, skip the search (`similar_tickets: []`, `keyword_used: null`).

One word only — JQL `~` stems and ANDs multi-word queries, silently filtering genuine matches.

Examples: `"Report of malfunctioning printer"` → `printer`. `"Login fails after password reset"` → `login`.

**Run:**
```
acli jira workitem search --jql 'project = <PROJECT> AND summary ~ "<keyword>" ORDER BY updated DESC' --fields 'key,summary,status,issuetype,priority' --limit 20 --json | jq '[.[] | {key, summary: .fields.summary, status: .fields.status.name, issuetype: .fields.issuetype.name, priority: .fields.priority.name}]'
```

Then: filter out the source ticket, cap at 10, preserve Jira's `updated DESC` order, no rerank.

On failure: `similar_tickets: []`, capture the message in `similar_search.error`, keep going.

## Step 4 — Write `jira.json`

```json
{
  "fetched_at": "2026-05-04T14:23:00Z",
  "key": "ABC-123",
  "project": "ABC",
  "summary": "...",
  "description": "...",
  "issuetype": "Bug",
  "status": "Open",
  "priority": "Medium",
  "labels": ["foo", "bar"],
  "reporter": {"name": "Jane Doe", "email": "jane@example.com"},
  "assignee": {"name": "Bob Smith", "email": "bob@example.com"},
  "created": "2026-04-15T09:12:00Z",
  "updated": "2026-05-01T16:40:00Z",
  "comments": [{"author": "Jane Doe", "created": "2026-04-15T09:30:00Z", "body": "..."}],
  "linked_issues": [{"key": "ABC-100", "relationship": "is blocked by", "summary": "...", "status": "In Progress"}],
  "similar_tickets": [{"key": "ABC-89", "summary": "...", "status": "Closed", "issuetype": "Bug", "priority": "Medium"}],
  "similar_search": {"keyword_used": "login", "query_used": "project = ABC AND summary ~ \"login\" ORDER BY updated DESC", "error": null}
}
```

Field rules:

- `fetched_at` — ISO 8601 UTC timestamp of this run.
- `description` — plain text from ADF flattening.
- `reporter`/`assignee` — `{name, email}` or `null` (not `{}`) when unassigned.
- `comments` — full thread, oldest-first, no truncation. Let the downstream session manage its own context budget.
- `linked_issues` — Jira native link types only; `null` summary/status if a fetch failed.
- `similar_tickets` — capped at 10, `updated DESC`, source filtered out. No `reason` field — this is a keyword hint, not a judgment.
- `similar_search.keyword_used` / `query_used` — `null` if search was skipped. `error` — non-null only on failure.

## Step 5 — Compact summary to chat

For the human in interactive mode (downstream skills run in fresh sessions and won't see this):

```
Fetched ABC-123 (project: ABC) → runs/ABC-123/jira.json

Status: Open · Priority: Medium · Type: Bug
Reporter: Jane Doe (jane@example.com)  Assignee: unassigned
Labels: login, regression
Created: 2026-04-15  Updated: 2026-05-01

Summary: Login fails after password reset

Comments: 4
Linked: ABC-100 (is blocked by), ABC-77 (relates to)
Similar (keyword "login"): ABC-89, ABC-54, ABC-12

Next: /triage ABC-123
```

Adaptations:
- Unassigned → `Assignee: unassigned`
- No priority → `Priority: (none)`
- No labels → `Labels: (none)`
- No linked issues → omit the `Linked:` line
- Similar — list up to 5 keys, append ` (+N more)` if more. Always show the keyword so the reader can judge it.
- No similar matches → `Similar: (none found for keyword "<kw>")`
- Search skipped (no usable keyword) → omit the `Similar:` line entirely
- Search failed → `Similar: (search failed: <error>)`

Always include the `Next: /jira-triage <KEY>` line — it's the next step in the workflow.
