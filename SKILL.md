# compass-close

Fast-close sub-skill for the compass loop (P25 Phase 1 — `docs/compass-close-sop.md`
in the `compass` skill).

Invoked from `/compass`'s RALF half when `path=fast`. **Phase 1 scope: fast close
only** — deep close and multi-namespace close still run inline in the parent
`~/.claude/skills/compass/SKILL.md` (its own "Deep close" / "Multi-namespace close"
sections) until Phases 2–3 of the design doc extract them too. Do not route those
paths here.

**v1 consolidation (2026-09-01, `docs/2026-08-31-consolidate-open-close-prompts-plan.md`
in the `compass` skill):** the boost check (old Step 2.1), goal completion status (old
Step 4.1), goal-to-learning mapping (old Step 4.2), tag suggestions (old Step 4.3), and
learning type/zone classification (old Step 4.4) are now one combined screen — **Step 4**
— since none of the four has a hard sequential dependency on another. Step 5's existing
checklist (already collapsed by the 2026-08-27 close-overhead audit) stays a separate
screen right after it, since its eligibility genuinely depends on Step 4's outputs
(completed goals, distilled learnings) and can't render before they exist. The artefact
capture offer (old "Step 2b.0", embedded in old Step 2.1's text) now has its own home at
Step 2.1. v2 (per-item confidence-gated auto-apply using P59 override history) is scoped
in the same plan doc but not built — see the Strategic backlog.

**Inputs (from invocation context):**
- `namespace` — the compass namespace closing
- current todo list state (inherited session context — no marshalling needed)

**Never call `compass.py close` directly as a shortcut.** Run Steps 1–6 in order.
The `close` call is the final action of Step 6, not a replacement for the steps
before it — skipping steps silently loses CLAUDE.md hygiene, code context, batched
zone/decay decisions, and learnings.

---

## Step 1 — Mark close start + narrow snapshot (no pause)

Run silently, in this order:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py mark-close-start <namespace>
python3 ~/.claude/skills/compass/scripts/compass.py close-context <namespace>
python3 ~/.claude/skills/compass/scripts/compass.py gitlog <namespace>
```

`mark-close-start` stamps `close_phase_started_at`/resets `close_phase_command_count` —
every namespace-bearing call from here to the final `close` tallies into that counter,
so the close response reports its own `close_duration_seconds`/`close_command_count`
(2026-08-27 close-overhead audit finding #6) without a hand transcript-tally.

`close-context` replaces the old `read` call — it returns `planned_actions`,
`reality_validation` (hashes), `goal_contracts`, `decay_candidates` (P2.2),
`fact_decay_candidates` (P-GC2), `retrieval_stale_candidates` (P58), and
`corpus_summary_due`: the slices this close path actually needs, instead of the full
orient context (finding #1). Hold all of these — they feed Step 4.1c (`goal_contracts`),
Step 5 item B (`reality_validation`), Step 5 item D / Step 6 (the three decay lists,
`corpus_summary_due`).

Read the current todo list state.

---

## Step 2 — Compact summary + single question

Present as a tight table, not prose:

```
## 🔄 Compass — <namespace> · fast close

✅ Done:      <completed todo items, comma-separated — "none" if empty>
⏸ Skipped:   <pending items — "none" if all done>
🔀 Git:       <N commit(s) since open — or "none">

Anything to add? (Enter to skip)
```

Wait for one response. Capture as `note`. Empty / "skip" / enter = `note = ""`.

If `note` is empty AND the todo list shows no completed items, prompt once: "Nothing recorded this session — confirm close? [Y/n]" before proceeding.

---

## Step 2.1 — Artefact capture offer (P41)

Unchanged from the parent SKILL.md's prior "Step 2b.0" — moved here (2026-09-01) so it
has its own step now that the boost check it used to sit inside has moved to Step 4.
Run after the compact summary, before code context. Trigger conditions, prompt, and
`save-artefact` call are identical to the parent's previous fast-close Step 2b.0 — see
`~/.claude/skills/compass/scripts/prompts/compass-commands.md` if you need the full text
restated; otherwise this is rare enough to keep inline knowledge of from prior sessions.

Trigger if **either**:
- **A:** note or completed todos contain "diagram", "chart", "widget", "SVG", "visual", "dashboard", "panel".
- **B (compass-dashboard namespace only):** `~/Downloads/compass-dashboard.html` modified after `last_open`.

If neither holds: skip entirely — no prompt. Otherwise offer once:
```
🖼 Visual artefact to save from this session? (title or skip)
```
- **skip / enter** → continue silently
- **title provided** → confirm tags/description inline, then `save-artefact`.

**Rule:** one prompt only — never re-offer mid-session or at ORIENT.

---

## Step 3 — Code context update

**When git commits exist this session** (from Step 1 gitlog output), auto-synthesise a
structured micro-handoff block and write it without prompting:

```
## Last updated: <YYYY-MM-DD>

**Last touched files:** <files changed in commits this session, comma-separated>
**In-progress approach:** <first incomplete todo, or continuation inferred from note>
**Next micro-step:** <first incomplete todo item — or "none" if all done>
**Tried and discarded:** <if note mentions a pivot, blocker, or abandoned approach>
```

Overwrite `~/.claude/loop/<namespace>/code_context.md` with this content. Omit the
"Tried and discarded" line if nothing in the note suggests a rejected approach.

**When no git commits exist this session:**

If `code_context.md` already exists for this namespace, ask once:
```
💻 Code context — update for next session? (active files, decisions in flight, next entry point — or skip)
```
Accept free text; overwrite with `Last updated: <date>` header + their response.
If empty / "skip" / enter, leave unchanged.

If `code_context.md` does not exist and the session note describes substantive work,
offer to create it:
```
💻 No code context file exists — create one for next session? (or skip)
```

---

## Step 4 — Consolidated close batch (v1 consolidation, 2026-09-01)

If session is open (has `planned_actions` from Step 1's `close-context` output), build
one combined screen covering the boost check (R5), goal completion status (P0.3), and
the per-learning goal-mapping (P0.2) / tag suggestions (P4.1) / type-and-zone
classification (P6, P1.1, P56) — none of these four families has a hard sequential
dependency on another, so they render together instead of as four separate gates.

**4.0 — Compute every default silently first:**

- **Boost candidates** (R5) — unchanged from the old Step 2.1: run `suggest-boosts` with
  the session context (note + completed items joined, `min_overlap: 2`, `max_results: 3`);
  if `candidates` is empty, this family contributes nothing to the screen. For each
  candidate, assess **boost** (session directly reinforced or depended on it) vs. **skip**
  (incidental keyword overlap) with a one-clause rationale — same judgment as before, just
  rendered here instead of its own screen.
- **Goal completion defaults** (P0.3 + P-GC4) — for each planned goal, check the todo list
  state and scan reality.md's most recent 'What exists and works' additions plus this
  session's git commits for shipping evidence (same cross-check the old flow ran *before*
  asking). If the todo item is marked done, or shipping evidence exists → default:
  **Completed**, citing the evidence. Otherwise there is no confident default — mark that
  item `(needs input)`; it is excluded from the bulk-accept shortcut and always needs an
  explicit letter, even when every other item is accepted via Enter.
- **Per-learning line** (P0.2 + P4.1 + P6/P1.1/P56) — for each learning distilled this
  session, combine three already-existing inference passes into one line instead of three
  separate ones: goal-origin (batch-inferred, P52, unchanged), suggested tags
  (`suggest-tags`, unchanged), and type + zone (use `inferred_zone` from `log-learning`'s
  response if present, else default type `fact` / zone `skip`).

Render:
```
🗂 Close batch — <N> item(s):

  1. [BOOST] "<learning text, first 60 chars>" [w:<weight>] → boost? (default: yes — <rationale>)
  2. [BOOST] "<learning text, first 60 chars>" [w:<weight>] → boost? (default: no — <rationale>)
  3. [GOAL 0] "<goal text, first 60 chars>" → status? (default: completed — evidence: <commit/reality match>)
  4. [GOAL 1] "<goal text, first 60 chars>" → status? (needs input — no evidence either way)
  5. [LEARNING 1] "<learning text, first 60 chars>" → goal: 0 | tags: [tooling, process] | type: fact | zone: skip (default: accept)
  6. [LEARNING 2] "<learning text, first 60 chars>" → goal: cross-goal | tags: [process] | type: procedural | zone: golden (default: accept)
  ...

Accept all defaults, or override by number (e.g. "3n" / "4:blocked" / "5:zone=warning" / "6:tags=+debugging")?
Item(s) marked "needs input" still need an explicit answer even on Enter.
```

**Parsing:** a bare **Enter** (or "y"/"yes") accepts every item that has a default;
`(needs input)` items always require their own token regardless. `<N><value>` overrides
one item: a status letter (`n`, `C`, `A`, `B`, `E`) for BOOST/GOAL items; for LEARNING
items, comma-separated `zone=<x>` / `type=<x>` / `tags=+<tag>` / `tags=-<tag>` /
`goal=<n|none>` sub-overrides on the same item (e.g. `6:zone=warning,type=procedural`).

**P-GC4 mismatch check on override:** if a GOAL item's override contradicts its
evidence-based default (default was Completed on shipping evidence, override says
Abandoned/Blocked), surface once per contradicted item before applying — this is the one
case the batch does not silently apply, same real-judgment carve-out the old pre-prompt
cross-check existed for:
```
Reality/git shows "<goal text>" shipped — still mark <status>? [Y/n]
```

**Route each family's response exactly as its original step did:**

- **BOOST** → do not flush yet. Add every entry confirmed BOOST (after any toggles) to a
  `pending_boosts` list held for the rest of this close. Step 5/6's DECAY-review boost
  sub-flow may add more entries to the same list. Flush the whole list in **one**
  `boost-learnings-batch` call right before Step 6's `close` (2026-08-28 audit finding #3
  — two separate close-time boost paths used to each flush immediately on their own,
  where the zone pattern below would have collapsed both into one call):
  ```bash
  python3 ~/.claude/skills/compass/scripts/compass.py boost-learnings-batch <namespace> \
    '{"texts": ["<exact learning text 1>", "<exact learning text 2>", ...]}'
  ```
  Skip this call entirely if `pending_boosts` ends up empty. **Boosting prior learnings
  (P4):** if this session's work reconfirmed a *prior* learning, add it to this same list
  too — never call `boost-learning` directly.

- **GOAL** → parse into a `goal_statuses` array (e.g. `["completed", "blocked", ...]`).
  For each goal marked **B**/**A**, immediately follow up (Step 4.1a below) for a reason
  — that needs new user-supplied content, so it stays a real follow-up, not a bulk default.

- **LEARNING** → apply goal_origin / tags / type / zone directly from the (possibly
  overridden) line. **Rule:** do not build the close payload until every learning has a
  `goal_origin` value. If the user confirms a zone for a learning already logged this
  close (or a prior session's), do not call `set-learning-zone` immediately — accumulate
  `{"text": ..., "zone": ...}` into a `pending_zone_assignments` list held for the rest of
  this close (Step 5's checklist and Step 6 may both add more entries, e.g. a prior-session
  learning confirmed via the DECAY sub-flow). Flush the whole list in **one** batched call
  right before Step 6's `close`:
  ```bash
  python3 ~/.claude/skills/compass/scripts/compass.py set-learning-zones-batch <namespace> \
    '[{"text": "<exact text 1>", "zone": "golden"}, {"text": "<exact text 2>", "zone": "warning"}]'
  ```
  This is finding #3 from the 2026-08-27 close-overhead audit — a shell loop over
  individual `set-learning-zone` calls still forks one subprocess per call even when
  batched into fewer *tool* calls; one script invocation for the whole close does not.
  Skip the call entirely if `pending_zone_assignments` ends up empty. For **hypothesis**
  type learnings, additionally ask confidence (high/medium/low) and test window (N days)
  as an immediate follow-up — new content, same carve-out as the goal reason capture.

### Step 4.1a — Reason capture for blocked/abandoned goals (P14)

For each goal marked **B** (blocked) or **A** (abandoned) at Step 4, immediately follow up:
```
Goal <N>: "<goal text>" → B (blocked)
  Brief reason? (or enter to skip)
```
Capture the response. Include in the close payload `incomplete` array as a dict:
```json
{"text": "<goal text>", "status": "blocked", "reason": "<user reason>"}
```
If the reason describes a deliberate reprioritization, offer to log it via `log-decision`.

### Step 4.1c — Verification contract scoring (P55)

Check `goal_contracts` in the close-context output *(carried into this step's context
from Step 1 — no second read needed)*. If no contracts exist for any completed goal,
skip silently.

For each completed goal **with a contract** (match by goal_text hash):
```
📋 Contract check — "<goal text>"
Criteria:
1. <criterion 1>
2. <criterion 2>
3. <criterion 3>

Which were met? (comma-separated numbers, or "all" / "none")
```
Capture response; call `verify-goal-contract` with `verified_criteria`/`unmet_criteria`.
Surface: `✓ Contract score: <N>/<total> criteria verified.`

### Step 4.5 — Cross-namespace learning conflicts (P2.1) — mandatory

Must run before the close payload is built:

```bash
python3 ~/.claude/skills/compass/scripts/compass.py log-learnings-batch <namespace> \
  '[{"text": "<learning text 1>", "tags": [...], "reconcile": true},
    {"text": "<learning text 2>", "tags": [...], "reconcile": true}]'
```

Walk the returned `results` in order (matched by `text`):

- **`ok: true`, no `reconciliation_action`** → already recorded; pass `"learnings": []`
  in the close payload for these.
- **`reconciliation_action: "pending"`** → near-duplicate found, write withheld:
  ```
  💭 Near-duplicate detected (Jaccard: <score>):
    New:      "<new text, first 80 chars>"
    Existing: "<existing text, first 80 chars>"
    Rewrite (merge) / Keep both / Suppress new?
  ```
  If `rewrite`: keep every concrete fact from both texts — never drop a name, number,
  date, condition, or exception assuming it's redundant. Prefer `keep_both` over a
  lossy rewrite when genuinely unsure. Apply via `apply-learning-reconciliation`.
- **`contradiction_candidate` present** (non-blocking advisory):
  ```
  ⚡ Possible contradiction (Jaccard: <score>):
    New:      "<new text, first 80 chars>"
    Existing: "<existing text, first 80 chars>"
    Mark existing as superseded by new? [Y / N]
  ```
  If `golden_zone_conflict: true`, prefix with `⚠ Golden-zone pattern being
  contradicted —` and treat with more urgency (still Y/N, never auto-supersede).
  **Y** → `approve-dream-merge` with the new text as anchor.
- **`require_conflict_resolution: true`** → conflict in another namespace:
  ```
  ⚠ Learning exists in other namespace(s):
  "<learning text>"
    Found in: global (weight: 2, tags: [process, architecture])
  How to handle? M: Merge / D: Diverge / L: Link
  ```
  **M** → increment weight in the source namespace, don't duplicate here.
  **D** / **L** → call `resolve-learning-conflict` directly with the full learning
  payload (`decision: "diverge"|"link"`) — it writes the entry itself. **Do not**
  call `log-learning` afterwards; it would re-raise the identical conflict.

### Step 4.6 — Intent change reason (P1.2)

Skip if drift was handled at ORIENT. If intent changed mid-session and no reason
recorded yet, ask and call `set-intent`.

All three data types get stored: goal completion stats → trend; goal_origin →
learning-to-goal mapping; hypothesis metadata → validation surfacing.

---

## Step 5 — Consolidated close checklist (finding #2 collapse)

The 2026-08-27 close-overhead audit's finding #2: six advisory prompts — boost check
(now handled at Step 4, which fires as soon as the session's completed work is known,
before the learnings distilled there feed this step's own eligibility checks) plus
**CLAUDE.md reflection (P5.1), outcome-link offer (P49), cross-namespace propagation,
skill-feedback (P52), and retrieval-stale decay review (P58)** — used to fire as five
*separate* sequential Y/N gates at the tail of every close. All five now share **one**
compute-then-render pass, so the common case (every goal completed, nothing
contradictory, no friction to report) costs one screen instead of five round trips. This
screen still can't merge into Step 4's — its eligibility genuinely depends on Step 4's
outputs (which goals completed, what got distilled), so it has to render after them.

**Compute all five silently** (each reuses data already in hand — no full `read`):

- **A — CLAUDE.md reflection:** gate on `repo_path` configured AND `CLAUDE.md` exists
  at that root. If gate passes, scan git commits/completed todos for module-map gaps
  and this close's distilled learnings for `tooling`/`debugging`-tagged gotchas. **Skip
  entirely (no signal) if the user already manually updated CLAUDE.md this session**
  (visible in git diff or mentioned in the note) — do not propose duplicate changes.
- **B — Outcome-link offer:** eligible if any goal was marked **C** at Step 4.
  **Outcome links are same-namespace only** — never attempt to link a completed goal
  to a reality bullet in another namespace, even if the fragment superficially matches.
- **C — Cross-namespace propagation:** eligible if any distilled learning carries a
  broad platform tag (architecture, process, tooling, testing, performance,
  integration, debugging, spring, kafka, redis).
- **D — Decay review:** from Step 1's `close-context` output — `decay_candidates`
  (P2.2, unvalidated hypotheses >60 days old), `fact_decay_candidates` (P-GC2,
  unreinforced weight-1 facts), and `retrieval_stale_candidates` (P58, surfaced often
  but never reinforced) — all three already fetched, no second call.
- **E — Skill-feedback:** always eligible, but reconcile first — call
  `get-skill-feedback <namespace>`, and if any `"status": "open"` entry's fix is
  already confirmed by this close's reality.md text, `action-skill-feedback` it
  silently before considering it for the checklist.

**If none of A–E have signal:** skip this whole step — no output — proceed to Step 6.

**Otherwise**, render only the items with actual signal as one screen:

```
🧾 Close checklist — <N> item(s):

  1. [CLAUDE.md] <N> candidate update(s) detected → apply? (default: yes)
  2. [OUTCOME]   <N> goal(s) completed → link to reality bullets? (default: yes)
  3. [PROPAGATE] "<learning, condensed>" tagged `<tag>` → push to global? (default: yes)
  4. [DECAY]     <N> decay/retrieval-stale candidate(s) → review for discard? (default: review)
  5. [FEEDBACK]  any protocol friction to log? (default: no)

Accept all defaults, or override by number (e.g. "1n 5:felt slow at step 3")? [Enter to accept all]
```

Number only the items actually rendered (skip letters with no signal — e.g. if only
C and E have signal, show them as 1 and 2).

Map the response:
- **Enter / "y" / "yes"** → apply every rendered item's default action.
- **Per-item override** (`"1n"` flips item 1's action to no; `"5:<text>"` supplies
  FEEDBACK's text directly) → apply the override, defaults for the rest.

**Then run each accepted item's full existing sub-flow — this step only collapses the
gating decision, not the underlying mechanics or script calls:**

- **CLAUDE.md → Y:** apply inline (module map entry and/or gotcha section). Use
  `claude-md-management:revise-claude-md` for broad changes; edit inline for 1–2 lines.
  `N`/default-declined → skip silently, do not re-surface.
- **OUTCOME → Y:** for each completed goal, ask which reality bullet(s) it produced —
  match the fragment against Step 1's `close-context` `reality_validation` hashes
  (already fetched, no `read`), **this namespace only** — never a watched or otherwise
  related namespace, even on a superficial text match. Call `link-goal-outcome` per
  match, then `get-outcome-stats` and confirm the updated rate. `n`/default-declined → skip.
- **PROPAGATE → Y:** read `~/.claude/skills/compass/scripts/prompts/cross-namespace-propagation.md`
  and follow it (covers the Y/Y-generic/n sub-choice and the `log-learning global` call).
- **DECAY → "review":** present the batched decay list exactly as Step 6 below
  describes, collecting Y/N per candidate. `N`/decline-all → skip the
  `approve-learning-decay` call entirely.
- **FEEDBACK → Y (or inline text):** if no text was supplied inline, ask *"One
  sentence: what felt slow, broken, or missing?"* then the step_ref/failure-dimension
  follow-up exactly as before, then `log-skill-feedback`. Declined/no signal → skip.

**Rule:** one screen, one round trip for the fully-nominal case. Per-item drill-in
only for items the user overrides or that inherently need free text.

---

## Step 6 — Auto-distil, decay, corpus health, reality, and close

**Reality rescue (no-todos guard):** if completed todos are empty AND `note` is
non-empty, work happened but was never tracked:
```
⚠ No completed todos, but your note describes work.
Derive a reality update from it? [Y/n]
```
**Y**/enter → extract 1–3 bullets describing what now exists and works. These are pure
additions — prefer one `append-reality-bullet <namespace> "## What exists and works"
"<bullet text>"` call per bullet (2026-08-28 audit finding #4) over the close payload's
`reality` field, which requires reconstructing the whole document. Reserve the
`reality` field / `update-reality` for cases that also reword existing bullets or
restructure sections. **N** → skip; state this explicitly at
the end.

**Auto-distil:** from completed todos, git log, and any user note, distil **one**
durable learning automatically — do not present it for approval unless the note
contains something non-obvious that warrants a check.

**History snapshot contract:** the close payload MUST include `completed` and
`incomplete` arrays from the current todo list state. If the todo list is empty,
derive `completed` from goals marked **C** at Step 4 instead.

**Learning decay (P2.2 + P-GC2 + P58):** if Step 5's DECAY item was accepted (or no
checklist fired because DECAY was the only signal and defaulted through), present the
batched list:
```
🗑 Decay candidates (<N>):

- "<text>" (65d ago, confidence: low)                       [hypothesis, P2.2]
  Discard? Y/N
- "<text>" (20 sessions unreinforced, tags: [tooling])       [fact, P-GC2]
  Y/N?
- "<text>" (surfaced 22 times, weight still 1)               [retrieval-stale, P58]
  Y/N?
...
```
Collect all **Y** into one `approve-learning-decay` call. If a removed text's response
carries non-empty `cross_namespace_refs`/`community_published`, surface the P63a
decay-cascade advisory once, batched, exactly as before.

**Corpus health (P10):** from Step 1's `close-context` output, `corpus_summary_due` —
already fetched, no second call:
- `false` → skip silently.
- `true` → `summarise-learnings <namespace>`; if `summarisation_needed: true`, offer
  distillation review exactly as the parent SKILL.md previously documented (group by
  theme, `approve-learning-decay` with `approved_ids`, `log-learning` with `weight: 3`
  per synthesis entry). Max once per 5 sessions per namespace.

**Deferred escalations (P1.3):** if any ORIENT escalation candidates weren't
completed, `defer-opportunity` for each.

**Flush pending zone assignments** (Step 4) via `set-learning-zones-batch` now, if
the list is non-empty and hasn't been flushed yet.

**Flush pending boosts** (Step 4) via `boost-learnings-batch` now, if
`pending_boosts` is non-empty and hasn't been flushed yet.

**Reality and close:**

`cmd_close`'s payload `reality` field has exactly one no-op case: an empty string.
Any non-empty value unconditionally overwrites `reality.md`, even if `update-reality`
was already called separately this close. If reality was already written via a prior
`update-reality` call, the `close` payload's `reality` field **must** be `""`.

**Prefer the single-bullet primitives when the change is a pure add or remove**
(2026-08-28 audit finding #4 — `update-reality` reads the whole document out,
requires reconstructing it in full, and writes it all back, so its cost scales with
total reality.md size, not the size of the actual edit):
```bash
python3 ~/.claude/skills/compass/scripts/compass.py append-reality-bullet <namespace> '<section_header>' '<bullet text>'
python3 ~/.claude/skills/compass/scripts/compass.py remove-reality-bullet <namespace> '<bullet_hash>'
```
One call per bullet — a "move" (e.g. a Backlog item shipping) is one `remove` plus one
`append`. Use `update-reality` only when a change also reworks existing bullets' wording
in place or restructures sections (e.g. a corpus distillation pass) — the reword-drift
carryover logic only runs on that full-document path.

If a full rewrite is genuinely needed, write it via the script — never write
reality.md directly. Pass the content by writing it to a file first and reading the
file back into the call, never via inline `$(...)` shell substitution:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py update-reality <namespace> '<new_reality_text>'
```
If this trips the bullet-collapse guard on a genuinely intentional rewrite (a large,
deliberate condensing of reality, not a broken substitution), re-run with `--force`.

**Check the response's `advisory` field (2026-09-02).** If non-null, `update-reality`
detected a pure addition (0 removed, 0 reworded) — exactly the shape
`append-reality-bullet` exists for. Surface it plainly rather than silently ignoring
it: `⚠ <advisory text>` — this is what makes the guidance above self-correcting instead
of relying on memory alone.

Auto-verify the subset of the response's `new_bullet_hashes` that correspond to
bullets promoted to "What exists and works" this session:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py verify-reality <namespace> \
  '<JSON array of new_bullet_hashes for completed-todo-derived bullets>'
```

**Prompt-count tally (2026-09-01, `docs/2026-08-31-consolidate-open-close-prompts-plan.md`
in the `compass` skill):** count every distinct interactive screen actually presented
across this sub-skill's Steps 2–5 (Step 4's batch counts as **one**, not one per family;
Step 5's checklist counts as one whether or not it renders). Include this total as
`close_prompt_count` in the payload below, alongside `open_prompt_count` carried forward
from `compass-open`'s own tally (omit either key if it genuinely wasn't tracked this
session — e.g. a deep-close or multi-namespace-close path that didn't go through
`compass-open`'s Step 4.6 hand-off). Write-only this session — no script or trend reads
these yet, see the Strategic backlog for the deferred read-side `avg_last_5`.

Then close:
```bash
python3 ~/.claude/skills/compass/scripts/compass.py close <namespace> '<payload_json>'
```
`<payload_json>` includes `"open_prompt_count": <N>, "close_prompt_count": <M>` alongside
the fields already documented above.

The response includes `close_duration_seconds`/`close_command_count` when
`mark-close-start` ran at Step 1 (it always does, in this sub-skill) — no action
needed, this is what a future overhead audit reads instead of hand-tallying. It also
echoes `open_prompt_count`/`close_prompt_count` back when supplied.

---

## After close (still in this sub-skill, before returning)

Run, in order, exactly as the parent SKILL.md's standalone top-level sections
document them (not duplicated here — same files, same triggers):

1. **Skill patch check** — `~/.claude/skills/compass/SKILL.md`'s "## Skill patch
   check" section.
2. **Close-time skill harvesting** (complexity trigger) — read
   `~/.claude/skills/compass/scripts/prompts/skill-opportunity-detection.md`
   (Close-time section) if the complexity trigger fires.

(Cross-namespace propagation already ran, if eligible, inside Step 5's checklist —
do not run it again here.)

Confirm to the user:
```
✓ Closed. <N> learning(s) recorded. Reality updated (or "unchanged").
Next: <first bullet of updated reality>
```

## Return

```json
{"result": "closed", "namespace": "<ns>", "learnings_count": N, "reality_updated": true}
```

Done. No further prompts.
