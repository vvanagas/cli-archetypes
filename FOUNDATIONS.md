# Foundations — the principle, the 12 stages, the markers

> **Provenance note:** this document is *distilled* from the corpus files in
> this repository. The original foundation text ("CONSOLE UTILITY DESIGN
> reference") is not yet published; every statement here is recoverable from
> the published dissection, combinations, and seams files — this just puts
> the foundation in one place, first.

## The failure-mode-first principle

> Two tools belong to the SAME archetype if they fail the same way.
> Same failure mode → same discipline → same scaffolding → same review
> checklist. When in doubt about classification, ask: **what is the
> dominant way this can be wrong in production?** The answer is the
> archetype.

Framework and protocol labels are *not* the unit of classification — a
"REST tool" and a "gRPC tool" that fail identically are one archetype; two
subcommands of one binary that fail differently are two.

## The 12-stage pipeline

Every tool invocation — CLI run, HTTP request, daemon event — passes
through the same conceptual pipeline. The stages are a **checklist for
thinking, not a directory structure** (see the rule below).

| # | Stage | What it is | Typical CLI form |
|---|-------|-----------|------------------|
| 1 | **Transport accept** | How the invocation arrives — the entire input surface | argv + stdin; for a daemon: timer/signal/socket/inotify events (NOT argv — modeling a daemon as "a CLI that loops" is the signature anti-pattern) |
| 2 | **Parse** | Raw input → structure | argv→flags; bytes→tree; the plan file→step graph. Dominant for transformers and orchestrators |
| 3 | **Routing** | Which operation handles this | subcommand dispatch; event-type→handler. Overkill for single-op tools |
| 4 | **Pre-handler middleware** | Setup that isn't the work itself | client/auth/region config; **lock acquisition**; signal handlers; daemon bootstrap. The heavy stage for checkers (client setup) and stateful CLIs (the lock) |
| 5 | **Binding** | Parsed values → typed inputs — the "legal invocation" contract | `{indent:"2"}` → `{indent:2}`; our flags → the child tool's argv (a wrapper's defining stage) |
| 6 | **Validation** | Is the input acceptable — three distinct kinds: **6a** invocation (bad flags — fail before reading stdin), **6b** parse-validity (malformed input; the parser IS the validator), **6c** semantic (violates a schema/rule set — exists ONLY if the tool is, in part, a validator) | adding 6c to a non-linter means you're building a validator and didn't notice |
| 7 | **Authorization** | Who may do this | in ops tooling: almost always **REMOTED** (the API server/IAM enforces; you map the 403) or **DROPPED** (no principal). Returns only for socket daemons authorizing local peers |
| 8 | **Handler** | The thin connector from edge to core | **never load-bearing, in any archetype**. A fat handler is a defect regardless of archetype — logic here is the fat-controller smell |
| 9 | **Application service** | Orchestrating the work | spawn+lifecycle (wrapper); graph executor (orchestrator); the plan/apply workflow with WAL checkpoints (stateful) |
| 10 | **Domain** | The decision logic, pure where possible | the transform (transformer — THE core); evaluation rules (checker); diff/plan computation (stateful); **deliberately near-empty for a wrapper** — a growing wrapper domain means you're reimplementing the child |
| 11 | **Side-effect boundary** | Where the world changes or external state is touched | the state DB; the spawned child; the remote query; cross-event daemon state (bounded, or leaks kill the process). Renamed from "persistence" because the old word kept apologizing for itself on tools that persist nothing |
| 12 | **Response shaping** | Output + the exit code | stdout carries data (logs to stderr, or `tool \| jq` breaks); `--format json`; **semantic exit codes — for a checker, the exit code IS the primary product**. Inverts for daemons: no exit, emit a metric/health state instead |

Cross-cutting (not stages — they span all of them):

- **Error mapping** — each failure *class* gets a distinct exit code /
  response. Don't overload `1`. "Checked and it's bad" ≠ "couldn't check."
- **Observability** — progress/logs to stderr; verdict/data to stdout;
  under systemd, stdout *is* the log sink and the rule inverts.
- **Transaction scope** — what commits together. Stateful CLIs: many short
  transactions (intent → effect → completion), never one long transaction
  wrapping network I/O.

## The markers

Per archetype, each stage carries one mark:

| Marker | Meaning |
|--------|---------|
| *(none)* | Load-bearing — implement as a distinct seam |
| `OVERKILL` | Exists in theory; a separate module for it is over-engineering for THIS archetype — collapse it |
| `DROP` | Genuinely absent — no analog exists |
| `REMOTED` | Exists, but lives on the far side of the wire (you handle its *result*) |
| `UNDER` | Inverse mark (daemons): commonly *skipped* in the wild but must not be — under-building is the archetype's signature failure |

The cross-archetype constants: stage 7 is overkill or remoted in nearly
every ops archetype, and **stage 8 is never load-bearing in any of them**.

## Present is not a module

A stage appearing in the pipeline does NOT mean it becomes a file, class,
or module. A stage becomes code — a real seam — only when it has at least
one of:

- independent policy (it can change without the others)
- distinct failure modes (it fails in its own way)
- its own tests (you would test it in isolation)
- a swappable implementation (prod vs test vs stub)

A stage with none of these is a *concept*: name it in the design, fuse it
in the code. This rule is what makes the marker scheme safe to use — and
why the three archetypes that resist collapse (orchestrator, stateful CLI,
daemon) are exactly the high-stakes ops tools where the framework earns
its keep.

## The archetype test (applies to everything here)

Nothing enters this corpus by being a nice idea. An archetype earns
existence by **recurrence** (seen more than twice in real systems) AND
**irreducibility** (its default control set is not expressible as
conditional triggers on an existing archetype — "fails in a way nothing
else does," made mechanical). A control earns existence by recurrence and
a checkable shape. Even the *pipeline stages of the corpus's own design
process* are held to the same test: a named failure mode killed +
irreducibility to a neighbor — anything that fails it becomes a
conditional on something that passed, not a new thing.

And the standing rule that keeps all of it honest: **what runs that fails
if this is violated?** No answer → it's not a control, it's literature.
