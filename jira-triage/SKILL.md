---
name: jira-triage
description: Triage a Jira customer support ticket and produce a routing brief at runs/<KEY>/triage.md. Reads the jira.json bundle from `jira-fetch`, applies routing rules, surfaces related tickets and same-reporter history, and writes a structured markdown brief for the next person. Read-only — never writes to Jira, never drafts customer-facing text. Use this skill whenever the user wants to triage a JSM/Jira support ticket, asks for a routing recommendation, names a Jira key in a triage context, or says things like "triage this", "route this ticket", "what should we do with ABC-123", "work through this support ticket", or "prep this for the team". Also use as the natural next step after `/jira-fetch`. Companion to `jira-fetch` — calls it automatically if jira.json is missing or stale (>30 min old).
---

# jira-triage

Read a Jira ticket bundle from `runs/<KEY>/jira.json` and produce a triage brief at `runs/<KEY>/triage.md` that recommends a kanban route and frames the ticket for whoever picks it up next.

Read-only. Never writes to Jira. Never drafts customer-facing content.

Companion to `jira-fetch` — see [jira-fetch/SKILL.md](../jira-fetch/SKILL.md) for the JSON contract this skill consumes.

## Output

Write to `${WHETSTONE_RUNS_DIR:-./runs}/<KEY>/triage.md` relative to the user's cwd.

Always overwrites. No history files — re-runs just produce the latest brief.

## Hard rules (never violate)

These are the design contract for the skill. They hold even when later promoted to posting comments to Jira:

- **Never write to Jira.** No comments, no field changes, no column moves, no labels, no transitions.
- **Never draft customer-facing text.** The brief is internal — for the BA, Dev, or Infra engineer who picks the ticket up next. If the customer needs a reply, the human writes it from the brief's "Information missing" list. The brief just makes that cheap.
- **Never name individuals.** No suggested assignees. Recommend column + category only. Naming people that have left, moved teams, or are on leave is worse than naming no-one.

## Pre-flight: ensure fresh `jira.json`

Triage operates on `runs/<KEY>/jira.json`. Three cases:

1. **Missing** → call `/jira-fetch <KEY>`. Continue once it lands.
2. **Present but stale** (`fetched_at` >30 minutes old) → call `/jira-fetch <KEY> --force`. The `--force` flag is required because `jira-fetch` refuses to refetch otherwise.
3. **Present and fresh** → use as-is.

Never write or modify `jira.json` directly from this skill. The fetch is `jira-fetch`'s job.

## Optional context

If `./CONTEXT.md` exists in the user's cwd, read it at the start of every run. Use it to disambiguate routing — project glossaries, product/system names, "anything mentioning the deploy pipeline is infra", tenant conventions, recent incidents the team is aware of. Without it the skill runs on generic judgment; with it routing precision improves over time.

When `CONTEXT.md` decides a call, mention the specific term that mattered in `Source signals` (e.g. *"`platform` mentioned per CONTEXT.md → Infra"*). Don't quote `CONTEXT.md` back wholesale.

## Step 1 — Read inputs

- Read `runs/<KEY>/jira.json` in full.
- Read `./CONTEXT.md` if present.
- Note the source ticket's `status`. The skill is designed for triaging from `New`. If the status is anything else, still produce the brief but lead the recommendation with: *"Note: ticket is currently in `<status>`, not `New` — this skill is designed for triaging from `New`. Confirm before re-routing."* Don't refuse — the human invoked explicitly.

## Step 2 — Detect duplicate short-circuit

If `linked_issues` contains a relationship matching `/duplicate/i` (e.g. `is duplicate of`, `duplicates`, `is cloned by` if used as duplicate in your project):

- Skip the full routing analysis and the same-reporter scan.
- Write a stub brief (see "Variant: duplicate short-circuit" below).
- Stop.

This is authoritative. The reporter or a previous triager has already linked the duplicate; re-running keyword similarity wastes calls and may surface noise that contradicts the link.

## Step 3 — Same-reporter scan

For non-duplicate cases, run one extra JQL pass for the source ticket's reporter. Prefer `reporter.email` over `reporter.name` — display names match unreliably; emails (or accountIds, if available) match cleanly.

```
acli jira workitem search \
  --jql 'reporter = "<email>" AND project = "<PROJECT>" AND key != "<KEY>" ORDER BY updated DESC' \
  --fields 'key,summary,status,issuetype,priority' \
  --limit 10 --json \
  | jq '[.[] | {key, summary: .fields.summary, status: .fields.status.name, issuetype: .fields.issuetype.name, priority: .fields.priority.name}]'
```

Always slim with `jq` — see the "always slim with jq" rule in `jira-fetch/SKILL.md`. Never read raw `acli` output.

Always filter to the source ticket's project (the JQL above includes `project = "<PROJECT>"`). Cross-project tickets from the same reporter — especially in scratch/sandbox projects like `KAN`, `TEST`, `SCRATCH` — are noise, not signal. If the human needs cross-project context they'll ask for it explicitly.

Best-effort:

- If the reporter's email is missing from `jira.json` → skip the section (don't try display-name fallback, it's noisy).
- If the JQL call fails → omit the section, capture nothing extra. Don't bail the whole run.
- If the reporter has no other tickets → render `(no other tickets from this reporter)`.

## Step 4 — Apply routing rules

Decision order:

1. **Awaiting Customer Feedback** — *only* if the ticket is genuinely unworkable. The bar is high: the description is empty, just restates the title, or says only "things are broken" / "doesn't work" with zero specifics. Soft gates apply for everything else: missing fields are noted in the brief but don't block routing. The BA/Dev/Infra picking it up can chase down small gaps without a customer round-trip.

2. **Escalated to Experts** if there's evidence of a system fault. Sub-route hint in the brief (single column, no labels):
   - **Dev signals** — stack traces, exception types, "bug in the product", reproducible wrong behaviour, small feature requests with clear acceptance criteria.
   - **Infra signals** — upgrades, platform/cloud issues, environment-specific failures, generic "errors" without code-level detail, auth/SSO/network/deploy/DNS keywords.
   - When both fire (e.g. *"after the upgrade we get this NPE"*), flag the ambiguity in the brief and give a best guess + the signals you used. Don't force a confident call you'll be wrong about half the time.

3. **In Review (BA/SME)** — the default. Change requests to be assessed, tax logic questions, product clarifications. Also the destination for tickets that are clearly out-of-scope or duplicates *without* an explicit duplicate link (the BA closes them out with the customer, since there's no Won't Fix column).

**Tie-breaks:**

- **BA vs Dev** → lean BA. Assessment usually precedes build; sending a half-formed change to Dev wastes their time.
- **Dev vs Infra** → flag, don't force.

## Step 5 — Confidence rating

- **High** — clear, distinctive signals (explicit stack trace + version regression; explicit "how do I configure tax rate Y for region Z"). No unresolved gates.
- **Medium** — signals present but mixed; alternative routes are unlikely but not ruled out. No unresolved gates.
- **Low** — significant guesswork, OR routing depends on an unresolved gate (e.g. a possible duplicate, a fix that may already be in flight in another version, a thin description with only generic signals) that could invalidate the call entirely. The brief should make the uncertainty visible in the `Why` line so the human can spot-check.

Always include the source signals — what specifically in the ticket pointed at this route. The reader needs to verify the call without re-reading the whole ticket.

## Step 6 — "Since last triage" delta

If `runs/<KEY>/triage.md` already exists, compare its mtime to `jira.json`'s `fetched_at`:

- `fetched_at` newer than `triage.md` mtime → ticket has been re-fetched since the last triage. Open the brief with a **"Since last triage"** section listing what's changed: new comments (author + date + one-line summary), new linked issues, status moves, label changes. Then continue with the standard sections — they're recomputed from current state, not delta-only.
- Otherwise → no delta section. The brief is just the standard format.

Don't try to render a full diff. Bullets summarising what's new is enough for the reader to see what triggered the re-triage.

## Step 7 — Write `triage.md`

The brief is for whoever picks the ticket up next — usually as a Jira comment in the future. **Tight by default (~6 lines).** Expand only when the ticket warrants it. Hard cap: 15 lines.

If a brief wants to exceed 15 lines, that's a sign the routing call itself is too soft — drop confidence to Low and add a "needs human review" note in `Why` rather than padding the brief with bullets.

### Default format (overwrite any existing file)

```markdown
> *AI-generated triage — review before acting.*

# Triage: ABC-123 — Login fails after password reset

**Route:** Escalated to Experts → Dev · **Confidence:** Medium
**Why:** `NullPointerException` in description; reporter says "started after 2.3 release". Conditional — check ABC-89 first; possibly already fixed in 2.4.
**Chase before working:** environment/tenant ID, last-known-good build, whether reproducible.
**Related:** ABC-89 (possibly already resolved in 2.4 — check first).
```

### Section rules

- **Route line** — column + sub-route (Dev/Infra/BA where applicable) · `Confidence:` in one word. No reasoning here — that goes in `Why`.
- **Why** — one or two sentences. The specific signals (quote where useful) plus any conditional clause (`Conditional — first check X`). Never settle for "looks like a bug." If the route hangs on an unresolved gate, say so here, not buried below — that's an unresolved-gate Low under the confidence rubric.
- **Chase before working** — one comma-separated line, max 5 items. The things the next person needs answered to do the work. Skip the line entirely if there's nothing material to chase. Pick from the per-route anchors below; only include items genuinely missing from the ticket.
  - **Dev**: stack trace, error message, repro steps, environment/version, last-known-good
  - **Infra**: timestamps, scope (one user / many / region), what was happening (deploy / upgrade), system/service name
  - **BA**: customer's actual ask (separate from symptoms), business context, urgency/deadline
- **Related** — only includes tickets that change behaviour: a probable duplicate, a known prior fix, a same-reporter ticket in the same area that's actively in progress. One line per entry, max 3 entries. **Drop the section entirely if no entry is load-bearing** — listing irrelevant keyword matches is noise. The `similar_tickets` array from `jira.json` and the same-reporter scan from Step 3 are both inputs; apply judgment to both.

### When to expand (and how)

Each optional section appears only when load-bearing. Add nothing speculatively.

| Section | Add when |
|---|---|
| `## Problem statement` (1–2 sentences max) | The ticket title doesn't convey the gist (e.g. "Bug", "Help needed", "Issue") |
| `## Caveats` | The ticket has a structural oddity that affects how to read it: template-filled description, conflicting comments, AI-generated text, multiple issues bundled in one ticket |

Reporter history can promote into the `Related` section as an extra bullet only if at least one ticket from the same reporter is genuinely load-bearing (active, same area, recent fix). Don't list reporter history just because it exists.

### Variant: Awaiting Customer Feedback

```markdown
> *AI-generated triage — review before acting.*

# Triage: ABC-123 — <summary>

**Route:** Awaiting Customer Feedback · **Confidence:** N/A
**Why:** Ticket too thin to route — no specifics about [what / who / when].
**Ask the customer:** [specific gaps].
```

The `Ask the customer` line lists internal prompts only. **Do not draft a customer-facing message under any circumstances.** This is a hard rule even if the gaps look generic.

### Variant: Duplicate short-circuit

Three lines, no other sections:

```markdown
> *AI-generated triage — review before acting.*

# Triage: ABC-123 — <summary>

This ticket is linked as a duplicate of **ABC-100** (status: Closed). Recommend close as duplicate; BA confirms with customer.
```

If the parent is open, the recommendation reads: *"Route to wherever ABC-100 is currently being worked."*

## Common failure modes

- **Re-running on a duplicate-linked ticket** — the second run still produces the duplicate stub. That's correct behaviour, not a bug.
- **Reporter email missing from `jira.json`** — Jira sometimes hides the email. Skip the same-reporter section; don't fall back to display name.
- **`acli` not authenticated** — `jira-fetch` would have surfaced this already. If you got past pre-flight you're fine.
- **`CONTEXT.md` is huge / unfocused** — read it but don't quote it back. Use it to bias the call; mention only the specific term that mattered.
- **`similar_tickets` is full of noise** — that's expected. The keyword match is a hint, not a judgment. Drop unrelated entries silently rather than padding the brief with disclaimers.

## Future hooks (do not implement)

These are deliberately out of scope for the current version. The skill is structured so they can be added without redesign:

- **Posting the brief as a Jira comment** (posture B/C). Brief format is already comment-ready — disclaimer line first, no UI-specific markdown.
- **Setting an `ai-triaged` label** alongside posting. Required for unattended scheduled runs (so the next run can skip already-triaged tickets).
- **Reading into top related tickets** — fetch full description + resolution comments of 1-3 most plausible matches to surface prior fixes ("(c)" in the design). Currently only titles + status are used.
- **Scheduled execution against all `New` tickets**. Requires posting as a comment first; until then the brief lives only on disk and no-one reads it.
