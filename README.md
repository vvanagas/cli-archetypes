# CLI Archetypes, Stages & Fusion Seams

Every CLI invocation — and every HTTP request, and every daemon event —
passes through the same **12-stage pipeline**. Six CLI archetypes light
those stages up differently, and that difference is the whole model:

| # | Stage | In a CLI |
|---|-------|----------|
| 1 | Transport accept | argv + stdin — or, for a daemon, **events** (timer/signal/socket/inotify): modeling a daemon as "a CLI that loops" is the signature anti-pattern |
| 2 | Parse | argv→flags; bytes→tree; plan-file→step graph — dominant for transformers and orchestrators |
| 3 | Routing | subcommand → operation; event type → handler |
| 4 | Pre-handler middleware | client/auth setup; **lock acquisition** (skip it in a stateful CLI = corruption); signal handlers; daemon bootstrap |
| 5 | Binding | parsed values → typed options: the "legal invocation" contract; a wrapper's flag-translation lives here |
| 6 | Validation | three kinds: 6a bad flags (fail before reading stdin) · 6b malformed input (the parser IS the validator) · 6c schema/rules — which exists only if the tool is partly a validator |
| 7 | Authorization | almost always **remoted** (the far API/IAM enforces; you map the 403) or dropped — ops tools rarely authorize locally |
| 8 | Handler | the thin edge→core connector — **never load-bearing, in any archetype**; a fat handler is a defect everywhere |
| 9 | Application service | spawn + child lifecycle (wrapper); the graph executor (orchestrator); plan→apply with checkpoints (stateful) |
| 10 | Domain | the decision logic: the transform, the evaluation rules, the diff — and deliberately **near-empty for a wrapper** (growth here = reimplementing the child) |
| 11 | Side-effect boundary | the state DB; the spawned child; the remote query; cross-event daemon state (bounded, or leaks kill the process over days) |
| 12 | Response shaping | stdout = data, stderr = logs (or `tool \| jq` breaks); **semantic exit codes — for a checker, the exit code IS the product** |

Cross-cutting all twelve: **error mapping** (distinct exit code per failure
class — "it's broken" ≠ "couldn't check"), **observability**, and
**transaction scope** (many short transactions, never one long one wrapping
network I/O).

An archetype is which stages dominate, which collapse, and which invert —
the per-stage verdict for all six is `archetype-stage-dissection.txt`; the
principle and the full stage semantics are in `FOUNDATIONS.md`. And the
stages are a checklist for *thinking*, not a directory layout: a stage
becomes a module only when it has its own policy, failure modes, tests, or
swappable implementation ("present is not a module").

## Why you might care, concretely

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

## Audit one CLI in 10 minutes (start here)

Do this before reading any theory:

1. **Run the greps** on a tool you own:

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

2. **Classify the tool** — one question: *what is the dominant way this can
   be wrong in production?* Pick from: transformer (bad output), invoker
   (wrong verdict about someone else's state), orchestrator (partial failure
   misreported), stateful (corrupted state file), daemon (dies or leaks).
3. **Map each grep hit** through the smell list at the top of
   `archetype-fusion-seams.txt` — it names the seam and the violated control.
4. **The only three findings to chase first** (highest frequency, highest
   damage, cheapest fixes):
   - **exit-code overload** — "it's broken" and "couldn't check" share a
     code → your CI gate lies;
   - **in-place state writes** — a kill mid-write corrupts the file your
     tool trusts most → temp + fsync + rename;
   - **no timeout on children** — one hung subprocess hangs you forever.
5. **Only then** read the theory: `FOUNDATIONS.md`, then the dissection.

A real audit following exactly this path found the first two of those three
in a green-suite, TDD-built gating tool, in under ten minutes —
`examples/worked-audit.md` is the full walkthrough.

## What's inside

| File | What it is |
|------|-----------|
| `archetype-stage-dissection.txt` | Six CLI archetypes (transformer, status checker, wrapper, orchestrator, stateful CLI, daemon) mapped onto a 12-stage pipeline, with per-stage markers: load-bearing / OVERKILL / DROP / UNDER. Includes the "present is not a module" rule. |
| `archetype-combinations.txt` | How archetypes combine: single-tool fusions (terraform, kubectl, alembic, node_exporter, k9s, make) and multi-tool pipe compositions, each with seam placement and the production caveats. |
| `archetype-fusion-seams.txt` | The detection layer: every CLI fusion is a stack of FOUR seams (VERDICT, STATE, SPAWN, ROLLUP). Smell list → seam → the controls the seam forces → how to check them. |
| `server-archetype-control-map.txt` | The control map (v0.9): per-archetype cards — web AND CLI — with default / conditional / forbidden control attachments and named triggers. |
| `server-controls.txt` | The control catalog (v0.9): every C-id with its obligation, its floor (when it may legitimately be dropped), and its behavioral proof — the engine the seams file resolves against. |
| `FOUNDATIONS.md` | The principle, the 12 stages with full semantics, the marker system, "present is not a module", and the archetype test — distilled from the corpus into one primer. |
| `server-corpus-open-issues.txt` | The maintainers' falsification ledger, published deliberately: 17 open findings AGAINST the corpus itself — mis-attached controls, judgment-flip risks, plane-coupling defects, candidate controls awaiting their promotion bar. The admission discipline, visible. |
| `examples/worked-audit.md` | A complete audit walkthrough: classify → grep → checklist → verdict. |
| `skills/` | Drop-in agent instructions for Claude Code and Codex — see below. |

**Why do files in a CLI repo say `server-`?** The catalog and the map are
*shared* across the server and CLI planes of a larger corpus — the names
record that origin and keep every cross-reference stable. The ledger's
OI-12 tracks the real underlying work (splitting web/DB-coupled proofs
from generic obligations); renaming follows that split, not the other way
around.

## What "archetype" means here (it is not a genre)

An archetype in this corpus is a **failure-mode equivalence class**: a
recurring tool shape that *breaks in a way nothing else does*. The
foundational principle (from the corpus's web-side master, publication
pending):

> Two tools belong to the SAME archetype if they fail the same way.
> Same failure mode → same discipline → same scaffolding → same review
> checklist. When in doubt about classification, ask: *what is the
> dominant way this can be wrong in production?* The answer is the
> archetype.

(Framework labels are NOT the unit of classification — which is why
"REST vs gRPC" is not an archetype distinction, and why a wrapper and a
checker that fail identically merged into one card.)

Two rules decide membership (published in the map's
`archetype_inclusion` section):

- **recurrence** — the shape appears repeatedly in real systems (more than
  twice, the same bar controls must meet);
- **irreducibility** — its default control set cannot be expressed as a few
  conditional triggers on an existing archetype without distorting that
  archetype's meaning.

That is why classification is the load-bearing step: an archetype is not a
label, it is a **control-set selector**. Say "stateful CLI" and you have
already said lock + atomic state writes + intent/completion ordering +
resume semantics — before reading a line of code. Say "wrapper" and you
have said the domain layer should be nearly empty, and a fat one is the
defect.

It also explains the two design moves that look odd at first glance:

- **the merge** — cli_wrapper and cli_status_checker became ONE archetype
  (cli_invoker) when their failure modes turned out to be flavors of the
  same thing, differing by exactly one trigger (verdict branching). Same
  failures, same archetype — however different the tools look.
- **the daemon inversion** — the daemon is marked UNDER (what people skip)
  where every other archetype is marked OVERKILL (what people over-build),
  because its signature failure is under-building. The markers follow the
  failure mode, not symmetry.

Function tells you what a tool does; archetype tells you how it will
break. The corpus classifies by the second, because that is the one you
audit.

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

**Honesty note on the agent story:** these instructions are field-used but
not yet *demonstrated* with a published eval — that eval (with-corpus vs
bare-prompt, scored on control coverage against known failure signatures)
is ledger item OI-10. Until it lands, treat the skill files as practitioner
instructions with one published worked audit behind them, not a proven
intervention.

## The method, in one paragraph

Classify the tool's layers against the six archetypes → each classification
attaches its card's default controls from the map, for free → the fusion
fires a few named conditional triggers a single-archetype tool would skip →
floors prune what's legitimately not needed → structural controls get
checked by grep/AST on every build, behavioral controls by named tests.
"Detection without that second half is a vibe."

## The four seams, in plain words

Every multi-archetype tool breaks at a *translation point* — where one
part's natural idiom must become another's. There are four, and they stack:

| Seam | The situation | How it breaks |
|------|---------------|---------------|
| **VERDICT** | Part of your tool produces a verdict (healthy/broken, drifted/clean) inside a part that owns the process exit — or has no exit at all | The verdict and a transport failure end up sharing an exit code; or an inner component calls `exit()` from inside a host that should own it. "It's broken" and "couldn't check" become indistinguishable |
| **STATE** | Something inner does mutating work while something outer owns the state file and the lock | Two locks deadlock; two state files disagree; the completion record lands *before* the effect it records; a kill mid-write leaves a torn file that defeats resume |
| **SPAWN** | You run a child tool or call a remote API — its stdout and exit code are now both your input and your problem | No timeout on the child; regexing its human-readable output; trusting it unvalidated; your logs bleeding into what you relay |
| **ROLLUP** | One invocation does N things whose N results must become one exit code and one report | "3 of 10 failed" has no representation in a binary exit; one failed sub-op crashes the run despite a continue-on-error intent; the partial-failure policy exists only implicitly |

## The controls you'll meet first

The corpus references controls by id (`C05`, `C33`, …); the catalog defines
all 58. The handful that appear in this README, in one line each:

- **C01** — anything entering from outside (child output included) is
  validated at the boundary, not trusted
- **C04** — when a caller must distinguish 2+ recoverable outcomes, they're
  a typed result, not a thrown surprise
- **C05** — each outcome class maps to a distinct, documented exit code
- **C06** — every external call and child process has a timeout
- **C12** — stdout carries data only; logs and progress go to stderr
- **C17** — side effects live behind a swappable port, so `--dry-run` can
  actually be dry and tests can stub the world
- **C32** — things that must not run twice concurrently hold a lock (with a
  floor: provably-single-instance tools may skip it)
- **C33** — state files are replaced by temp-write + fsync + atomic rename,
  never edited in place

## Known draft gaps

- The web/async **seam layers** (server-web-archetype-fusions,
  server-async-archetype-fusions) and the **client corpus** (CL* ids) are
  not yet published; references to them are forward references.
- The **open-issues ledger** IS published (`server-corpus-open-issues.txt`)
  — start with OI-12 if you want to see the corpus criticize itself with
  precision.
- Known ledger item: five controls (C08-C11, C50) are flagged as
  web/DB-plane-coupled (their proofs assume SQL transactions / outbox
  tables / endpoint flags), pending a generic-obligation / surface-proof
  split. For file-state CLIs, C33 (atomic write) and C19 transfer cleanly;
  read the STATE seam's C08-C11 mentions with that in mind.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — see LICENSE.
