# compass-close

Fast-close sub-skill for the compass loop (P25 Phase 1 — `docs/compass-close-sop.md`
in the `compass` skill).

Invoked from `/compass`'s RALF half when `path=fast`. **Phase 1 scope: fast close
only** — deep close and multi-namespace close still run inline in the parent
`~/.claude/skills/compass/SKILL.md` (its own "Deep close" / "Multi-namespace close"
sections, now loaded from `scripts/prompts/router-sections.md`) until Phases 2–3 of the design doc extract them too. Do not route those
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
- current todo list state (inherited session context — no marshalling needed; falls
  back to `state.json`'s `goal_progress` when this runtime has no TodoWrite-equivalent
  tool, see Step 1)

**Never call `compass.py close` directly as a shortcut.** Run Steps 1–6 in order.
The `close` call is the final action of Step 6, not a replacement for the steps
before it — skipping steps silently loses CLAUDE.md hygiene, code context, batched
zone/decay decisions, and learnings.

---

## Step 1 — Mark close start + narrow snapshot (no pause)

Run silently, in this order:
```bash
~/.claude/skills/compass/scripts/compass.sh mark-close-start <namespace>
~/.claude/skills/compass/scripts/compass.sh close-context <namespace>
~/.claude/skills/compass/scripts/compass.sh gitlog <namespace>
```

`mark-close-start` stamps `close_phase_started_at`/resets `close_phase_command_count` —
every namespace-bearing call from here to the final `close` tallies into that counter,
so the close response reports its own `close_duration_seconds`/`close_command_count`
(2026-08-27 close-overhead audit finding #6) without a hand transcript-tally.

`close-context` replaces the old `read` call (for exact key names, run
`~/.claude/skills/compass/scripts/compass.sh schema close-context`) — it returns `planned_actions`,
`reality_validation` (hashes), `session_appended_bullets` (`{hash, text, section}` for
bullets appended this session), `goal_contracts`, `decay_candidates` (P2.2),
`fact_decay_candidates` (P-GC2), `retrieval_stale_candidates` (P58), and
`corpus_summary_due`: the slices this close path actually needs, instead of the full
orient context (finding #1). Hold all of these — they feed Step 4.1c (`goal_contracts`),
Step 5 item B (`reality_validation`), Step 5 item D / Step 6 (the three decay lists,
`corpus_summary_due`).

Read the current todo list state — this runtime's TodoWrite-equivalent tool if it has
one, otherwise fall back to `close-context`'s `planned_actions` cross-referenced
against `read`'s `goal_progress` array (2026-09-02 — a real session found `ToolSearch`
returning nothing for TodoWrite at all; `goal_progress` is `open`'s host-independent
substitute, updated via `set-goal-status` as goals complete mid-session, see
compass-open's Step 4).

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

Run after the compact summary, before code context. Trigger if **either**: A) the note or completed todos contain "diagram", "chart", "widget", "SVG", "visual", "dashboard" or "panel"; B) (`compass-dashboard` namespace only) `~/Downloads/compass-dashboard.html` was modified after `last_open`. Neither → skip, no prompt. Otherwise load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 2.1 — Artefact capture offer (P41)"` and follow it.

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
- **Goal completion defaults** (P0.3 + P-GC4) — for each planned goal, check the todo
  list state (or `goal_progress[index]` on the fallback path — see Step 1) and scan
  reality.md's most recent 'What exists and works' additions plus this session's git
  commits for shipping evidence (same cross-check the old flow ran *before* asking).
  If the todo item / `goal_progress` entry is marked done, or shipping evidence exists
  → default: **Completed**, citing the evidence. Otherwise there is no confident
  default — mark that item `(needs input)`; it is excluded from the bulk-accept
  shortcut and always needs an explicit letter, even when every other item is
  accepted via Enter.
- **Per-learning line** (P0.2 + P4.1 + P6/P1.1/P56) — for each learning distilled this
  session, combine three already-existing inference passes into one line instead of three
  separate ones: goal-origin (batch-inferred, P52, unchanged), suggested tags
  (`suggest-tags`), and type + zone (use `inferred_zone` from the same `suggest-tags`
  response — pass `learning_type` in its payload when the type is not `fact` — because
  `log-learning` runs only after this screen; `null` renders as zone `none`; never render
  `skip` — `set-learning-zones-batch` rejects it).

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
  ~/.claude/skills/compass/scripts/compass.sh boost-learnings-batch <namespace> \
    '{"learning_ids": ["<learning_id 1>", ...], "texts": ["<exact text, for entries with no id>"]}'
  ```
  Use the `learning_id` that `log-learnings-batch` returned for this session's learnings;
  a prior-session learning from `surface-learnings` or a `suggest-boosts` candidate also
  carries one. Do not re-send the text. A `suggest-boosts` candidate with `learning_id: null`
  is a legacy entry: put its `full_text` (the `text` field is cut at 80 characters) in `texts`.
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
  ~/.claude/skills/compass/scripts/compass.sh set-learning-zones-batch <namespace> \
    '[{"learning_id": "<id 1>", "zone": "golden"}, {"learning_id": "<id 2>", "zone": "warning"}]'
  ```
  This is finding #3 from the 2026-08-27 close-overhead audit — a shell loop over
  individual `set-learning-zone` calls still forks one subprocess per call even when
  batched into fewer *tool* calls; one script invocation for the whole close does not.
  Skip the call entirely if `pending_zone_assignments` ends up empty. For **hypothesis**
  type learnings, additionally ask confidence (high/medium/low) and test window (N days)
  as an immediate follow-up — new content, same carve-out as the goal reason capture.

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
- **F — Backlog match:** call `match-backlog <namespace> '{"context": "<note +
  completed-items text>"}'`. Key names: `~/.claude/skills/compass/scripts/compass.sh schema match-backlog`. The script
  saves the result and `close` writes it into the history snapshot (`## Backlog match`),
  so don't copy candidates into the note. Eligible if `candidates` is non-empty (items this session
  appended are already left out, in `excluded_same_session` — don't offer those). This catches a
  gap outcome-linking (item B) can't: B only checks goals *declared* at DECIDE, but a
  Tactical/Strategic backlog item can also be resolved by ad-hoc work the user asked
  for mid-session — never a formal goal — which B has no way to see. Without this
  check, a resolved item (e.g. "blocked on X" after X clears) can sit stale across
  sessions with nothing to surface it (real gap found 2026-09-15).

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

Number only the items actually rendered (skip letters with no signal — e.g. if only
C and E have signal, show them as 1 and 2). A `match-backlog` response can return more
than one candidate — render each as its own numbered `[BACKLOG MATCH]` line rather than
collapsing them, since each needs its own accept/decline.

Map the response:
- **Enter / "y" / "yes"** → apply every rendered item's default action.
- **Per-item override** (`"1n"` flips item 1's action to no; `"5:<text>"` supplies
  FEEDBACK's text directly) → apply the override, defaults for the rest.

**Then run each accepted item's full existing sub-flow — this step only collapses the
gating decision, not the underlying mechanics or script calls:**

- **CLAUDE.md → Y:** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — CLAUDE.md"` and follow it.

- **OUTCOME → Y:** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — OUTCOME"` and follow it.

- **PROPAGATE → Y:** decide the as-is vs generalised wording **at render time** (above), so
  the checklist's accept-all is the only confirmation — never a second "use this wording?"
  turn. `3o` pushes the original wording; `3n` skips. Then load
  `~/.claude/skills/compass/scripts/compass.sh prompt cross-namespace-propagation --section "Cross-namespace learning propagation"`
  and follow it (its "From the close checklist" block routes as-is vs generalised and covers
  the `log-learning global` call; no sub-prompt).
- **DECAY → "review":** present the batched decay list exactly as Step 6 below
  describes, collecting Y/N per candidate. `N`/decline-all → skip the
  `approve-learning-decay` call entirely.
- **FEEDBACK → Y (or inline text):** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — FEEDBACK"` and follow it.

- **BACKLOG MATCH → Y (per candidate):** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — BACKLOG MATCH"` and follow it.

**Rule:** one screen, one round trip for the fully-nominal case. Per-item drill-in
only for items the user overrides or that inherently need free text.

---

## Step 6 — Auto-distil, decay, corpus health, reality, and close

**Reality rescue (no-todos guard):** if completed todos are empty AND `note` is non-empty, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Reality rescue (no-todos guard)"` and follow it. Otherwise skip.

**Auto-distil:** from completed todos, git log, and any user note, distil **one**
durable learning automatically — do not present it for approval unless the note
contains something non-obvious that warrants a check.

**History snapshot contract:** the close payload MUST include `completed` and
`incomplete` arrays from the current todo list state. If the todo list is empty,
derive `completed` from goals marked **C** at Step 4 instead.

**Learning decay (P2.2 + P-GC2 + P58):** if Step 5's DECAY item was accepted (or DECAY was the only signal and defaulted through), load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Learning decay review"` and follow it. Otherwise skip.

**Corpus health (P10):** if Step 1's `close-context` `corpus_summary_due` is `true`, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Corpus health (P10)"` and follow it. `false` → skip silently.

**Default path: one `close-apply` call.** Bundle this close's pending zone assignments, `pending_boosts`, goal outcome links, approved `log-learning global` payloads, reality ops and the close payload as `{zones, boosts, outcomes, propagate, reality_ops, close}` (add `conflicts` and `learnings` if Step 4.5 has not written them yet). It validates the whole bundle first, applies in order, restores the namespace on any failure (`failed_step`, nothing half-applied) and runs reality compaction last. Shapes: `compass.py help close-apply` and `schema close-apply`. When reality_ops is used, `close.reality` must be `""`. Use the separate commands below only for a full-rewrite `update-reality` or a deep close.

**Reality and close:**

`cmd_close`'s payload `reality` field has exactly one no-op case: an empty string.
Any non-empty value unconditionally overwrites `reality.md`, even if `update-reality`
was already called separately this close. If reality was already written via a prior
`update-reality` call, the `close` payload's `reality` field **must** be `""`.

Reality edits go through the script, never the file. A pure add or remove → `append-reality-bullet` / `remove-reality-bullet`; two or more edits → one all-or-nothing `reality-bullets-batch`; a full rewrite (rewording or restructuring) → `update-reality`. For the call shapes, the move-keeps-verification rule and the `advisory` field, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Reality write primitives"`. Collect this close's BACKLOG MATCH removals and reality-rescue additions into the same batch.

Auto-verify the subset of the response's `new_bullet_hashes` that correspond to
bullets promoted to "What exists and works" this session. Bullets already stamped by
verify-on-write (the response's `write_verified`) are left out of `new_bullet_hashes`:
```bash
~/.claude/skills/compass/scripts/compass.sh verify-reality <namespace> \
  '<JSON array of new_bullet_hashes for completed-todo-derived bullets>'
```

**Prompt-count tally:** each time Steps 2–5 show the user an interactive screen, run
`compass.py tally-prompt <namespace> close <screen>` with a short key (`summary`, `batch`,
`checklist`, `conflicts`): repeating a key never double counts, and Step 4's batch is
**one**, not one per family. `close` takes both counts from the script's tally, so the
payload needs no prompt counts, and warns when a tally looks implausibly high. Pass
`open_prompt_count`/`close_prompt_count` only as an estimate for a phase you didn't
tally; a tally always wins.

Without `close-apply`, close directly:
```bash
~/.claude/skills/compass/scripts/compass.sh close <namespace> '<payload_json>'
```
`<payload_json>` holds the fields already documented above. For exact key names run
`compass.py schema close` (its `_payload` block) rather than grepping the script; a key the
script does not read comes back as an `Unknown close payload key` entry in `warnings`.

**Reality compaction (archiving cadence, docs/reality-compaction-cadence-design.md in
the `compass` skill):** `close-apply` already ran it; otherwise always call this — it checks its own due-ness internally and is
a silent no-op most sessions. Run this LAST, after `close` itself, since `close` is the
last thing that can rewrite `reality.md` (the payload's `reality` field, `update-reality`,
`verify-reality` all run earlier in this step) — running compaction any earlier risks a
later full-document reality write restoring bullets compaction just archived:
```bash
~/.claude/skills/compass/scripts/compass.sh run-reality-compaction <namespace>
```
If `archived > 0`, surface one line: `📦 Archived <N> long-verified bullet(s) to
reality_archive.md.` If `archived: 0`, say nothing — this is not a cadence worth a
prompt or an explanation when it doesn't fire.

The response includes `close_duration_seconds`/`close_command_count` when
`mark-close-start` ran at Step 1 (it always does, in this sub-skill) — no action
needed, this is what a future overhead audit reads instead of hand-tallying. It also
echoes `open_prompt_count`/`close_prompt_count`, with `prompt_count_sources` saying
whether each came from the tally or the payload.

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
