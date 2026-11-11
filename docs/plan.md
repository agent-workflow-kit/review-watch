# review-watch: implementation plan

This plan implements [interface.md](interface.md) and [design.md](design.md). Issues are expanded into bite-sized test-first steps when they are picked up; this document fixes the order, the conventions and the version roadmap.

## Conventions for every stage

- **Gateway boundary.** Every subprocess call (forgectl, review-run, pr-comment, gatectl, jobsctl, the editor, crontab, detached review processes) goes through `src/review_watch/gateway.py`. No other module imports `subprocess`. Tests replace the gateway with fakes.
- **Fakes and pinned schemas.** Fake binaries for forgectl, review-run, pr-comment, gatectl and jobsctl live in `tests/fakebin/` (copied per repo, never shared) and print documents that validate against pinned copies of the siblings' schemas in `tests/schemas/<tool>/` (each pinned to a sibling version). A pinned schema is never imported at run time.
- **Process and live tests.** Process tests run real detached processes, real cron blocks and real SQLite against the fakes. `live` tests (real hosts, real `claude` through review-run) are marked `live`, deselected by default and never run in CI.
- **Golden files.** Every `--json` output, hook payload, audit and log record has a golden file validated against `schemas/`; human output is not golden.
- **Spike regressions.** No spike ran in this session; the sibling behaviors listed in `design.md` (Live verification) each become a `live` test or keep the conservative default named there.
- **Transitions.** Every state change is a compare-and-set on the round's `rev`; a property test drives random interleavings of `poll`, `show`, `accept`, `reject`, `resume` and `rerun` and checks that no terminal state changes and no round is posted twice.
- **`make verify`** (ruff, pyright, pytest, markdown lint) passes before every commit. The private-content guard hooks are never bypassed.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `config` | 0 | Pydantic models, drop-ins, precedence, the stricter-only rule for per-repo safety values |
| `store` | 0 | SQLite, migrations, leases, compare-and-set |
| `sanitize`, `log`, `audit` | 0 | the one sanitizer, `log.jsonl`, `audit.jsonl` |
| `gateway` | 0 | every subprocess call and the fakes' contract |
| `discover` | 1 | host listing, snapshots, scope, identity, skip rules, item matching |
| `tiers` | 2 | deterministic tiers and the hard rules |
| `rounds` | 3 | the state machine, caps, spend, retries, staleness |
| `launch` | 3 | jobsctl submission, detached processes, capacity |
| `finalize` | 3 | reading review-run, copying drafts, resolving held jobs, cancel |
| `post` | 4 | edit revisions, accept, re-anchoring, pr-comment calls |
| `gate` | 5 | gatectl calls, the declared input, outcomes |
| `hooks` | 6 | hook execution, payloads, outbox |
| `tui` | 6 | the optional Textual browser |
| `cli` | 0 to 6 | subcommands, exit codes, `--json`, the error envelope |

## Stage 0: foundation

**Goal**: A package that loads and validates config, opens and migrates the store, writes the log and audit, prints the error envelope and `--version`.
**Success Criteria**: `config.example.yaml` validates against `schemas/config.json`; the stricter-only rule and the required `hosts` key fail fast with the key named; the store creates every table, takes and releases leases (a dead holder's lease is taken over) and applies a compare-and-set; the sanitizer marks control and bidirectional characters and caps length; a newer store version exits 12 `state_unsafe`.
**Tests**: config precedence and drop-ins; per-repo override rules; migration from an empty store; lease takeover with a dead PID; sanitizer table tests; log and audit golden records; `schemas/` meta-validation.
**Status**: Not Started

## Stage 1: discovery

**Goal**: `poll --no-start` lists review requests on the configured hosts, snapshots candidates, applies scope and skip rules and records items; `list`, `show` and `status` read them.
**Success Criteria**: items are keyed `(pr_id, reviewer, instance)`; a new instance is a new item, a new head under one instance is not, a null instance starts once; every skip reason from the interface is recorded and re-evaluated; out-of-scope requests are counted and not recorded; a failed host or an incomplete listing changes nothing and is reported; `poll` exits 0 with errors in `errors[]` and 12 only as specified.
**Tests**: golden `poll`, `list`, `show`, `status` output; fake forgectl scenarios for GitHub and GitLab shapes (finished GitLab requests, team requests next to a direct one, null instance, null `is_bot`, draft made ready, `already_reviewed` with and without the marker); a host failing in the middle of a cycle; the `max_snapshots` budget; a second `poll` exiting 0 `already_running`.
**Status**: Not Started

## Stage 2: tiers and hard rules

**Goal**: Pure functions that compute the tier, its reasons and the hard-rule result.
**Success Criteria**: every row of the tier table and every hard rule behaves as specified; an unknown fact gives tier 3 or fails a rule closed; model output only raises the tier; per-repo lists add to and never remove from the global ones.
**Tests**: table-driven tests for each rule and threshold edge; a property test that removing a fact never lowers a tier; `off_limits` glob matching (`*`, `**`, renames: both paths count); `meta.review_run` signal combinations.
**Status**: Not Started

## Stage 3: rounds, launch and finalize

**Goal**: `poll` creates rounds, submits them through jobsctl or detached processes, finalizes runs, handles escalation, retries, staleness and closed PRs.
**Success Criteria**: the round states and transitions of the interface hold; the cap and spend-cap rules apply as specified; no round is submitted twice (an existing run is found by label first); `pass_env_undefined` and `created: false` are handled; a dead job or `poll` loses nothing and the next `poll` finalizes; a held job is resolved when the run ends or the round is rejected; an escalated run makes an `escalated` round and fires nothing twice; a merged or closed PR closes the item; a head change marks a drafted round stale.
**Tests**: scenario tests against the fakes for each transition; both exit-13 shapes of review-run; the unique-key reuse; retry after a retryable failure up to `max_attempts`; detached-process liveness by PID plus start time; the property test of Conventions; golden hook payloads for `post-attention`.
**Status**: Not Started

## Stage 4: drafts and human actions

**Goal**: `edit`, `accept`, `reject`, `rerun` and `resume` work against pr-comment and review-run.
**Success Criteria**: revisions are kept with hashes and refuse edits to the identity fields; `accept` shows pr-comment's plan, binds the post with `--expect-plan` and pins head and base; a stale round is refused with the right reason and `--reanchor` stores `reanchored_draft`; a failure after the write may have started leaves `posting` and the same `accept` is the answer; every pr-comment exit passes through with its fields; `rerun` and `resume` follow the interface including the caps and human-only rules.
**Tests**: golden `accept` results and dry runs; each pr-comment refusal and failure shape from its schemas; the editor round trip with a fake editor; no-TTY behavior (exit 13); `request_changes` with `--approve`; idempotent `accept` after a simulated crash between the post and the state write.
**Status**: Not Started

## Stage 5: automatic actions and outcomes

**Goal**: Gate calls, automatic post and approve, outcome filing and the optional spend cap.
**Success Criteria**: a gate is called once per round per policy with only declared fields; only `enforced: true` with `allow` acts; shadow, `off`, `not_applicable`, deny, escalate, errors and a missing gatectl leave the draft for a human; approve follows the post gate and never applies to a re-anchored draft; outcomes are filed exactly as the table in Gate outcomes says and once per decision id; the spend cap queues rounds and alerts once a day, and shows `cost_unavailable` until review-run reports `cost_usd`.
**Tests**: a fake gatectl returning every result shape; golden gate inputs; the outcome table; hard-rule interplay; `status` with and without gatectl; the spend cap across midnight in a non-UTC time zone.
**Status**: Not Started

## Stage 6: hooks, cron, housekeeping, TUI and release checks

**Goal**: Hooks with the outbox, `cron`, `export`, `prune`, `browse`, the `live` suite and the macOS checklist.
**Success Criteria**: hooks deliver at most once with `event_id`, never carry text, and `pre-round` veto and escalate behave as specified; the cron block matches the golden shape and is replaced idempotently, including a block without an end line; `prune` never touches a live item; the TUI covers every action, writes `--result-file` only when asked and refuses to start without a TTY; the `live` suite passes against both hosts; the README and examples match the interface.
**Tests**: hook payload goldens; outbox redelivery after a crash; cron golden and replacement tests (the awkit block shape included); a Textual pilot test per key; the `live` list in `design.md`; the macOS checklist run by hand.
**Status**: Not Started

## Version roadmap

- **0.x:** stages 0 to 6 in order; each stage ships as `feat` PRs behind `make verify`. 0.1 is the first release with stage 3 (discovery and reviews, drafts read through `show --draft`), 0.2 adds stage 4 (accept), 0.3 stage 5 (automatic actions), 0.4 stage 6.
- **1.0** requires: the `live` suite green on GitHub and GitLab with real review-run runs, a real escalation resumed, a held job resolved, the gate flow exercised in shadow for a week on the author's machine, and the macOS checklist done by hand, plus the sibling interfaces (forgectl, review-run, pr-comment) being at least at their own stable versions.
- **1.x (additive):** an early-poll trigger from a wired pr-watch hook (documented recipe becomes a command), team-level request discovery if forgectl gains a listing, approving an already posted draft, a revert detector filing `gatectl outcome --signal revert`, per-author rules, notification templates, a `doctor`-style readiness command, and the 1.x tracking issues in the issue list.
- **2.0:** nothing planned; interactive review mode and a learned risk score are not designed.
