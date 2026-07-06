# Worked audit — a baseline-gating CLI, end to end

A generalized replay of a real audit (internal tool, anonymized): a
TypeScript CLI that probes services, crawls routes, compares the snapshot
against an accepted baseline, and exits with a verdict — the thing your CI
calls before and after a production update. Green 88-test suite, TDD-built.
The audit took about ten minutes and found two HIGH defects the suite
couldn't see. Every step below is the corpus procedure, verbatim.

## Step 1 — classify (the load-bearing step)

Two entry points:

- `baseline-run` — one invocation drives N service probes + N crawled
  routes, joins them into one report and one exit code → outer
  **cli_orchestrator** (ROLLUP), hosting a **checker** (the comparison =
  VERDICT) and **spawns** (subprocess probes, a browser child = SPAWN).
- `baseline-accept` — mutates `accepted.json` + a history file → thin
  **cli_stateful** (STATE seam), no lock.

Seams present: **VERDICT + SPAWN + ROLLUP** on run; **STATE** on accept.
(Per the seams file: check all four, they stack.)

## Step 2 — attach defaults, fire triggers (free by classification)

From the map: cli_orchestrator defaults C04, C05, C12, C13, C16, C17, C18;
cli_stateful brings C33 via `persisted_run_state`; SPAWN's card defaults
add C01, C06; `handles_secret_token_or_credential` fires C25.

## Step 3 — structural greps (minutes, not meetings)

```bash
# exit sites + what they mean (VERDICT: smell #1)
rg -n 'process\.exit' src/
#   run.ts:82   return comparison.status === "fail" ? 1 : 0;
#   run.ts:94   process.exit(1);        <-- in the catch-all
#   accept.ts:26 process.exit(1);

# atomic state writes (STATE: C33)
rg -n 'writeFile|rename' src/store.ts
#   store.ts:156 writeFile(historyPath, ...)     <-- in place
#   store.ts:157 writeFile(acceptedPath, ...)    <-- in place, no rename anywhere

# stdout discipline (C12), timeouts (C06), child validation (C01)...
rg -c 'console\.log' src/          # 0 — clean
rg -n 'timeout' src/probes.ts      # explicit everywhere — clean
```

## Step 4 — the checklist (findings, not vibes)

| Control | Status | Evidence |
|---------|--------|----------|
| C05 distinct exit per outcome class | **ABSENT — violated** | `run.ts:82` exits 1 on regression; `:94` exits 1 on crash. "Stack drifted" and "couldn't check" share a code — a CI gate cannot distinguish them. Smell #1 verbatim. |
| exit-code behavioral test | **ABSENT** | No test asserts any exit code. Note: the structural single-exit-owner check PASSED (all exits in entry files) while the behavior was wrong — the corpus's recorded blind spot ("single ownership is necessary, not sufficient") catching its first live case. |
| C33 atomic state write | **ABSENT** | In-place `writeFile` of the gate's ground truth; kill mid-accept truncates `accepted.json`. |
| C13 documented exit codes | ABSENT (minor) | No table anywhere. |
| C12 stdout=data / stderr=logs | present | data → stdout, fatals → stderr, zero console.log. |
| C04 typed partial outcomes | present | down service records `status:"fail"`, run continues. |
| C06 child/probe timeouts | present | explicit on subprocess and fetch paths. |
| C01 child output validated | present | schema-validated at re-entry. |
| C25 secrets not echoed | present | env-only credential, stack traces gated behind a debug flag. |
| C32 single-instance lock | floor-dropped | manual operator runs, not cron — floor holds, recorded. |
| C27 SIGTERM checkpoint | floor-dropped | short runs, scratch outputs. |

## Step 5 — the fixes (same day, TDD)

1. **C05**: shared `EXIT_CODES = { clean: 0, regression: 1, harnessFailure: 2 }`;
   catch paths exit 2; spawned-CLI tests assert all codes; README gains the
   contract table + a migration note ("consumers gating on `exit == 1` must
   now also handle 2").
2. **C33**: `atomicWriteFile` (temp in same dir + fsync + rename) for all
   three store writes. Test note worth stealing: a read-only-file probe
   can't falsify atomicity when tests run as root (root bypasses permission
   checks) — the portable observable is the **inode swap**: rename installs
   a new inode; an in-place write keeps the old one.

Suite after: 91 pass / 0 fail.

## Why this example matters

The tool was *well built* — TDD, typed errors, injected clocks, a real
review history. Its suite pinned everything it asserted, and both defects
lived exactly in what nothing asserted: the exit-code *contract* and the
write *mechanism*. That's the corpus's job description: it looks where
suites structurally don't.
