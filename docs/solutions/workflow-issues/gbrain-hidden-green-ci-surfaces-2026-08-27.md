---
title: "GBrain hidden-green CI/local surfaces — exit 0 while omitting coverage"
date: 2026-08-27
category: workflow-issues
module: gbrain-testing-ci
problem_type: workflow_issue
component: testing_framework
severity: high
applies_when:
  - "Trusting bun run test, verify, check:all, or test:full as full coverage"
  - "Reading test-status green on a PR without checking cache-check.hit"
  - "Assuming e2e.yml or heavy-tests.yml gate the default Test workflow"
  - "Adding docs/** contracts without ci-cache-hash ALLOW_PATTERNS"
  - "Shipping fail-closed test guards for EXIT-HANG, isolation allowlist, or ZE vacuous pass"
tags:
  - "hidden-green"
  - "exit-hang"
  - "cache-check"
  - "test-status"
  - "verify-parity"
  - "isolation-allowlist"
  - "zeroentropy"
  - "brainbench"
related_components:
  - "development_workflow"
  - "tooling"
  - "documentation"
---

# GBrain hidden-green CI/local surfaces — exit 0 while omitting coverage

## Context

On 2026-08-27 a deterministic BUILD+TEST audit of `garrytan/gbrain` (local checkout `/Users/anshmudgil/Projects/gbrain`) mapped every TypeScript/CI surface that can **exit 0 while omitting coverage** — "hidden green." The audit is documentation-only: recommended fail-closed tests were specified and **not** implemented. Do not treat this page as proof those guards shipped.

Product evidence lives in gbrain scripts and workflows (`docs/TESTING.md`, `scripts/run-unit-parallel.sh`, `.github/workflows/test.yml`, `e2e.yml`, `heavy-tests.yml`). This page is the compounding capture so the next session retrieves the map instead of re-walking `package.json`.

Writable mirror for upstream PR work: `~/Desktop/Workspace/forks/gbrain-fork` (`anshmudgil/gbrain-fork`). Do not push engineering learnings to `garrytan/gbrain` without a PR.

## Guidance

### 1. Pipeline surfaces omit by design — exit 0 is not "full suite"

| Command / surface | Proves | Silently omits | Exit 0 on omit? |
|---|---|---|---|
| `bun run test` (`run-unit-parallel.sh`) | Fast unit shards + `*.serial.test.ts` | typecheck; all `check:*`; `*.slow.test.ts`; `test/e2e/*`; heavy | **yes** |
| `bun run verify` (`CHECKS` in `run-verify-parallel.sh`) | Parallel pre-gates + `typecheck` | unit/e2e/slow/brainbench; CI-only `check-bun-test-timeout.sh`; `check:all`-only extras | **yes** |
| `bun run check:all` | Sequential subset of greps | Most of verify (15+ gates); typecheck | **yes** — **not** a verify superset |
| `bun run test:slow` / `test:serial` | That quarantine only | Everything else | **yes** |
| `bun run test:full` | verify + units + slow + conditional e2e | e2e when `DATABASE_URL` unset (`\|\| echo` notice) | **yes** |
| `bun run test:e2e` | Real-Postgres e2e runner | Suites that `describe.skip` without DB/keys | **yes** if 0 fails |
| `test:heavy` / `heavy-tests.yml` | Ops-shape / opt-in nightly | Default PR CI | **yes** when skipped |
| CI `cache-check` hit | Prior green at content hash | Entire gitleaks/verify/serial/slow/brainbench/matrix | **yes** (`test-status` exit 0) |
| CI `test` matrix (`test-shard.sh`) | Weight-sharded units **incl.** slow (minus outliers); **excl.** serial | serial; e2e; typecheck (verify job) | n/a |
| CI `brainbench` | HEAD vs `origin/master` baseline (or first-landing) | Ungated run if no baseline anywhere | can **yes** |
| CI `test-status` | Aggregate "did Test workflow pass?" | **`e2e.yml` and `heavy-tests.yml` entirely** | **yes** if only this check is required |
| `e2e.yml` Tier1 / jsonb-parity | Postgres-backed mechanical + hard DB require (#2339 class) | Tier2 if secrets/openclaw missing | Tier2: **yes** on skip |
| `e2e.yml` Tier2 | Skills + ZE live | ZE tests that `return` without `test.skip` | **yes** (vacuous **pass**) |

Counts at audit time: **41** `check:*` keys in `package.json` (40 excl. `check:all`); **38** `CHECKS` entries (37 `check:*` + `typecheck`); **25** scripts in `check:all`; **51** isolation allowlist entries; **1** CI verify extra beyond CHECKS (`check-bun-test-timeout.sh`).

**In package `check:*` but not in verify CHECKS:** `check:admin-embedded`, `check:exports-count`, `check:newlines` (plus `check:all` itself).

### 2. Hidden-green list (severity ↓)

1. **S1 — EXIT-HANG warn-pass.** `scripts/run-unit-parallel.sh` ~533–572; pinned in `docs/TESTING.md` ~64–70. Watchdog kill + idle ≥300s + 0 `(fail)` + all assigned files started → pass-with-warning. Residual: last-file import hang. **Local `bun run test` only** — CI uses `test-shard.sh` and does not get this absolution path.

2. **S2 — CI cache hit skips the whole suite.** `.github/workflows/test.yml` cache-check (~39–79); jobs `if: hit != 'true'`; `test-status` (~341–361) HIT → exit 0. Hash deny-list: `scripts/ci-cache-hash.sh` — new contracts under denied `docs/**/*.md` without ALLOW_PATTERNS can share a green hash. `cache-write` correctly requires `success()` first.

3. **S3 — `test:full` E2E omit exits 0.** `package.json` `test:full` uses `\|\| echo '[test:full] skipped E2E…'`.

4. **S4 — Isolation allowlist (51 files).** `scripts/check-test-isolation.sh` + `scripts/check-test-isolation.allowlist` — verify stays green while allowlisted files can leak env/mocks/PGLite across shards.

5. **S5 — Local `bun test` / `bun run test` without typecheck.** Type errors only fail `verify` / `typecheck`.

6. **S6 — E2E `hasDatabase()` / Tier2 / agent doors → skip green.** Widespread `describe.skip` / `skipIf`; `heavy-tests.yml` `real-agent-e2e` can `exit 0`. Mitigation: `e2e.yml` jsonb-parity hard-requires `DATABASE_URL`.

7. **S7 — ZEROENTROPY vacuous pass.** `test/e2e/zeroentropy-live.test.ts` — `if (skipAll) { console.warn…; return; }` inside `test(` → bun counts **pass**, not skip.

8. **S8 — OOM / external-kill serial rescue.** `run-unit-parallel.sh` ~711–782 sets `TOTAL_RC=0` with `oom_rescued` note (mostly local).

9. **S9 — `check:all` ↔ `verify` divergence.** Neither is a superset of the other; operators who run only `check:all` miss typecheck and many verify gates.

10. **S10 — BrainBench ungated first landing.** `scripts/ci-brainbench-gate.sh` — no baseline on ref and none committed → run without `--compare`.

11. **S11 — Heavy placeholder / missing door scripts exit 0.** Contrast: `hermes-door` refuses green if `pass_count == 0`.

12. **S12 — Empty shard exit 0.** `run-unit-shard.sh` / `run-e2e.sh` when misconfigured `SHARD` yields no files.

13. **S13 — Separate workflows not in `test-status`.** If branch protection only requires `test-status`, red/skipped E2E/heavy is invisible.

### 3. Can `test-status` go green if a required job is skipped?

| Situation | Green? |
|---|---|
| **Cache HIT** (`needs.cache-check.outputs.hit == true`) | **Yes.** Designed — gated jobs skipped; aggregate exit 0. |
| **Cache MISS** | **No.** Loop requires every listed result `== "success"`. `skipped` / `failure` / `cancelled` fails the aggregate. |

`e2e` / `heavy` are never in this aggregate.

### 4. Fail-closed prevention contract (specified, not implemented)

Do **not** implement from this page unless a tracked ticket + PR says so. Exact deterministic additions:

1. `test/scripts/forbid-exit-hang-warn-pass.test.ts` — remove warn-pass or gate behind `GBRAIN_TEST_EXIT_HANG_FAIL=1` default-on in CI; assert CI `test-shard.sh` never contains warn-pass / EXIT-HANG absolution.
2. `test/scripts/test-full-e2e-omit-fails.test.ts` — under `GBRAIN_TEST_FULL_REQUIRE_E2E=1`, unset `DATABASE_URL` → non-zero; pin `package.json` fail-closed branch.
3. `test/scripts/check-test-isolation-allowlist-shrink.test.ts` — ceiling + forbid new paths vs `origin/master`.
4. `test/scripts/verify-checks-parity.test.ts` — `CHECKS` ⊇ every `check:*` except explicit `LOCAL_ONLY`; `typecheck` ∈ CHECKS; `check:admin-embedded` not orphaned.
5. `test/scripts/ci-test-status-no-skip-on-miss.test.ts` — cache-miss rejects `skipped`; `brainbench` stays in the aggregate loop.
6. `test/e2e/zeroentropy-live-no-vacuous-pass.test.ts` — forbid `if (skipAll) { return; }` inside `test(`; require `describe.skipIf` / `test.skip`.
7. `e2e-status` mirror of `test-status` (or branch-protection doc) so Tier1/jsonb-parity cannot be invisible.
8. Add `scripts/check-bun-test-timeout.sh` to local `CHECKS` so `verify` matches CI verify extras.
9. BrainBench: delete ungated path or `exit 2` once a baseline exists on `MAIN_REF`.
10. `test/scripts/ci-cache-hash-policy-docs.test.ts` — every `docs/**` path referenced by a `check:*` or content-contract test must be in ALLOW_PATTERNS.

## Why This Matters

Operators and agents treat green as "safe to merge." On GBrain, green often means "this wrapper's omitted set is empty of failures," not "all coverage classes ran." Cache HIT, warn-pass hangs, vacuous ZE passes, and `check:all`≠`verify` create false confidence that survives branch protection when only `test-status` is required. Capturing the map once compounds: the next PR that touches test scripts retrieves S1–S13 instead of rediscovering them from bash.

Related pattern: `brain/patterns/fail-closed-trust-boundary.md` (deny by default at trust edges). Hidden-green is the same idea for **coverage edges**.

## When to Apply

- Before claiming "CI is green" or "I ran everything locally."
- When changing `package.json` test scripts, `run-unit-parallel.sh`, `run-verify-parallel.sh`, `test.yml`, `e2e.yml`, or isolation allowlist.
- When adding markdown contracts under `docs/` that must invalidate the CI content cache.
- When designing fail-closed meta-tests (the 10-item contract above).
- When branch-protection checklists are edited — ask whether `e2e` / `heavy` are required.

## Examples

**False confidence — local**

| Operator ran | Thought | Actually omitted |
|---|---|---|
| `bun run test` | Full unit confidence | typecheck, verify greps, slow, e2e |
| `bun run check:all` | "All checks" | Most of `verify` + typecheck |
| `bun run test:full` without Postgres | "Everything CI runs" | E2E (exit 0 via `\|\| echo`) |

**False confidence — CI**

| Signal | Thought | Reality |
|---|---|---|
| `test-status` green + cache HIT | Suite re-ran | Jobs skipped; prior hash reused |
| `test-status` green | E2E covered | `e2e.yml` never aggregates here |
| ZE live job green without secret | Live embed coverage | Vacuous **pass** via early `return` |

**Before / after for ZE (guidance only)**

```typescript
// BEFORE (vacuous pass) — audit-observed pattern
if (skipAll) {
  console.warn('[skip] ZEROENTROPY_API_KEY not set');
  return; // bun counts pass
}

// AFTER (fail-closed / honest skip) — not shipped; prevention contract
describe.skipIf(skipAll)('ZE live — embed round-trip', () => {
  // ...
});
```

## Related

- Product: `~/Projects/gbrain/docs/TESTING.md` (EXIT-HANG warn-pass triage; CI matrix vs local wrappers)
- Product workflows: `.github/workflows/test.yml`, `e2e.yml`, `heavy-tests.yml`
- Engineering-gbrain: `brain/patterns/gbrain-hidden-green-ci-surfaces.md`, `brain/patterns/fail-closed-trust-boundary.md`
- Fork mirror: `~/Desktop/Workspace/forks/gbrain-fork/docs/solutions/workflow-issues/` (same filename)
- Desktop return arrow: `~/Desktop/agent native engineering workflows/04-compound-learning.md`
- Do **not** confuse with ASI fulfillment / ontology solutions pages — different clock and repo surface
