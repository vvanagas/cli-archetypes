# CLI Archetypes — AGENTS.md block (Codex / any AGENTS.md-convention agent)

Paste the section below into your repository's `AGENTS.md`, and vendor the
five corpus files (adjust paths if you place them elsewhere).

---

## CLI design & review discipline (cli-archetypes corpus)

When designing, changing, or reviewing any command-line tool, daemon, or
cron job in this repository, consult the corpus in `docs/cli-archetypes/`:

- `archetype-stage-dissection.txt` — six archetypes on a 12-stage pipeline
- `archetype-combinations.txt` — fusions and pipe compositions
- `archetype-fusion-seams.txt` — the four seams, smells, check procedure
- `server-archetype-control-map.txt` — per-archetype control cards
- `server-controls.txt` — the control catalog (obligation / floor / proof)

Rules:

1. CLASSIFY FIRST. Before writing code, name which of the six archetypes
   (transformer, invoker, orchestrator, stateful, daemon — or a fusion)
   the tool is. Read that card in the control map; its defaults are your
   obligations, its forbidden list is the misreach warning.
2. On review, run the seams file's DETECTION FIRST smell list. Any change
   adding a gating flag (--check, --fail-if) means the tool grew a checker:
   design its exit codes (distinct code per outcome class, documented) —
   never inherit them from the underlying call.
3. Exit-code contract: "thing is broken" and "could not check" NEVER share
   an exit code. One exit owner per binary, and each outcome class maps to
   a distinct, tested code.
4. State files: temp-write + fsync + atomic rename, never in-place; one
   lock owner across all layers; record-completion only AFTER the side
   effect.
5. Children/subprocesses: always a timeout; the child's output re-enters as
   untrusted input (validate; prefer its --format json); host logs go to
   stderr, never into relayed stdout.
6. Output a CHECKLIST (control: present / absent / floor-dropped, with
   evidence) — not prose reassurance. Absent with no floor is a gap to
   close or explicitly justify.
