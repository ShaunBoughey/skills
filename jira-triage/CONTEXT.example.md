<!--
TEMPLATE — this file is `CONTEXT.example.md`, not `CONTEXT.md`.
The /jira-triage skill only reads `./CONTEXT.md` in the user's cwd.
This file exists as scaffolding only and is never loaded.

To use:
  1. Copy this file to your project root as `CONTEXT.md`.
  2. Replace example bullets below with your real org context.
  3. Anything left in HTML comments stays inert — the skill reads the
     raw file, but inline examples marked "Example:" are clearly
     placeholders, not directives.

Keep CONTEXT.md short. A few sharp rules outperform a wall of context.
Long files dilute the signal and slow every triage run.

Sections below are suggestions. Drop, rename, or reorder freely.
-->

# CONTEXT.md

<!--
## Glossary

Map system / product / tenant names the model can't infer from a ticket
alone. Format: `name — what it is — usual route hint (optional)`.

Example:
- Atlas — internal tax-rules engine — Dev (changes are code, not config)
- "the platform" — shared infra layer (Kubernetes, ingress, auth) — Infra
- DEMOCO — sandbox tenant used by sales for demos; ignore alerts from here
-->

<!--
## Routing biases

Project-specific overrides for the default routing rules. Each one says
WHAT to look for and WHERE to send it. Use sparingly — every bias the
model has to remember is a bias it can misapply.

Example:
- Anything mentioning "deploy pipeline", "CI", or "GHA" → Infra,
  regardless of whether there's a stack trace.
- Tax-rate questions for region X → BA, even if phrased as a bug.
- Tickets labelled "Tier1" → Tier1 customer; mention this in `Why`
  so the next person treats it as priority.
-->

<!--
## Tenant / customer conventions

Reporter-side patterns the model wouldn't otherwise know.

Example:
- acme.com email domain = customers from the acquired Acme Corp.
  They paste Salesforce case IDs in descriptions — that's the SF
  case ref, not a Jira link.
- finance@example.com is a known power user; their thin tickets
  are unusual and worth flagging in Caveats.
-->

<!--
## Active incidents / known issues

Short-lived context. Prune aggressively — stale entries here cause
the worst kind of routing error (wrong because of yesterday's truth).

Example:
- 2026-05-04: Print queue outage across all London offices.
  Any printer ticket from London this week → link to SAM1-200,
  do not route fresh.
-->

<!--
## Team conventions

How the team operates. Affects framing in `Why` and `Chase`, not
the routing decision itself.

Example:
- "Escalated to Experts" reviewed at daily standup; tickets in BA
  queue >5 days get escalated automatically — mention age in Why
  if relevant.
- We don't use Won't Fix; close as "Resolved: Out of scope" instead.
- Every ticket needs a customer-comms note before it can move to
  Resolved — not the AI's job, but worth knowing the next person
  has that step ahead of them.
-->
