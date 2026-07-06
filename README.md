# CLI Archetypes, Stages & Fusion Seams

Your deploy tool exits `1` when the deploy failed. It also exits `1` when it
couldn't reach the API. Your CI pipeline cannot tell "production is drifted"
from "the checker is broken" — and nobody notices until the wrong one pages
someone at 3am.

That bug has a name here (**smell #1, EXIT-CODE OVERLOAD**), a diagnosis (the
**VERDICT seam**), a control it violates (**C05**), a grep that finds it, and
a test shape that keeps it fixed. So do eight more of the most common ways
CLI tools go wrong in production.

**Status: DRAFT** — published as-is from a working corpus, falsification
notes included. When a check over- or under-reported on a real tool
([modd](https://github.com/cortesi/modd), zoxide), the blind spot was
recorded in the file — those annotations are the point, not a defect.

## Try it in 60 seconds — audit any CLI you own

Structural smells are greppable. From your tool's repo:

```bash
# smell #1 — exit-code overload: how many places set the exit, and can
# "thing is broken" and "couldn't check" collide on one code?
rg -n 'process\.exit|os\.Exit|sys\.exit\(|exit [0-9]' src/

# C33 — state files written in place (kill -9 mid-write = torn state)
rg -l 'writeFile|WriteFile|open\(.*[\"'"'"']w' src/ | xargs rg -Ln 'rename'

# C12 — logs polluting stdout (breaks `yourtool | jq`)
rg -n 'console\.log|fmt\.Print[^E]|^\s*print\(' src/

# smell #3 — parsing your own child's human stdout instead of --format json
rg -n 'spawn|exec|subprocess' -l src/ | xargs rg -n 'split|regex|match.*stdout'

# smell #6 — a "daemon" that's actually a while-true loop
rg -n 'while true|while True|sleep [0-9]' src/
```

Every hit is a lead: the seams file tells you which seam it belongs to,
which control it violates, and what the fix must guarantee. A real audit
using exactly this procedure found two HIGH defects (smell #1 and the C33
torn-write) in a green-suite, TDD-built gating tool in under ten minutes —
`examples/worked-audit.md` walks the full shape.

## What's inside

| File | What it is |
|------|-----------|
| `archetype-stage-dissection.txt` | Six CLI archetypes (transformer, status checker, wrapper, orchestrator, stateful CLI, daemon) mapped onto a 12-stage pipeline, with per-stage markers: load-bearing / OVERKILL / DROP / UNDER. Includes the "present is not a module" rule. |
| `archetype-combinations.txt` | How archetypes combine: single-tool fusions (terraform, kubectl, alembic, node_exporter, k9s, make) and multi-tool pipe compositions, each with seam placement and the production caveats. |
| `archetype-fusion-seams.txt` | The detection layer: every CLI fusion is a stack of FOUR seams (VERDICT, STATE, SPAWN, ROLLUP). Smell list → seam → the controls the seam forces → how to check them. |
| `server-archetype-control-map.txt` | The control map (v0.9): per-archetype cards — web AND CLI — with default / conditional / forbidden control attachments and named triggers. |
| `server-controls.txt` | The control catalog (v0.9): every C-id with its obligation, its floor (when it may legitimately be dropped), and its behavioral proof — the engine the seams file resolves against. |
| `examples/worked-audit.md` | A complete audit walkthrough: classify → grep → checklist → verdict. |
| `skills/` | Drop-in agent instructions for Claude Code and Codex — see below. |

## Use cases

1. **Audit an existing tool** (10–30 min). Run the smell greps, classify the
   tool's layers, walk `HOW TO CHECK A FUSION` in the seams file. Output: a
   present / absent / floor-dropped checklist — findings, not vibes.
2. **Design review before writing a new tool.** Answer "which archetype am
   I?" against the dissection; the map hands you your default control set
   and the archetype's signature trap (a wrapper's domain layer should be
   *empty*; a daemon's failure mode is *under*-building). Ten minutes here
   beats a rewrite.
3. **Context for a coding agent.** Drop the skill file in (below) and your
   agent classifies before it codes, attaches the controls, and stops
   inventing exit-code semantics ad hoc. This is the corpus's primary use in
   its home environment.
4. **PR review checklist for CLI changes.** The "one more flag that gates"
   smell (#4) is a review-time catch: the moment `--fail-if` appears, the
   tool grew a checker layer and its exit codes need *designing*, not
   inheriting from the API call's success.
5. **Vocabulary for design arguments.** "That's a VERDICT seam — the inner
   checker surrenders its exit code to the outer layer" ends a debate that
   "the error handling feels weird" only starts.
6. **Post-incident analysis.** A pipe that swallowed a failing check
   (pipefail, B1), a cron pair acting on stale state (B4), a stateful tool
   that corrupted its file on kill (C33) — the combinations file names the
   mechanism and the missing seam.

## Using it with a coding agent

**Claude Code:** copy `skills/claude-code/SKILL.md` to
`.claude/skills/cli-archetypes/SKILL.md` in your project (adjust the corpus
paths inside if you vendor the files elsewhere).

**Codex / any AGENTS.md-convention agent:** paste the block from
`skills/codex/AGENTS.md` into your repo's `AGENTS.md`.

Both do the same thing: force *classification before code*, route the agent
to the smell list on review and the stage dissection on design, and make it
emit a checklist instead of an opinion.

## The method, in one paragraph

Classify the tool's layers against the six archetypes → each classification
attaches its card's default controls from the map, for free → the fusion
fires a few named conditional triggers a single-archetype tool would skip →
floors prune what's legitimately not needed → structural controls get
checked by grep/AST on every build, behavioral controls by named tests.
"Detection without that second half is a vibe."

## Known draft gaps

- The web/async **seam layers** (server-web-archetype-fusions,
  server-async-archetype-fusions), the **client corpus** (CL* ids), and the
  maintainers' **open-issues ledger** are not yet published; references to
  them are forward references.
- Known ledger item: five controls (C08-C11, C50) are flagged as
  web/DB-plane-coupled (their proofs assume SQL transactions / outbox
  tables / endpoint flags), pending a generic-obligation / surface-proof
  split. For file-state CLIs, C33 (atomic write) and C19 transfer cleanly;
  read the STATE seam's C08-C11 mentions with that in mind.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — see LICENSE.
