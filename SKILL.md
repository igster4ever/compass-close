# compass-close

Fast-close sub-skill for `/compass` (P25; `docs/compass-close-sop.md` in the `compass` skill),
invoked from its RALF half when `path=fast`. Deep close and multi-namespace close still run from
the router's `scripts/prompts/router-sections.md`; do not route them here.

**Consolidation (v1, 2026-09-01, `docs/2026-08-31-consolidate-open-close-prompts-plan.md`):** the
boost check, goal completion status, goal-to-learning mapping, tag suggestions and learning
type/zone classification are one screen, **Step 4**. Step 5's checklist stays a separate screen
because its eligibility depends on Step 4's outputs. The artefact capture offer lives at Step 2.1.

**Input:** `namespace`, plus the todo list state (inherited context; with no TodoWrite-equivalent
tool, Step 1 falls back to `state.json`'s `goal_progress`).

**Never call `compass.py close` directly as a shortcut.** Run Steps 1–6 in order: `close` /
`close-apply` is the last action of Step 6, and skipping steps silently loses CLAUDE.md hygiene,
code context, batched zone/decay decisions and learnings.

---

## Step 1 — Mark close start + narrow snapshot (no pause)

Run silently, one call, with the draft session context (note + completed items joined):
```bash
~/.claude/skills/compass/scripts/compass.sh close-prepare <namespace> '{"context": "<note + completed items>"}'
```

`close-prepare` replaces `mark-close-start`, `close-context`, `gitlog`, `get-skill-feedback`,
`match-backlog` and `suggest-boosts` (keys: `compass.sh schema close-prepare`). It stamps
`close_phase_started_at` and resets `close_phase_command_count` (so the close response reports
`close_duration_seconds`/`close_command_count` without a hand tally), and returns `close_context`,
`gitlog`, `skill_feedback`, `backlog_match`, `boost_candidates` and `skipped_gates`. A part that failed is listed
under `errors` with a null key: run that command by hand. If the final note differs a lot from
the draft context, re-run `suggest-boosts` / `match-backlog` with the final text.

`close_context` holds only the slices this close needs (key names: `compass.sh schema
close-context`): `planned_actions`, `reality_validation` (hashes), `session_appended_bullets`
(`{hash, text, section}`), `goal_contracts`, `decay_candidates` (P2.2), `fact_decay_candidates`
(P-GC2), `retrieval_stale_candidates` (P58) and `corpus_summary_due`. Hold them: Step 4.1c uses
`goal_contracts`, Step 5 item B `reality_validation`, Step 5 item D and Step 6 the decay lists and
`corpus_summary_due`.

Read the todo list state from the runtime's TodoWrite-equivalent tool if it has one; otherwise
cross-reference `close-context`'s `planned_actions` against `read`'s `goal_progress` (the
host-independent substitute `set-goal-status` updates as goals complete; a real session found no
TodoWrite tool at all, 2026-09-02).

---

## Step 2 — Compact summary + single question

Present as a tight table, not prose:

```
## 🔄 Compass — <namespace> · fast close

✅ Done:      <completed todo items, comma-separated — "none" if empty>
⏸ Skipped:   <pending items — "none" if all done>
🔀 Git:       <N commit(s) since open — or "none">
⏭ Gates:     <skipped_gates: "research (+4 overdue, deferred 2×), dream (+1 overdue)" — omit the line when empty>

Anything to add? (Enter to skip)
```

Wait for one response. Capture as `note`. Empty / "skip" / enter = `note = ""`.

If `note` is empty AND the todo list shows no completed items, prompt once: "Nothing recorded this session — confirm close? [Y/n]" before proceeding.

**One screen with Step 4 (default):** when the session is open and nothing needs free text first —
no `(needs input)` goal, and not the "Nothing recorded" case above — compute Step 4.0, then show
this table and Step 4's batch together. Replace the "Anything to add?" line with the batch's
last line, extended: `or "note: <text>" to add a note (Enter accepts all defaults, empty note)`.
Overrides come first, the note after `note:`. Tally one `batch` screen, no `summary`. Keep the two
screens (this one, then Step 4) otherwise.

---

## Step 2.1 — Artefact capture offer (P41)

Run after the compact summary, before code context. Trigger if **either**: A) the note or completed todos contain "diagram", "chart", "widget", "SVG", "visual", "dashboard" or "panel"; B) (`compass-dashboard` namespace only) `~/Downloads/compass-dashboard.html` was modified after `last_open`. Neither → skip, no prompt. Otherwise load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 2.1 — Artefact capture offer (P41)"` and follow it.

---

## Step 3 — Code context update

**When git commits exist this session** (from Step 1's `gitlog`), auto-synthesise a
structured micro-handoff block and write it without prompting:

```
## Last updated: <YYYY-MM-DD>

**Last touched files:** <files changed in commits this session, comma-separated>
**In-progress approach:** <first incomplete todo, or continuation inferred from note>
**Next micro-step:** <first incomplete todo item — or "none" if all done>
**Tried and discarded:** <if note mentions a pivot, blocker, or abandoned approach>
```

Omit the "Tried and discarded" line if nothing in the note suggests a rejected approach.

Write every `code_context.md` (here and below) with the script, never the Write tool (it fails
on an existing file that was not Read first). Pipe the content on stdin, or pass `--file <path>`:
```bash
~/.claude/skills/compass/scripts/compass.sh write-code-context <namespace> <<'EOF'
<block>
EOF
```

**When no git commits exist this session:**

If `code_context.md` already exists for this namespace, ask once:
```
💻 Code context — update for next session? (active files, decisions in flight, next entry point — or skip)
```
Accept free text; write it with a `Last updated: <date>` header + their response.
If empty / "skip" / enter, leave unchanged.

If `code_context.md` does not exist and the session note describes substantive work,
offer to create it:
```
💻 No code context file exists — create one for next session? (or skip)
```

---

## Step 4 — Consolidated close batch (v1 consolidation, 2026-09-01)

If the session is open (`planned_actions` from Step 1's `close-context`), build one screen (shared
with Step 2's table by default, see there)
covering the boost check (R5), goal completion status (P0.3) and the per-learning goal-mapping
(P0.2) / tag suggestions (P4.1) / type-and-zone classification (P6, P1.1, P56). None of the four
depends on another, so they render together.

**4.0 — Compute every default silently first:**

- **Boost candidates (R5):** use Step 1's `boost_candidates` (`suggest-boosts` defaults:
  `min_overlap: 2`, `max_results: 3`); no `candidates` → no BOOST lines. Per
  candidate, judge **boost** (the session directly reinforced or depended on it) vs **skip**
  (incidental keyword overlap), with a one-clause rationale.
- **Goal completion defaults (P0.3 + P-GC4):** per planned goal, check the todo state (or
  `goal_progress[index]`, Step 1) and scan reality.md's latest 'What exists and works' additions
  plus this session's commits for shipping evidence. Marked done, or evidence found → default
  **Completed**, citing it. Otherwise no confident default: mark it `(needs input)`, excluded from
  bulk accept.
- **Per-learning line (P0.2 + P4.1 + P6/P1.1/P56):** for each learning distilled this session,
  one line combining goal-origin (batch-inferred, P52), suggested tags (`suggest-tags`) and type +
  zone. Take the zone from `suggest-tags`' `inferred_zone` (pass `learning_type` in its payload
  when the type is not `fact`), since `log-learning` runs only after this screen. `null` renders
  as zone `none`; never render `skip` (`set-learning-zones-batch` rejects it).

Render:
```
🗂 Close batch — <N> item(s):

  1. [BOOST] "<learning text, first 60 chars>" [w:<weight>] → boost? (default: yes — <rationale>)
  2. [BOOST] "<learning text, first 60 chars>" [w:<weight>] → boost? (default: no — <rationale>)
  3. [GOAL 0] "<goal text, first 60 chars>" → status? (default: completed — evidence: <commit/reality match>)
  4. [GOAL 1] "<goal text, first 60 chars>" → status? (needs input — no evidence either way)
  5. [LEARNING 1] "<learning text, first 60 chars>" → goal: 0 | tags: [tooling, process] | type: fact | zone: none (default: accept)
  6. [LEARNING 2] "<learning text, first 60 chars>" → goal: cross-goal | tags: [process] | type: procedural | zone: golden (default: accept)
  ...

Accept all defaults, or override by number (e.g. "3n" / "4:blocked" / "5:zone=warning" / "6:tags=+debugging")?
Letters — BOOST: y / n · GOAL: C completed, S skipped, A abandoned, B blocked.
Item(s) marked "needs input" still need an explicit answer even on Enter.
```

**Parsing:** a bare **Enter** (or "y"/"yes") accepts every item that has a default. Enter alone
leaves a `(needs input)` goal unanswered, so ask once more with just that goal and the letters
(`Goal 1 — C/S/A/B?`); never default it. `<N><value>` overrides one item: `y`/`n` for BOOST; a
status letter for GOAL (`C`→`completed`, `S`→`skipped`, `A`→`abandoned`, `B`→`blocked`, the
`goal_statuses` values; the old `E` had no meaning in the script; a full status word also works);
for LEARNING, comma-separated `zone=<x>` / `type=<x>` / `tags=+<tag>` / `tags=-<tag>` /
`goal=<n|none>` on the same item (e.g. `6:zone=warning,type=procedural`).

**P-GC4 mismatch check:** if a GOAL override contradicts its evidence-based default (default
Completed on shipping evidence, override Abandoned/Blocked), ask once per such item before
applying: `Reality/git shows "<goal text>" shipped — still mark <status>? [Y/n]`. This is the one
case the batch does not silently apply.

**Route each family's response:**

- **BOOST** → do not flush yet. Add each confirmed BOOST to a `pending_boosts` list held for the
  whole close (Step 5/6's DECAY-review boost sub-flow may add more, and so may a *prior*
  learning this session reconfirmed (P4); never call `boost-learning` directly). Use the
  `learning_id` that `log-learnings-batch` (or `surface-learnings` / a `suggest-boosts` candidate)
  returned, not the text. A candidate with `learning_id: null` is a legacy entry: put its
  `full_text` (`text` is cut at 80 characters) in `texts`. Flush once, right before `close`:
  `boost-learnings-batch <namespace> '{"learning_ids": [...], "texts": [...]}'`, or as `close-apply`'s
  `boosts` key. Skip the call if the list ends up empty.

- **GOAL** → build the `goal_statuses` array (e.g. `["completed", "blocked", ...]`). For each goal
  marked **B**/**A**, follow up at Step 4.1a for a reason: that needs new user content, so it
  stays a real follow-up.

- **LEARNING** → apply goal_origin / tags / type / zone from the (possibly overridden) line. **Do
  not build the close payload until every learning has a `goal_origin`.** A zone confirmed for a
  learning already logged this close (or a prior session's) is not applied immediately: add
  `{"learning_id": ..., "zone": ...}` to a `pending_zone_assignments` list held for the whole close
  (Step 5/6 may add more) and flush once, right before `close`:
  `set-learning-zones-batch <namespace> '[{"learning_id": "<id>", "zone": "golden"}, ...]'`, or as
  `close-apply`'s `zones` key. Skip it if the list is empty. For **hypothesis** learnings also ask
  confidence (high/medium/low) and test window (N days) as an immediate follow-up (new content,
  same carve-out as the goal reason).

### Step 4.1a — Reason capture for blocked/abandoned goals (P14)

For each goal marked **B** or **A** at Step 4, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.1a — Reason capture for blocked/abandoned goals (P14)"` and follow it. No B/A goals → skip.

### Step 4.1c — Verification contract scoring (P55)

If `goal_contracts` in Step 1's `close-context` output holds a contract for any completed goal, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.1c — Verification contract scoring (P55)"` and follow it. Otherwise skip silently.

### Step 4.5 — Cross-namespace learning conflicts (P2.1) — mandatory

Must run before the close payload is built:

```bash
~/.claude/skills/compass/scripts/compass.sh log-learnings-batch <namespace> \
  '[{"text": "<learning text 1>", "tags": [...], "reconcile": true},
    {"text": "<learning text 2>", "tags": [...], "reconcile": true}]'
```

Walk the returned `results` in order (matched by `text`) and keep each `learning_id`: zone and boost batches take it in place of the full text. An `ok: true` result with no `reconciliation_action` is already recorded — pass `"learnings": []` in the close payload for it. If any result carries `reconciliation_action: "pending"`, a `contradiction_candidate`, or `require_conflict_resolution: true`, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.5 — Result handling (reconciliation, contradiction, cross-namespace conflict)"` and follow it.

### Step 4.6 — Intent change reason (P1.2)

Skip if drift was handled at ORIENT. If intent changed mid-session and no reason
recorded yet, ask and call `set-intent`.

All three data types get stored: goal completion stats → trend; goal_origin →
learning-to-goal mapping; hypothesis metadata → validation surfacing.

---

## Step 5 — Consolidated close checklist (finding #2 collapse)

One screen replaces what used to be five sequential Y/N gates (2026-08-27 close-overhead audit,
finding #2): **CLAUDE.md reflection (P5.1), outcome-link offer (P49), cross-namespace
propagation, skill-feedback (P52), retrieval-stale decay review (P58)**, plus a backlog match. It
renders after Step 4 because its eligibility depends on Step 4's outputs (completed goals,
distilled learnings). The common case (all goals completed, nothing contradictory, no friction)
costs one screen.

**Compute A–F silently** (each reuses data already in hand; no full `read`):

- **A — CLAUDE.md reflection:** gate on `repo_path` configured AND `CLAUDE.md` at that root. If
  it passes, scan git commits/completed todos for module-map gaps and this close's learnings for
  `tooling`/`debugging`-tagged gotchas. **Skip entirely if the user already updated CLAUDE.md
  this session** (git diff or the note): never propose duplicates.
- **B — Outcome-link offer:** eligible if any goal was marked **C** at Step 4. **Same-namespace
  only**: never link a completed goal to a reality bullet in another namespace, even on a
  superficial match.
- **C — Cross-namespace propagation:** eligible if any distilled learning carries a broad
  platform tag (architecture, process, tooling, testing, performance, integration, debugging,
  spring, kafka, redis).
- **D — Decay review:** from Step 1's `close-context`: `decay_candidates` (P2.2, unvalidated
  hypotheses >60 days), `fact_decay_candidates` (P-GC2, unreinforced weight-1 facts) and
  `retrieval_stale_candidates` (P58, surfaced often, never reinforced). No second call.
- **E — Skill-feedback:** always eligible, but reconcile first: Step 1's
  `skill_feedback`, and `action-skill-feedback` silently any `"status": "open"` entry whose fix this
  close's reality.md text already confirms.
- **F — Backlog match:** Step 1's `backlog_match` (keys: `compass.sh schema match-backlog`). The script saves the result and `close`
  writes it into the history snapshot (`## Backlog match`), so do not copy candidates into the
  note. Eligible if `candidates` is non-empty; items this session appended are in
  `excluded_same_session`, so do not offer them. This catches what item B cannot: B sees only
  goals *declared* at DECIDE, while a backlog item can be resolved by ad-hoc work never made a goal,
  leaving it stale across sessions (real gap, 2026-09-15).

**If none of A–F have signal:** skip this whole step — no output — proceed to Step 6.

**Otherwise**, render only the items with actual signal as one screen:

```
🧾 Close checklist — <N> item(s):

  1. [CLAUDE.md]      <N> candidate update(s) detected → apply? (default: yes)
  2. [OUTCOME]         <N> goal(s) completed → link to reality bullets? (default: yes)
  3. [PROPAGATE]       "<learning, condensed>" tagged `<tag>` → push to global as-is? (default: yes)
                       — or, when the wording names modules/paths/services/skills:
                       → push to global as: "<generalised wording>"  [3o = original wording]
  4. [DECAY]           <N> decay/retrieval-stale candidate(s) → review for discard? (default: review)
  5. [FEEDBACK]        any protocol friction to log? (default: no)
  6. [BACKLOG MATCH]   "<item text, condensed>" ([tactical|strategic]) looks resolved by
                       this session's work → clear it? (default: yes)

Accept all defaults, or override by number (e.g. "1n 5:felt slow at step 3")? [Enter to accept all]
```

Number only the items rendered (with signal only on C and E, show them as 1 and 2). A
`match-backlog` response can hold several candidates: render each as its own `[BACKLOG MATCH]`
line, since each needs its own accept/decline.

**Enter / "y" / "yes"** applies every rendered item's default. A per-item override (`"1n"` flips
item 1 to no; `"5:<text>"` supplies FEEDBACK's text) applies that item and defaults the rest.

**Then run each accepted item's full existing sub-flow** (this step collapses only the gating
decision, not the mechanics):

- **CLAUDE.md → Y:** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — CLAUDE.md"` and follow it.
- **OUTCOME → Y:** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — OUTCOME"` and follow it.
- **PROPAGATE → Y:** the wording (as-is vs generalised) was decided at render time, so accept-all
  is the only confirmation, never a second "use this wording?" turn; `3o` pushes the original,
  `3n` skips. Load `~/.claude/skills/compass/scripts/compass.sh prompt cross-namespace-propagation --section "From the close checklist"`
  and `--section "With close-apply"` (as-is vs generalised routing, then the `propagate` item shapes);
  the "Deep-close prompt" section is not needed here.
- **DECAY → "review":** present the batched decay list as Step 6 describes, Y/N per candidate;
  decline-all skips `approve-learning-decay` entirely.
- **FEEDBACK → Y (or inline text):** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — FEEDBACK"` and follow it.
- **BACKLOG MATCH → Y (per candidate):** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — BACKLOG MATCH"` and follow it.

**Rule:** one screen, one round trip for the fully nominal case; drill into an item only when
the user overrides it or it needs free text.

---

## Step 6 — Auto-distil, decay, corpus health, reality, and close

**Reality rescue (no-todos guard):** if completed todos are empty AND `note` is non-empty, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Reality rescue (no-todos guard)"` and follow it. Otherwise skip.

**Auto-distil:** from completed todos, git log and any note, distil **one** durable learning
automatically; present it for approval only if the note holds something non-obvious.

**History snapshot contract:** the close payload MUST include `completed` and `incomplete` arrays
from the current todo list state. If the todo list is empty, derive `completed` from the goals
marked **C** at Step 4.

**Learning decay (P2.2 + P-GC2 + P58):** if Step 5's DECAY item was accepted (or was the only
signal and defaulted through), load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Learning decay review"` and follow it. Otherwise skip.

**Corpus health (P10):** if `close-context`'s `corpus_summary_due` is `true`, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Corpus health (P10)"` and follow it. `false` → skip silently.

### Default path: one `close-apply` call

Bundle this close's pending zone assignments, `pending_boosts`, goal outcome links, accepted
propagations (see the propagation prompt), reality ops and the close payload as `{zones, boosts,
outcomes, propagate, reality_ops, close}`; add `conflicts` and `learnings` if Step 4.5 has not
written them yet. It validates the whole bundle first, applies in order, restores the namespace on
any failure (`failed_step`; nothing half-applied) and runs reality compaction last. Shapes:
`compass.sh help close-apply` and `schema close-apply`.

- **Reality ops:** a pure add or remove is one op; collect this close's BACKLOG MATCH removals and
  reality-rescue additions into the same `reality_ops`. A `section` must be a leaf from
  `close-prepare`'s `reality_sections` (a parent header with children is refused and cascades into
  outcome errors). `close.reality` must then be `""`. `outcomes` apply
  after `reality_ops`, so they can link the bullets the same bundle appends.
- **Verify:** `reality.write_verified` lists bullets already stamped on write. Run `verify-reality
  <namespace> '<JSON array>'` for the rest of `reality.appended_hashes` that correspond to
  completed-todo-derived bullets promoted to "What exists and works".
- **Close payload:** key names in `compass.sh schema close` (`_payload`). `close` takes both prompt
  counts from the `tally-prompt` tally and warns when one looks implausibly high; pass
  `open_prompt_count`/`close_prompt_count` only as an estimate for a phase you did not tally.
- **Tally keys:** note each interactive screen Steps 2–5 show (`summary`, `batch`, `checklist`,
  `conflicts`) and pass them as the bundle's `screens` list; `close-apply` tallies them before
  `close`. Outside `close-apply`, run `compass.sh tally-prompt <namespace> close <a>,<b>`. Repeating
  a key never double counts, and Step 4's batch is **one**.

### Without `close-apply` (full-rewrite `update-reality`, or a deep close)

Reality edits always go through the script, never the file. A single add or remove →
`append-reality-bullet` / `remove-reality-bullet`; two or more → one all-or-nothing
`reality-bullets-batch`; a full rewrite → `update-reality`. For call shapes, the
move-keeps-verification rule and the `advisory` field, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Reality write primitives"`.
Flush pending zones with `set-learning-zones-batch` and `pending_boosts` with
`boost-learnings-batch` (once each, right before `close`).

`close`'s payload `reality` has exactly one no-op value, `""`; anything else overwrites
`reality.md`, even after a separate `update-reality`. If reality was already written, it **must** be
`""`. Auto-verify the response's `new_bullet_hashes` that correspond to bullets promoted to "What
exists and works" (those in `write_verified` are already stamped and left out):
```bash
~/.claude/skills/compass/scripts/compass.sh verify-reality <namespace> '<JSON array of new_bullet_hashes>'
```
Then close, then compact. An unknown payload key comes back in `warnings` as `Unknown close
payload key`:
```bash
~/.claude/skills/compass/scripts/compass.sh close <namespace> '<payload_json>'
~/.claude/skills/compass/scripts/compass.sh run-reality-compaction <namespace>
```
Run compaction **last**: `close` is the last thing that can rewrite `reality.md`, and compacting
earlier risks a later full-document write restoring bullets it just archived. It is a silent no-op
most sessions (checks its own due-ness).

### After the close call

If `compaction.archived` (close-apply) or `archived` (run-reality-compaction) is above 0, say
`📦 Archived <N> long-verified bullet(s) to reality_archive.md.`; at 0 say nothing. The response
includes `close_duration_seconds`/`close_command_count` (`close-prepare` ran at Step 1) and the
`open_prompt_count`/`close_prompt_count` with `prompt_count_sources` (tally or payload): no action.

---

## After close (still in this sub-skill, before returning)

Run, in order, exactly as the parent's sections document them (not duplicated here —
same files, same triggers):

1. **Skill patch check** — load the section with
   `~/.claude/skills/compass/scripts/compass.sh prompt router-sections --section "Skill patch check"`
   and follow it.
2. **Close-time skill harvesting** (complexity trigger) — if the complexity trigger
   fires, load the section with
   `~/.claude/skills/compass/scripts/compass.sh prompt skill-opportunity-detection --section "Close-time: skill harvesting"`
   and follow it.

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
