# review-watch: implementation plan

This plan implements [interface.md](interface.md) and [design.md](design.md). Issues are expanded into bite-sized test-first steps when they are picked up; this document fixes the order, the conventions and the version roadmap.

## Conventions for every stage

- **Gateway boundary.** Every subprocess call (forgectl, review-run, pr-comment, gatectl, jobsctl, the editor, crontab) goes through `src/review_watch/gateway.py`. No other module imports `subprocess`. Tests replace the gateway with fakes.
- **Fakes and pinned schemas.** Fake binaries for forgectl, review-run, pr-comment, gatectl and jobsctl live in `tests/fakebin/` (copied per repo, never shared) and print documents that validate against pinned copies of the siblings' schemas in `tests/schemas/<tool>/` (each pinned to a sibling version). A pinned schema is never imported at run time.
- **Process and live tests.** Process tests run real cron blocks and real SQLite against the fakes; the fake review-run returns each run object and error envelope of the run-outcomes table. `live` tests (real hosts, real `claude` through review-run) are marked `live`, deselected by default and never run in CI.
- **Golden files.** Every `--json` output, hook payload, audit and log record has a golden file validated against `schemas/`; human output is not golden.
- **Spike regressions.** No spike ran in this session; the sibling behaviors listed in `design.md` (Live verification) each become a `live` test or keep the conservative default named there.
- **Transitions.** Every state change is a compare-and-set on the round's `rev`; a property test drives random interleavings of `poll`, `show`, `accept`, `reject`, `resume` and `rerun` and checks that no terminal state changes, no round is posted twice, and the loser of every race exits `round_busy` or re-evaluates to a valid refusal.
- **Prose equals schema.** A test asserts that every enum list and every result-object field list in `docs/interface.md` equals the corresponding schema enum or `properties`, and that the rule-code prose lists equal the schema enums.
- **`make verify`** (ruff, pyright, pytest, markdown lint) passes before every commit. The private-content guard hooks are never bypassed.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `config` | 0 | Pydantic models, drop-ins, precedence, the stricter-only rule for per-repo safety values |
| `store` | 0 | SQLite, migrations, leases, compare-and-set |
| `sanitize`, `log`, `audit` | 0 | the one sanitizer, `log.jsonl`, `audit.jsonl` |
| `gateway` | 0 | every subprocess call and the fakes' contract |
| `discover` | 1 | host listing, snapshots, scope, identity, skip rules, item matching |
| `tiers` | 2 | deterministic tiers, the path matcher, fact binding and the hard rules |
| `rounds` | 3 | the state machine, caps, the daily limit, retries, staleness |
| `launch` | 3 | jobsctl submission, inline `review-run run --no-wait` starts, capacity |
| `finalize` | 3 | reading review-run, the run-outcomes table, copying drafts, resolving held jobs, cancel |
| `post` | 4 | edit revisions, accept, the post intent and its recovery, re-anchoring, pr-comment calls |
| `gate` | 5 | gatectl calls, the declared input, outcomes |
| `hooks` | 6 | hook execution, payloads, outbox |
| `tui` | 6 | the optional Textual browser |
| `cli` | 0 to 6 | subcommands, exit codes, `--json`, the error envelope |

## Stage 0: foundation

**Goal**: A package that loads and validates config, opens and migrates the store, writes the log and audit, prints the error envelope and `--version`.
**Success Criteria**: `config.example.yaml` validates against `schemas/config.json`; the stricter-only rule and the required `hosts` key fail fast with the key named; the store creates every table, takes and releases leases (a dead holder's lease is taken over) and applies a compare-and-set; the sanitizer marks control and bidirectional characters and caps length; a newer store version exits 12 `state_unsafe`; every identifier read from a sibling or a hook is validated against its grammar; temp files are `O_EXCL|O_NOFOLLOW` mode 0600.
**Tests**: config precedence and drop-ins; per-repo override rules and the cross-field checks of Config validation (`auto_approve.enabled` without `auto_post.enabled`, safety keys from the environment); prose field lists equal schema properties; temp-file creation and `state_unsafe` cases; a bad identifier is `lower_contract_mismatch`; the 50-revision and 8 MiB bounds; migration from an empty store; lease takeover with a dead PID; sanitizer table tests; log and audit golden records; `schemas/` meta-validation.
**Status**: Not Started

## Stage 1: discovery

**Goal**: `poll --no-start` lists review requests on the configured hosts, snapshots candidates, applies scope and skip rules and records items; `list`, `show` and `status` read them.
**Success Criteria**: items are keyed `(pr_id, reviewer, instance)`; a new instance is a new item, a new head under one instance is not, a null instance starts once; every skip reason from the interface is recorded and re-evaluated; out-of-scope requests are counted and not recorded; a failed host or an incomplete listing changes nothing and is reported; `poll` exits 0 with errors in `errors[]` and 12 only as specified.
**Tests**: golden `poll`, `list`, `show`, `status` output; snapshot selection (40 finished GitLab requests and one new request: the new one is snapshotted in the first cycle; permanent skips are never re-snapshotted; a GitHub `posted` item listed again is snapshotted; `snapshots_deferred` reported); instance golden scenarios (null then real instance gives one item; real then null the same item; `incomplete` review requests create no item); item transitions (a `failed` item closes when its PR merges; a reopened PR reopens the item without a round; three `not_found` reads set `unreachable`; a GitLab `reviewed` to `outstanding` change adds a `re_request` round); a renamed repo in fake forgectl updates the item and a changed `pr_id` is refused; fake forgectl scenarios for GitHub and GitLab shapes (finished GitLab requests, team requests next to a direct one, null instance, null `is_bot`, draft made ready, `already_reviewed` per host: GitHub only with a null `requested_at`, GitLab by `state`, own marker reviews ignored for an item with a round); a host failing in the middle of a cycle; the `max_snapshots` budget; a second `poll` exiting 0 `already_running`.
**Status**: Not Started

## Stage 2: tiers and hard rules

**Goal**: Pure functions that compute the tier, its reasons and the hard-rule result.
**Success Criteria**: every row of the tier table and every hard rule behaves as specified; an unknown fact gives tier 3 or fails a rule closed; model output only raises the tier; per-repo lists add to and never remove from the global ones.
**Tests**: table-driven tests for each rule and threshold edge; a property test that removing a fact never lowers a tier; the path matcher's golden table (`src/Auth/login.py`, `a/b.txt` against `a/*.txt`, `.github/CODEOWNERS`, ASCII case folding) and rename cases; one test per `meta_signal` source and each missing field; one test per rule code including `automatic_request_changes`, `approve_path_not_allowed`, `approve_author_not_trusted`, `edited`, `repo_changed`, `facts_unbound`; a pure-function check that facts read at another head fail closed.
**Status**: Not Started

## Stage 3: rounds, launch and finalize

**Goal**: `poll` creates rounds, submits them through jobsctl or inline with `review-run run --no-wait --json`, finalizes runs, handles escalation, retries, the daily limit, staleness and closed PRs.
**Success Criteria**: the round states and transitions of the interface hold; the per-PR cap and the daily limit (`limits.rounds_per_day`, built here and nowhere else) apply as specified; no round is submitted twice (an existing run is found by label first); `pass_env_undefined` and `created: false` are handled; a dead job or `poll` loses nothing and the next `poll` finalizes; a held job is resolved when the run ends or the round is rejected; an escalated run makes an `escalated` round and fires nothing twice; a merged or closed PR closes the item; a head change marks a drafted round stale; a retry never passes through `failed`.
**Tests**: scenario tests against the fakes for each transition; every row of the run-outcomes table with a fake review-run (envelope or run object) and a fake jobsctl; both exit-13 shapes of review-run; two items for one PR at one head get two jobs (key `review:<round id>`); `pass_env_undefined` cancels the created job; a crash after `add` recovers the job by key; a retryable failure then success leaves no `failed` record in `log.jsonl`, and `retry_after_s` is honored across two polls; the argv is `--json --mode headless` plus `--no-wait` inline, pinned to a golden; a pruned previous run is dropped (`previous_unavailable`) and the launch succeeds; a run anchored at H1 finalized after a push to H2 is `stale` before any gate step; the daily limit across midnight in a non-UTC time zone, one `post-attention` a day and `rerun --force` passing it; the property test of Conventions; golden hook payloads for `post-attention`.
**Status**: Not Started

## Stage 4: drafts and human actions

**Goal**: `edit`, `accept`, `reject`, `rerun` and `resume` work against pr-comment and review-run.
**Success Criteria**: revisions are kept with hashes and refuse edits to the identity fields; `accept` shows pr-comment's plan, binds the post with `--expect-plan` and pins head and base; a stale round is refused with the right reason and `--reanchor` stores `reanchored_draft`; `posting` stores its intent and only an unknown outcome stays `posting`, resolved by the same `accept` (re-anchored when the head moved and the intent has no `approve`, refused when it has one); every pr-comment exit passes through with its fields; `rerun` and `resume` follow the interface including the caps and human-only rules; `resume` never blocks.
**Tests**: golden `accept` results and dry runs; each pr-comment refusal and failure shape from its schemas; the editor round trip with a fake editor; no-TTY behavior (exit 13 `confirmation_required`) for each human-only command; `request_changes` with `--approve`; two concurrent `accept`s (one exits `round_busy`); a crash between the host write and the state write, then a push, then `accept` (the re-anchored retry reports `already_posted`); an approve intent with a moved head refuses; every row of the pr-comment result table with a fake pr-comment whose accepted flags are pinned to its synopsis (no `--expected-base`); `edit` on an `escalated` round exits 20 `not_editable`; fake-editor tests (`$VISUAL` before `$EDITOR`, none set, non-zero exit, invalid result keeps the temp file); `resume` returns at once with a fake review-run that would block, and the `pre_run` path resolves the held job and submits once.
**Status**: Not Started

## Stage 5: automatic actions and outcomes

**Goal**: Fact binding, gate calls, automatic post and approve and outcome filing.
**Success Criteria**: a gate is called once per round per policy with only declared fields; only `enforced: true` with `allow` acts; shadow, `off`, `not_applicable`, deny, escalate, errors and a missing gatectl leave the draft for a human; approve follows the post gate, uses one dry run with the final flags and never applies to a re-anchored or edited draft; the gate's tier comes from facts bound to the draft's head; outcomes are filed per gate exactly as the table in Gate outcomes says and once per decision id.
**Tests**: a fake gatectl returning every result shape and `--timeout 60` (whole seconds) in its argv; each warning blocks; golden gate inputs (`event` and `verdict` present); a push between creation and the session (the draft anchors H2 and the facts are read at H2); `files_complete: false` blocks; the outcome table per gate; hard-rule interplay including an edit between gate and post; a round finalized by `show` is gated by the next `poll`; three `gate_error`s then stop; an automatic post interrupted then retried by `poll`; a closed PR under `posting`; `status` with and without gatectl.
**Status**: Not Started

## Stage 6: hooks, cron, housekeeping, TUI and release checks

**Goal**: Hooks with the outbox, `cron`, `export`, `prune`, `browse`, the `live` suite and the macOS checklist.
**Success Criteria**: hooks deliver at least once with `event_id`, never carry text, use the object types of the interface, and `pre-round` veto and escalate behave as specified; the cron block matches the golden shape and is replaced idempotently, including a block without an end line is never replaced; `prune` never forgets an item of an open PR; the TUI renders untrusted text as plain text; the TUI covers every action, writes `--result-file` only when asked and refuses to start without a TTY; the `live` suite passes against both hosts; the README and examples match the interface.
**Tests**: hook payload goldens and object types (`round`, `item`, `host`); outbox redelivery after a crash with a stable `event_id`; a vetoed item is not re-asked on the next poll and is re-asked after a push; a held round is not submitted and `resume` releases it with the other hooks re-run; a rejected item of an open PR survives `prune --older-than 0` and the next poll starts nothing; cron refusals (`'`, `%`, a newline in a path) and the parsed-back comparison; a Textual pilot test with `[b]`, `[link=...]` and `[@click=app.quit]` titles; cron golden and replacement tests (the awkit block shape included); a Textual pilot test per key; the `live` list in `design.md`; the macOS checklist run by hand.
**Status**: Not Started

## Version roadmap

- **0.x:** stages 0 to 6 in order; each stage ships as `feat` PRs behind `make verify`. 0.1 is the first release with stage 3 (discovery and reviews, drafts read through `show --draft`), 0.2 adds stage 4 (accept), 0.3 stage 5 (automatic actions), 0.4 stage 6.
- **1.0** requires: the `live` suite green on GitHub and GitLab with real review-run runs, a real escalation resumed, a held job resolved, a `pre-run` veto and escalation exercised, the gate flow exercised in shadow for a week on the author's machine, and the macOS checklist done by hand, plus the sibling interfaces (forgectl, review-run, pr-comment) being at least at their own stable versions.
- **1.x (additive):** an early-poll trigger from a wired pr-watch hook (documented recipe becomes a command), team-level request discovery if forgectl gains a listing, approving an already posted draft through review-watch (a human runs `pr-comment approve` until then), a revert detector filing `gatectl outcome --signal revert`, a dollar spend cap once review-run reports `cost_usd` (0.x has the daily round limit), per-author rules, notification templates, a `doctor`-style readiness command, and the 1.x tracking issues in the issue list.
- **2.0:** nothing planned; interactive review mode and a learned risk score are not designed.
