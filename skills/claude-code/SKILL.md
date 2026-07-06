---
name: cli-archetypes
description: Use when designing, building, reviewing, or auditing a command-line tool — a new CLI/daemon/cron job, a change adding a gating flag (--check, --fail-if, --dry-run), an exit-code or state-file decision, a subprocess/child-tool integration, or a "why does this tool feel weird" review. Classifies the tool against six CLI archetypes, detects fusion seams, and resolves the concrete controls the design owes — producing a present/absent/floor-dropped checklist, not an opinion.
---

# CLI Archetypes — classify before you code

This skill routes to a corpus of five files (vendored beside this skill or
at the repo root — adjust paths to where you placed them):

- `archetype-stage-dissection.txt` — the six archetypes on a 12-stage pipeline
- `archetype-combinations.txt` — fusions and pipe compositions, with caveats
- `archetype-fusion-seams.txt` — the four seams, smells, and check procedure
- `server-archetype-control-map.txt` — per-archetype control cards (defaults/conditionals/forbidden)
- `server-controls.txt` — the control catalog (obligation, floor, proof per C-id)

## Route by task

**Designing a new tool** → read the dissection first. Answer: which of the
six archetypes is this? Then read that archetype's card in the map — its
defaults are your obligations, its OVERKILL/DROP marks are what NOT to
build, its forbidden list is the misreach warning. Remember: "present is
not a module" — the 12 stages are a thinking checklist, not a directory
layout.

**Reviewing/auditing an existing tool** → read the seams file's DETECTION
FIRST smell list; grep for the structural smells. If ≥2 archetypes are
present, run HOW TO CHECK A FUSION: classify both layers → attach card
defaults → fire the seam's forced triggers → floor_drop against the
catalog → check structural controls by grep now, behavioral by test.

**A change adds a gating flag** (`--check`, `--fail-if`, exit-on-condition)
→ smell #4: the tool just grew a checker layer. Its exit codes must be
designed (distinct code per outcome class, documented) — never inherited
from the underlying call's success/failure.

**Touching a state file** → STATE seam: temp+fsync+rename (C33), one lock
owner, effect-then-record ordering. Kill -9 mid-write must leave the old
state intact.

**Spawning a child tool** → SPAWN seam: timeout (C06), child output is a
trust boundary (C01 — prefer the child's --format json over parsing prose),
child behind a port so dry-run can be dry (C17), host logs never pollute
relayed stdout (C12).

## Hard rules

1. **Classify before writing code.** Wrong classification = wrong control
   set; it is the load-bearing step.
2. **Emit a checklist, not a vibe**: every control present / absent /
   floor-dropped, each with its evidence (grep, file:line, or test name).
   Absent + no floor = a real gap to close or justify on the record.
3. **Exit codes are a designed contract.** "Thing is unhealthy" and
   "couldn't check" never share a code. One exit owner per binary — and
   verify the owner actually maps distinct outcomes to distinct codes
   (single ownership is necessary, not sufficient).
4. **Floors are legitimate.** Dropping a control whose floor holds is
   correct engineering, not a violation — but record which floor.
