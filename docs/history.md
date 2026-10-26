# compass-close — moved history

Rationale and provenance text moved out of SKILL.md. Nothing here is a step the close runs.

## 2026-10-26 — Phase 2 provenance strip (agentskills alignment plan)

SKILL.md went from 25,146 to 19,989 bytes (plan G1: ≤ 20KB). Removed, with no step, number,
default or order change:
- Header: "(P25; `docs/compass-close-sop.md`)" and the "Consolidation (v1, 2026-09-01,
  `docs/2026-08-31-consolidate-open-close-prompts-plan.md`)" paragraph.
- Heading and label tags: (P41), (v1 consolidation, 2026-09-01), (R5), (P0.3), (P0.2), (P4.1),
  (P6), (P1.1), (P56), (P-GC4), (P14), (P55), (P2.1), (P1.2), (finding #2 collapse), (P2.2),
  (P-GC2), (P58), (P52), (P10), (P4).
- Step 1: "`close-prepare` replaces `mark-close-start`, `close-context`, `gitlog`,
  `get-skill-feedback`, `match-backlog` and `suggest-boosts`"; the "a real session found no
  TodoWrite tool at all, 2026-09-02" note.
- Step 4.6: "All three data types get stored: goal completion stats → trend; goal_origin →
  learning-to-goal mapping; hypothesis metadata → validation surfacing."
- Step 5: "One screen replaces what used to be five sequential Y/N gates (2026-08-27
  close-overhead audit, finding #2)"; item F's "(real gap, 2026-09-15)".
- Step 4 GOAL parsing: "the old `E` had no meaning in the script".
- After the close call: the response fields that need no action (`close_duration_seconds`,
  `close_command_count`, prompt counts, `prompt_count_sources`).
The full script path is now written once. `--section` load strings keep their tags.

**Correction (same day, after code review):** the strip first replaced the full script path with
a bare `compass.sh` shorthand, which is not on PATH. Full paths are restored and more text was
reworded to stay ≤ 20KB. `scripts/tests/test_subskill_protocol.py` in `compass` now guards the
full path, every `--section` load, the step headings and the size budget.
The first strip also dropped two instructions, now restored: a prior learning reconfirmed this
session can add to `pending_boosts`, and the mismatch check fires on any override that
contradicts the evidence-based default.
