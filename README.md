# CLI Archetypes, Stages & Fusion Seams

**Status: DRAFT** — published as-is from a working corpus. Rough edges are
part of the deal; the falsification annotations are the point.

A working taxonomy for designing and auditing command-line tools in the
infra / sysadmin / devops space:

| File | What it is |
|------|-----------|
| `archetype-stage-dissection.txt` | Six CLI archetypes (transformer, status checker, wrapper, orchestrator, stateful CLI, daemon) mapped onto a 12-stage pipeline, with per-stage markers: load-bearing / OVERKILL / DROP / UNDER. Includes the "present is not a module" rule. |
| `archetype-combinations.txt` | How archetypes combine: single-tool fusions (terraform, kubectl, alembic, node_exporter, k9s, make) and multi-tool pipe compositions, each with seam placement and the caveats that break them in production. |
| `archetype-fusion-seams.txt` | The detection layer: every CLI fusion is a stack of FOUR seams (VERDICT, STATE, SPAWN, ROLLUP). Smell list → seam → the controls the seam forces → how to check them structurally and behaviorally. |

## How it's meant to be used

Not as a style guide — as a **detection and audit layer**. Classify the
tool's layers, let each classification attach its default controls, fire the
seam triggers, prune by floors, then check: structural controls by grep/AST
on every build, behavioral controls by named tests. "Detection without that
second half is a vibe."

The corpus maintains itself by falsification against real tools: when a
check over- or under-reports on a real program, the blind spot is recorded
in the file (see the daemon sub-type split and the single-exit-owner blind
spot, both found via [modd](https://github.com/cortesi/modd), built and run).

## Known draft gaps

- `archetype-fusion-seams.txt` references a control **catalog**
  (`server-controls.txt` v0.9, ids C01..C48) and per-archetype **control
  cards** (`server-archetype-control-map.txt` v0.9) that are not yet
  published. The seam annotations carry enough per-control meaning to be
  usable standalone; the catalog follows in a later release.
- Known ledger item on the catalog side: five referenced controls
  (C08-C11, C50) are flagged as web/DB-plane-coupled (their proofs assume
  SQL transactions / outbox tables), pending a generic-obligation /
  surface-proof split. For file-state CLIs, C33 (atomic write) and C19
  transfer cleanly; read the STATE seam's C08-C11 mentions with that in
  mind.
- The "CONSOLE UTILITY DESIGN reference document" both companions cite is
  the unpublished source text these files were dissected from.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — see LICENSE.
