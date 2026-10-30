# compass-close

Fast-close sub-skill for `/compass`, invoked from its RALF half when `path=fast`. Deep close and
multi-namespace close run from the router's `scripts/prompts/router-sections.md`; do not route
them here. Step 4 is one screen (boosts, goal statuses, per-learning lines). Step 5's checklist is
a separate screen because its eligibility depends on Step 4's outputs.

**Input:** `namespace`, plus the todo list state (with no TodoWrite-equivalent tool, use
`state.json`'s `goal_progress`).

**Never call `compass.py close` directly as a shortcut.** Run Steps 1–6 in order: `close` /
`close-apply` is the last action of Step 6, and skipping steps silently loses CLAUDE.md hygiene,
code context, batched zone/decay decisions and learnings.

---

## Step 1 — Mark close start + narrow snapshot (no pause)

Run silently, one call, with the draft session context (note + completed items joined):
```bash
~/.claude/skills/compass/scripts/compass.sh close-prepare <namespace> '{"context": "<note + completed items>"}'
```

It stamps the close-phase start (so `close` reports duration and command count) and returns
`close_context`, `gitlog`, `skill_feedback`, `backlog_match`, `boost_candidates`,
`skipped_gates`, the After-close gates and `brief_markdown` (keys: `~/.claude/skills/compass/scripts/compass.sh schema
close-prepare`). A failed part is listed under `errors` with a null key: run that command by hand.
If the final note differs a lot from the draft, re-run `suggest-boosts` / `match-backlog` with it.

Hold `close_context` (`planned_actions`, `reality_validation`, `session_appended_bullets`,
`goal_contracts`, the three decay lists, `corpus_summary_due`) for Steps 4.1c, 5 and 6.

Read the todo state from the host's TodoWrite-equivalent tool if it has one; otherwise
cross-reference `planned_actions` against `goal_progress` (which `set-goal-status` updates).

---

## Step 2 — Compact summary + single question

Show Step 1's `brief_markdown` lines as they are, under the header. If a host todo tool
disagrees with `goal_progress`, the todo tool wins.

```
## 🔄 Compass — <namespace> · fast close

<brief_markdown lines: ✅ Done, ⏸ Skipped, 🔀 Git, ⏭ Gates (only when a gate was skipped), …>

Anything to add? (Enter to skip)
```

Wait for one response. Capture as `note`. Empty / "skip" / enter = `note = ""`.

If `note` is empty AND no todo is completed, ask once: "Nothing recorded this session — confirm close? [Y/n]".

**One screen with Step 4 (default):** when the session is open, no goal is `(needs input)` and
it is not the "Nothing recorded" case, compute Step 4.0 and show this table and Step 4's batch together. Replace the "Anything to add?" line with the batch's
last line, extended: `or "note: <text>" to add a note (Enter accepts all defaults, empty note)`.
Overrides come first, the note after `note:`. Tally one `batch` screen, no `summary`. Keep the two
screens (this one, then Step 4) otherwise.

---

## Step 2.1 — Artefact capture offer

Trigger if **either**: A) the note or completed todos contain "diagram", "chart", "widget", "SVG", "visual", "dashboard" or "panel"; B) (`compass-dashboard` namespace only) `~/Downloads/compass-dashboard.html` was modified after `last_open`. Neither → skip, no prompt. Otherwise load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 2.1 — Artefact capture offer (P41)"` and follow it.

---

## Step 3 — Code context update

**When git commits exist this session** (Step 1's `gitlog`), write this block without prompting:

```
## Last updated: <YYYY-MM-DD>

**Last touched files:** <files changed in commits this session, comma-separated>
**In-progress approach:** <first incomplete todo, or continuation inferred from note>
**Next micro-step:** <first incomplete todo item — or "none" if all done>
**Tried and discarded:** <only if the note mentions a pivot, blocker, or abandoned approach>
```

Write every `code_context.md` with the script, never the Write tool (it fails on an existing
file that was not Read first). Pipe the content on stdin, or pass `--file <path>`:
```bash
~/.claude/skills/compass/scripts/compass.sh write-code-context <namespace> <<'EOF'
<block>
EOF
```

**No commits this session:** if `code_context.md` exists, ask once:
```
💻 Code context — update for next session? (active files, decisions in flight, next entry point — or skip)
```
Write the reply under a `Last updated: <date>` header; empty / "skip" leaves it unchanged. If it
does not exist and the note describes substantive work, offer:
```
💻 No code context file exists — create one for next session? (or skip)
```

---

## Step 4 — Consolidated close batch

If the session is open (`planned_actions`), build one screen (shared with Step 2's table by
default) covering boosts, goal statuses and one line per distilled learning. None depends on
another.

**4.0 — Compute every default silently first:**

- **Boost candidates:** Step 1's `boost_candidates`; no `candidates` → no BOOST lines. Per
  candidate, judge **boost** (the session directly reinforced or depended on it) vs **skip**
  (incidental keyword overlap), with a one-clause rationale.
- **Goal completion defaults:** per planned goal, check the todo state (or
  `goal_progress[index]`) and scan reality.md's latest 'What exists and works' additions plus
  this session's commits for shipping evidence. Marked done, or evidence found → default
  **Completed**, citing it. Otherwise no confident default: mark it `(needs input)`, excluded from
  bulk accept.
- **Per-learning line:** for each learning distilled this session, one line with goal-origin
  (batch-inferred), suggested tags (`suggest-tags`) and type + zone. Take the zone from
  `suggest-tags`' `inferred_zone` (pass `learning_type` in its payload when the type is not
  `fact`), since `log-learning` runs only after this screen. `null` renders as zone `none`; never
  render `skip` (`set-learning-zones-batch` rejects it).

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
status letter or full status word for GOAL (`C`→`completed`, `S`→`skipped`, `A`→`abandoned`,
`B`→`blocked`); for LEARNING, comma-separated `zone=<x>` / `type=<x>` / `tags=+<tag>` /
`tags=-<tag>` / `goal=<n|none>` on the same item (e.g. `6:zone=warning,type=procedural`).

**Mismatch check:** if a GOAL override contradicts its evidence-based default (Completed on
shipping evidence, override Abandoned/Blocked), ask once per item before applying: `Reality/git shows "<goal text>" shipped —
still mark <status>? [Y/n]`.

**Route each family's response:**

- **BOOST** → add each confirmed BOOST to a `pending_boosts` list held for the whole close (the
  DECAY review and a prior learning this session reconfirmed may add more); never call `boost-learning` directly. Use the `learning_id`, not the
  text; a legacy candidate (`learning_id: null`) goes in `texts` as its `full_text`. Flush once as
  `close-apply`'s `boosts` key, which also records them as applied (P64). Skip it if the list
  is empty.

- **GOAL** → build the `goal_statuses` array (e.g. `["completed", "blocked", ...]`). For each goal
  marked **B**/**A**, ask for a reason at Step 4.1a. Omit `goal_types`: close uses the ones `open`
  stored. Pass it only if the user re-tagged a goal mid-session.

- **LEARNING** → apply goal_origin / tags / type / zone from the (possibly overridden) line. **Do
  not build the close payload until every learning has a `goal_origin`.** Add each zone as
  `{"learning_id": ..., "zone": ...}` to a `pending_zone_assignments` list held for the whole close;
  flush once as `close-apply`'s `zones` key (or `set-learning-zones-batch` right before `close`).
  Skip it if the list is empty. For **hypothesis** learnings also ask
  confidence (high/medium/low) and test window (N days) as an immediate follow-up.

### Step 4.1a — Reason capture for blocked/abandoned goals

For each goal marked **B** or **A** at Step 4, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.1a — Reason capture for blocked/abandoned goals (P14)"` and follow it. No B/A goals → skip.

### Step 4.1c — Verification contract scoring

If `goal_contracts` holds a contract for any completed goal, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.1c — Verification contract scoring (P55)"` and follow it. Otherwise skip silently.

### Step 4.5 — Cross-namespace learning conflicts — mandatory

Must run before the close payload is built:

```bash
~/.claude/skills/compass/scripts/compass.sh log-learnings-batch <namespace> \
  '[{"text": "<learning text 1>", "tags": [...], "reconcile": true},
    {"text": "<learning text 2>", "tags": [...], "reconcile": true}]'
```

Keep each result's `learning_id` (matched by `text`) for the zone and boost batches. An `ok: true` result with no `reconciliation_action` (or `"no_match"`) is already recorded — pass `"learnings": []` in the close payload for it. If any result carries `reconciliation_action: "pending"`, a `contradiction_candidate`, or `require_conflict_resolution: true`, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 4.5 — Result handling (reconciliation, contradiction, cross-namespace conflict)"` and follow it.

### Step 4.6 — Intent change reason

Skip if drift was handled at ORIENT. If intent changed mid-session and no reason
recorded yet, ask and call `set-intent`.

---

## Step 5 — Consolidated close checklist

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
- **D — Decay review:** `close_context`'s `decay_candidates` (unvalidated hypotheses >60 days),
  `fact_decay_candidates` (unreinforced weight-1 facts) and `retrieval_stale_candidates`
  (surfaced often, never reinforced). No second call.
- **E — Skill-feedback:** always eligible. First `action-skill-feedback` silently any open
  `skill_feedback` entry whose fix this close's reality.md text already confirms.
- **F — Backlog match:** Step 1's `backlog_match`, eligible if `candidates` is non-empty (it
  catches items resolved by work that was never a goal). Never offer `excluded_same_session`
  items, and do not copy candidates into the note (`close` records them).

**If none of A–F have signal:** skip this whole step — no output — proceed to Step 6.

**Otherwise**, render only the items with actual signal as one screen:

```
🧾 Close checklist — <N> item(s):

  1. [CLAUDE.md]      <N> candidate update(s) detected → apply? (default: yes)
  2. [OUTCOME]         <N> goal(s) completed → link to reality bullets? (default: yes)
  3. [PROPAGATE]       "<learning, full text>" tagged `<tag>` → push to global as-is? (default: yes)
                       — or, when the wording names modules/paths/services/skills, or is too
                       long to show whole:
                       → push to global as: "<generalised or condensed wording>"  [3o = original wording]
  4. [DECAY]           <N> decay/retrieval-stale candidate(s) → review for discard? (default: review)
  5. [FEEDBACK]        any protocol friction to log? (default: no)
  6. [BACKLOG MATCH]   "<item text, condensed>" ([tactical|strategic]) looks resolved by
                       this session's work → clear it? (default: yes)

Accept all defaults, or override by number (e.g. "1n 5:felt slow at step 3")? [Enter to accept all]
```

Number only the items rendered (signal only on C and E → 1 and 2). Render each
`match-backlog` candidate as its own `[BACKLOG MATCH]` line.

**Enter / "y" / "yes"** applies every rendered item's default. A per-item override (`"1n"` flips
item 1 to no; `"5:<text>"` supplies FEEDBACK's text) applies that item and defaults the rest.

**Then run each accepted item's sub-flow:**

- **CLAUDE.md → Y:** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — CLAUDE.md"` and follow it.
- **OUTCOME → Y:** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — OUTCOME"` and follow it.
- **PROPAGATE → Y:** the wording was decided at render time, so accept-all is the only
  confirmation, never a second "use this wording?" turn; `3o` pushes the original, `3n` skips.
  Load `~/.claude/skills/compass/scripts/compass.sh prompt cross-namespace-propagation --section "From the close checklist" --section "With close-apply"`
  in one call; the "Deep-close prompt" section is not needed here.
- **DECAY → "review":** present the batched decay list as Step 6 describes, Y/N per candidate;
  decline-all skips `approve-learning-decay` entirely.
- **FEEDBACK → Y (or inline text):** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — FEEDBACK"` and follow it.
- **BACKLOG MATCH → Y (per candidate):** load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 5 item — BACKLOG MATCH"` and follow it.

Drill into an item only when the user overrides it or it needs free text. Batch Y items'
sections from one file into one call.

---

## Step 6 — Auto-distil, decay, corpus health, reality, and close

**Reality rescue (no-todos guard):** if completed todos are empty AND `note` is non-empty, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Reality rescue (no-todos guard)"` and follow it. Otherwise skip.

**Auto-distil:** from completed todos, git log and the note, distil **one** durable learning;
ask for approval only if the note holds something non-obvious.

**History snapshot contract:** the close payload MUST include `completed` and `incomplete` arrays
from the current todo list state. If the todo list is empty, derive `completed` from the goals
marked **C** at Step 4.

**Learning decay:** if Step 5's DECAY item was accepted (or was the only signal and defaulted
through), load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Learning decay review"` and follow it. Otherwise skip.

**Corpus health:** if `corpus_summary_due` is `true`, load `~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Corpus health (P10)"` and follow it. `false` → skip silently.

### Default path: one `close-apply` call

Bundle the pending zone assignments, `pending_boosts`, goal outcome links, accepted propagations,
reality ops and the close payload as `{zones, boosts, outcomes, propagate, reality_ops, close}`;
add `conflicts` and `learnings` if Step 4.5 has not written them yet, and Step 6's `decay` /
`distil` when they were accepted. It validates first, rolls
back on any failure (`failed_step`) and runs reality compaction last. Shapes: `~/.claude/skills/compass/scripts/compass.sh help close-apply` and
`schema close-apply`.

- **Reality ops:** one op per add or remove, including BACKLOG MATCH removals and reality-rescue
  additions. A `section` must be a leaf from `reality_sections` (a parent with children is
  refused). `close.reality` must then be `""`. `outcomes` apply after `reality_ops`, so they can
  link bullets the same bundle appends.
- **Verify:** run `verify-reality <namespace> '<JSON array>'` on the `unverified_appended` hashes
  (What exists appends not stamped on write or by an outcome link) that came from completed todos.
- **Close payload:** keys in `~/.claude/skills/compass/scripts/compass.sh schema close`. Prompt counts come from the tally; pass
  `open_prompt_count`/`close_prompt_count` only for a phase you did not tally.
- **Tally keys:** pass each screen Steps 2–5 showed as the bundle's `screens` list (`summary`,
  `artefact`, `batch`, `reason`, `contract`, `conflicts`, `intent`, `checklist`, `decay`, `corpus`; others are
  refused), or `[]` when none waited for a response. Step 4's batch is **one**. Outside
  `close-apply`, run `~/.claude/skills/compass/scripts/compass.sh tally-prompt <namespace> close <a>,<b>` (or `none`).

### Without `close-apply` (full-rewrite `update-reality`, or a deep close)

Load
`~/.claude/skills/compass/scripts/compass.sh prompt close-conditional-flows --section "Step 6 — Close without close-apply"`
and follow it: script-only reality writes, the flushes, `reality: ""`, verify, `close`, then
compaction last.

### After the close call

If `compaction.archived` (close-apply) or `archived` (run-reality-compaction) is above 0, say
`📦 Archived <N> long-verified bullet(s) to reality_archive.md.`; at 0 say nothing.

---

## After close (still in this sub-skill, before returning)

Run in order. Both read Step 1's `close-prepare` fields; a false or empty field means skip and
load nothing.

1. **Skill patch check** — only if at least one learning was distilled and
   `skill_patch_matches` is non-empty (installed skills named in the session text, minus
   skills edited this session). Load the section with
   `~/.claude/skills/compass/scripts/compass.sh prompt router-sections --section "Skill patch check"`
   and follow it for those skills only.
2. **Close-time skill harvesting** — only if `harvest_due` is true (`harvest_signals` says
   which two of the three signals held). Load the section with
   `~/.claude/skills/compass/scripts/compass.sh prompt skill-opportunity-detection --section "Close-time: skill harvesting"`
   and follow it from Step 2 (Step 1's measurement is done).

Cross-namespace propagation already ran in Step 5; do not run it again here.

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
