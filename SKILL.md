# compass-close

Fast-close sub-skill for `/compass`, invoked from its RALF half when `path=fast`. Deep and
multi-namespace close run from the router's `scripts/prompts/router-sections.md`, not here.
**Input:** `namespace`, plus the todo list state (no TodoWrite-equivalent → `goal_progress`).

Run Steps 1–6 in order; `close-apply` is the last action. Never call `close` as a shortcut.
Maintenance (skill harvesting, skill patches, SkillOpt, friction logging, corpus summarisation)
never runs here: it is `/compass maintain`.

---

## Step 1 — Prepare (no pause)

```bash
~/.claude/skills/compass/scripts/compass.sh close-prepare <namespace> '{"context": "<note + completed items>", "compact": true}'
```
`compact: true` drops `gitlog` and `ground` (`brief_markdown` has them) so the reply reads whole.
Never cut it with `head`: the candidates sit near the end. A repeat call is safe.
It stamps the close-phase start and returns `close_context`, `gitlog`, `reality_sections`,
`backlog_match`, `boost_candidates` and `brief_markdown` (keys:
`~/.claude/skills/compass/scripts/compass.sh schema close-prepare`). A failed part is listed under
`errors` with a null key: run that command by hand. The host todo tool, if any, beats `goal_progress`.

## Step 2 — Summary and close batch (one screen)

Compute the defaults silently first:
- **Goals:** marked done, or shipping evidence in this session's commits → **completed** (cite it).
  Otherwise `(needs input)`, never defaulted.
- **Boosts:** per `boost_candidates` entry, **boost** if the session relied on it, else **skip**,
  with a one-clause reason. Keep each entry's `learning_id`: Step 6 boosts by id, not by text.
- **Learnings:** distil the session's durable learnings (usually one; at most three). Per learning:
  goal-origin, tags and zone from `suggest-tags` per learning (`inferred_zone`; `null` renders as
  `none`, never `warning`). Show a hand-set zone as `warning (override)`.

```
## 🔄 Compass — <namespace> · fast close
<brief_markdown lines>

🗂 Close batch:
  1. [GOAL 0] "<goal, 60 chars>" → completed (evidence: <commit>)
  2. [GOAL 1] "<goal, 60 chars>" → (needs input)
  3. [BOOST] "<learning, 60 chars>" [w:N] → yes — <reason>
  4. [LEARNING] "<learning, 60 chars>" → goal: 0 | tags: [process] | type: fact | zone: none
Enter accepts all, or override by number ("2:B", "3n", "4:zone=warning"), or "note: <text>".
GOAL letters: C completed, S skipped, A abandoned, B blocked.
```

Enter never answers a `(needs input)` goal: ask once more for just that goal. If a GOAL override
contradicts shipping evidence, confirm once. If nothing was completed and the note is empty, ask
"Nothing recorded this session — confirm close? [Y/n]" instead.

## Step 2.1 — Artefact capture

Only if the note or completed todos mention a diagram, chart, widget, SVG, visual, dashboard or
panel: load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 2.1 — Artefact capture offer (P41)"`.

## Step 3 — Code context

With commits this session, write without asking (`write-code-context` on stdin, never the Write
tool):
```bash
~/.claude/skills/compass/scripts/compass.sh write-code-context <namespace> <<'EOF'
## Last updated: <YYYY-MM-DD>
**Last touched files:** <files> · **Next micro-step:** <first incomplete todo, or "none">
**Tried and discarded:** <only if the note mentions a pivot>
EOF
```
No commits: leave `code_context.md` alone.

## Step 4 — Learnings and goal reasons

For each goal marked **B** or **A**, load
`~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.1a — Reason capture for blocked/abandoned goals (P14)"`.

### Step 4.5 — Log learnings with conflict check (mandatory)

```bash
~/.claude/skills/compass/scripts/compass.sh log-learnings-batch <namespace> - <<'EOF'
[{"text": "<learning>", "tags": [...], "learning_type": "fact", "reconcile": true}]
EOF
```
Keep each `learning_id` for the zone and boost lists. If a result carries
`reconciliation_action: "pending"`, a `contradiction_candidate` or `require_conflict_resolution`,
load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.5 — Result handling (reconciliation, contradiction, cross-namespace conflict)"`.

### Step 4.6 — Intent change

If intent changed mid-session with no reason recorded, ask once and call `set-intent`.

## Step 5 — Close checklist (only items with signal)

- **CLAUDE.md:** `repo_path` has a `CLAUDE.md` and this session's commits or learnings expose a gap
  the user has not already fixed.
- **PROPAGATE:** a learning carries a broad platform tag (architecture, process, tooling, testing,
  performance, integration, debugging, spring, kafka, redis).
- **DECAY:** `close_context` lists `decay_candidates`, `fact_decay_candidates` or
  `retrieval_stale_candidates`.
- **BACKLOG MATCH:** one line per `backlog_match` candidate (never `excluded_same_session`).

No signal → skip the step. Otherwise one screen, numbered by what renders:
```
🧾 Close checklist:
  1. [CLAUDE.md]     <N> update(s) → apply? (default: yes)
  2. [PROPAGATE]     "<learning>" → push to global as: "<generalised wording>"? (default: yes)
  3. [DECAY]         <N> candidate(s) → review? (default: review)
  4. [BACKLOG MATCH] "<item>" looks resolved → clear it? (default: yes)
Enter accepts all, or override by number.
```
**Never write to `global` unseen.** `propagate` is read by every project. Include it only after the
user has seen item 2 and accepted it. If they cannot confirm, omit it and add a Tactical backlog bullet
("Propagate: <learning>").

Accepted items, sections batched into one call per file:
`close-conditional-flows` → `"Step 5 item — CLAUDE.md"`, `"Step 6 — Learning decay review"`,
`"Step 5 item — BACKLOG MATCH"`; propagation →
`~/.claude/skills/compass/scripts/compass.sh prompt cross-namespace-propagation --section "From the close checklist" --section "With close-apply"`.

## Step 6 — Reality and close

If no todo was completed but the note is non-empty, first load
`~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Reality rescue (no-todos guard)"`.

**Reality rule:** `reality.md` says what is true now, not what happened; git log is the changelog.
Add a "What exists and works" bullet only when a capability a future session must know about
appeared or changed, and remove the bullets it supersedes. Keep that section under ~20 bullets.

One `close-apply` call with `{zones, boosts, propagate, reality_ops, screens, close}` (plus `decay`
when accepted). Shapes: `~/.claude/skills/compass/scripts/compass.sh help close-apply`.
- `reality_ops`: one op per add, remove or replace (`replace` edits in place when a
  capability changed), including BACKLOG MATCH removals (`"position": "first"` on an append op starts the section); a `section` must be a
  leaf from `reality_sections`; `close.reality` is then `""`.
- `boosts`: pass `{"learning_ids": [...]}` using the `learning_id` from `boost_candidates`. Never pass
  `texts` copied from the screen or from `boost_candidates`: both are cut to ~60-80 characters, a text
  match needs the full wording, and the call fails with "Learning not found".
- `close` must carry `completed` and `incomplete` arrays from the todo state (or the goals marked C).
- `screens`: the screens that waited for a response (`batch`, `reason`, `conflicts`, `intent`,
  `checklist`, `decay`, `artefact`), or `[]`.
- Then `verify-reality <namespace> '<JSON array>'` on the response's `unverified_hashes`.

A full-rewrite `update-reality` or a deep close instead loads
`~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Close without close-apply"`.

Confirm: `✓ Closed. <N> learning(s) recorded. Reality updated (or "unchanged").` Then return
`{"result": "closed", "namespace": "<ns>", "learnings_count": N}`.
