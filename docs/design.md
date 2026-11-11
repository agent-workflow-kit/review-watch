# review-watch: design

Status: designed, 2026-11-11 (interface 0.1.0-design). The rules are in [interface.md](interface.md), the single normative source; this document records the decisions, the structure and the test strategy and links to the headings there instead of restating them. Section references (§N) point to `../awkit-docs/architecture.md`, and "design guide" to `../awkit-docs/design-guide.md`.

## Role and layer

Layer 3 (workflow). Finds the PRs that ask you to review them by name, runs a review on each through review-run and holds the result as a draft you accept. Works fully standalone (architecture C2): it needs forgectl, review-run and pr-comment, and optionally gatectl and jobsctl. The harness and any other caller are unknown to it; behavior differs only by explicit arguments. It is shareable (architecture C4): no tmux and no notifier is needed, and macOS is supported.

## Uses

| Tool | Binding | What for |
|---|---|---|
| forgectl | A (hard) | `prs --review-requested-from me`, `snapshot`, `show`, `files`, `whoami` (read-only, `--json`); the minimum version is a constant of the release |
| review-run | A (hard) | `run`, `resume`, `cancel`, `list --label`, `show` (`--draft`, `--no-refresh`); the minimum version must support `--no-implicit-done` |
| pr-comment | A (hard) | `draft check`, `review` (`--dry-run`, `--reanchor-to`, `--expect-plan`, `--approve`) |
| gatectl | A (optional) | `check`, `outcome`, `stats`; without it nothing is ever automatic |
| jobsctl | A (optional) | `add`, `capacity`, `show --unique-key`, `resolve`; without it, detached processes under an internal cap |

pr-watch is not a dependency (D-9). Third parties: none directly; `$EDITOR` for `edit`. The package's own dependencies are `pyyaml` (the stdlib cannot read YAML), `pydantic` (config validation) and `textual` (the optional `browse` TUI, imported only by that command), as the design guide prescribes; everything else is the standard library (`argparse`, `sqlite3`, `subprocess`, `json`, `fnmatch`, `os`).

## Decisions

| ID | Decision | Rationale | Interface |
|---|---|---|---|
| D-1 | Two objects: an item per review request, keyed by `(pr_id, reviewer, instance)`, and a round per review at one head; the head sha belongs to the round | A new head under one instance must not start a review (§5.3), yet drafts, runs and staleness are per head; history is kept without overwriting drafts | [Identifiers](interface.md#identifiers), [Round states](interface.md#round-states) |
| D-2 | A round counts toward the per-PR cap once its session was launched; failures before the launch are free; the cap binds automatic starts and `rerun`; only a human `--force` or a config change passes it | Matches the architecture wording and review-respond's `max_rounds`; a veto or a missing tool must not use up rounds | [Creating a round](interface.md#creating-a-round) |
| D-3 | Hosts are named in config (no default); optional repo globs narrow it; requests out of scope are not recorded | Cost and exposure are bounded by the hosts the user chose; the review itself is sandboxed by review-run | [Scope](interface.md#scope) |
| D-4 | The tier is the maximum over independent deterministic rules; unknown facts give tier 3; model output can only raise it; it only feeds the gate input and two built-in limits | Explainable, golden-testable, safe by default (§5.3: a null size is the largest tier) | [Risk tiers](interface.md#risk-tiers) |
| D-5 | `poll` re-reads each non-terminal round's PR (one `show`) and marks a drafted round stale on a head or base change; `accept` always re-reads live and pins head and base | One cheap read per open round; the live check is the safety net, the flag is only information | [Staleness](interface.md#staleness) |
| D-6 | One gate call per round per policy, with a closed declared-field input; only `enforced: true` with `allow` acts; outcomes are filed from the human's accept or reject | §12.3 and the gatectl carry-forwards; an automatic `agree` never counts | [Automatic actions](interface.md#automatic-actions), [Gate outcomes](interface.md#gate-outcomes) |
| D-7 | The job is `review-run run` itself, with no wrapper; `poll` (and `show`, `accept`) is the only finalizer, under a per-round lease; `review-run list --label` finds an existing run before any submission | A dead job or `poll` loses nothing: review-run's own record is the truth; review-run does not dedupe `run` | [Launch](interface.md#launch), [Finalize](interface.md#finalize) |
| D-8 | An `escalated` round state with `resume` and `reject` as human actions; held jobs are resolved by `poll`; no automatic resume | §10.1 step 9; escalation always goes to the human | [`resume`](interface.md#resume), [Finalize](interface.md#finalize) |
| D-9 | pr-watch is dropped from the Uses list | Its only query watch is `--mine --open`, it emits no event for a review request, and `poll` re-reads drafted rounds itself; a user may wire its hooks to `review-watch poll` | [Detection and the poll cycle](interface.md#detection-and-the-poll-cycle) |
| D-10 | Skipped requests are stored and re-evaluated each poll; team-level requests are never seen; out-of-scope requests are counted only | `list` and `status` can explain why a PR got no review; a draft PR made ready starts without a new request | [Skip rules](interface.md#skip-rules) |
| D-11 | `edit` keeps revisions with hashes and the `edited` flag; `accept` shows pr-comment's exact plan and binds the post to its `plan_sha256`; no pending store | The `edited` flag decides the gate outcome; the exact-plan guarantee comes from pr-comment; without a pending store a non-human `accept` exits 13 and records nothing | [`edit`](interface.md#edit), [`accept`](interface.md#accept) |
| D-12 | Hooks: `pre-round` (veto or escalate), `post-draft`, `post-attention` (one hook, a `reason`), `post-post`; no `post-pending` | Design guide section 8 reserves `post-pending` for a tool with a pending store; one attention hook keeps wiring to a notifier simple | [Hooks](interface.md#hooks) |
| D-13 | `poll` is isolated per host, item and round; exit 0 for a completed cycle even with errors, 12 only when the cycle could not run or every host failed; a persistent host failure raises `post-attention` | Cron should not flood on transient errors; absence in a failed listing means nothing | [Detection and the poll cycle](interface.md#detection-and-the-poll-cycle) |
| D-14 | State in `state.db` (drafts as blobs with hashes), a public `log.jsonl`, an `audit.jsonl` of external actions and human decisions; no PR title or body is stored | The audit covers the decisions that feed the gate's track record; untrusted text is not kept | [Audit and log](interface.md#audit-and-log), [Files](interface.md#files) |
| D-15 | No `reconcile` command: `poll` repairs state against reality | A second repair path would only duplicate it (pr-watch pass 2 reached the same result) | [Commands](interface.md#commands) |
| D-16 | The spend cap needs `cost_usd` on review-run's run object (an additive field, requested as an issue); until then it cannot trigger and `status` says so | review-run exposes no cost today; reading it through agent-spawn or claude-session would add a hidden dependency across layers | [Spend cap](interface.md#spend-cap) |
| D-17 | Safety settings (tiers, caps) in a per-repo override may only be stricter than the global value; off-limits and sensitive lists add and never remove | A repo entry must not weaken a global safety setting silently (design guide section 5 precedence puts repo above env on purpose) | [Config](interface.md#config) |

## Resolved open topics

The six open topics of the seed and architecture §20 are resolved as follows: the **state machine** in [Items, rounds and states](interface.md#items-rounds-and-states) (D-1); **risk tier rules** in [Risk tiers](interface.md#risk-tiers) (D-4); **rounds** in [Creating a round](interface.md#creating-a-round) (D-2); **staleness** in [Staleness](interface.md#staleness) (D-5); **shadow mode and track records** in [Automatic actions](interface.md#automatic-actions) and `status` (D-6); and the **config shape** in [Config](interface.md#config) (D-3, D-17).

## Not in 0.1

Candidates for 1.x, each additive: an early-poll trigger when a pr-watch hook is wired (a documented recipe is all 0.1 has); team-level request discovery (forgectl has no listing, and a per-repo scan would be costly); `approve` of an already posted draft through review-watch; a revert detector feeding `gatectl outcome --signal revert`; per-author rules; interactive review mode; notification text templates; a learned risk score (the prior art describes one; this tool does not implement it); a `doctor`-style readiness command (review-run's `run --dry-run` serves for now).

## Live verification

No third-party behavior is wrapped directly: git-host behavior is forgectl's, and the session is review-run's. The behaviors review-watch relies on were verified in those tools' sessions (forgectl spikes A to F, pr-watch spikes A to C, review-run spikes A to C, gatectl spikes A, A2 and B). No new spike was run in this session (the user agreed); every sibling is still unimplemented, so there is nothing real to run against. The behaviors below become `live` tests, and the unverifiable ones keep a conservative default.

| Behavior | Source | Test or default |
|---|---|---|
| `prs --review-requested-from me` lists direct requests only; GitLab also lists finished requests | forgectl interface | live test on both hosts; every candidate is snapshotted (default) |
| A re-request is a new `instance`; a push is not; a null instance starts once | forgectl CF-2 | live test on GitHub and GitLab; null keeps the one-start rule |
| `review-run run` exits 13 in two shapes; an escalated run is continued by `resume` | review-run interface | live test with a real escalation |
| A jobsctl job whose command exits 13 is `held` and `resolve` ends it | jobsctl interface | live test |
| `gatectl check` with a declared head returns `enforced`, `decision` and a `decision_id`; `show` returns the declared head | gatectl interface | live test; a missing id means no automatic action |
| `pr-comment review --dry-run` then `--expect-plan` posts exactly the previewed plan | pr-comment interface | live test on both hosts |
| Cron block parsing with and without the end line | pr-watch and gatectl precedent | golden test; macOS checklist |
| The editor round trip (`$EDITOR`, `$VISUAL`) in a TTY and in the TUI | this tool | process test with a fake editor; macOS checklist |

## Modules

| Module | Responsibility |
|---|---|
| `cli` | `argparse` subcommands, exit codes, `--json` printing, the error envelope |
| `config` | Pydantic models, drop-ins, precedence, per-repo overrides, the stricter-only rule |
| `store` | SQLite access, migrations, leases, compare-and-set transitions |
| `discover` | host listing, snapshots, scope, identity, skip rules, item matching |
| `tiers` | deterministic risk tiers and the hard rules |
| `rounds` | the state machine, caps, spend, retries, staleness |
| `launch` | jobsctl submission or detached processes, capacity |
| `finalize` | reading review-run, copying drafts, resolving held jobs, cancel |
| `gate` | gatectl calls, the declared input, outcomes |
| `post` | edit revisions, accept, re-anchoring, pr-comment calls |
| `hooks` | hook execution, payloads, outbox delivery |
| `audit`, `log` | `audit.jsonl` and `log.jsonl` writers |
| `sanitize` | the one output sanitizer |
| `tui` | the optional Textual browser |
| `gateway` | every subprocess call (forgectl, review-run, pr-comment, gatectl, jobsctl, the editor, cron), so tests can substitute fakes |

## Data model

Tables in `state.db` (additive migrations, schema version recorded): `items` (id, `pr_id`, `pr`, `repo`, reviewer id, `instance`, state, `skip_reason`, listed and snapshot `updated_at`, seen times), `rounds` (id, item, `n`, state and reason fields, `rev`, head and base, tier and reasons, run ids, job or process id and start time, attempt, gate records, flags, failure, post result, timestamps), `revisions` (round, number, author, sha256, blob), `runs` (round, run id, attempt, state), `events` (the per-item log mirror and the hook outbox with `event_id`), `leases` (`poll`, per-round finalize), `hosts` (consecutive failure count, last error, last alert). The public contracts are `log.jsonl` and `audit.jsonl`; `state.db` is private.

## Concurrency

`poll`, `reconcile`-style repair inside it, and finalize take leases (holder PID plus process start time plus expiry; a dead holder's lease may be taken over). Every state change is a compare-and-set on the round's `rev`, so `poll`, `show`, `accept`, `reject` and the TUI cannot both apply a transition. Hook delivery is an outbox written in the transition's transaction and delivered by the committing process; an undelivered event is redelivered by the next `poll`. A detached process is identified by PID plus start time and never signaled by pattern. `poll` signals nothing: cancellation goes through `review-run cancel` and `jobsctl cancel`, never a signal.

## Error handling

A failure on one object never stops another; nothing is changed because a read failed (Detection and the poll cycle). Lower tools' exits are mapped, not hidden: refusals pass through with `lower_tool`, `lower_code` and `lower_reason`; a vetoed or escalated `pr-comment` step surfaces as exit 11 or 13 with `escalated_by`. A malformed lower-tool output is `lower_contract_mismatch` (exit 12). An unexpected exception is exit 1 with a log record and never leaves a round in a state it cannot leave: a `posting` round is retried by running `accept` again.

## Test strategy

All subprocess calls go through `gateway.py`; fakes for forgectl, review-run, pr-comment, gatectl and jobsctl live in `tests/fakebin/` (copied per repo, never shared) and are driven by pinned copies of the siblings' schemas in `tests/schemas/`. Golden files cover every `--json` output and hook payload, validated against `schemas/`. Process tests run real detached processes and cron blocks against the fakes. `live` tests (deselected by default, never in CI) cover the real round trip on both hosts, request-instance semantics, a real escalation, a held job, the gate flow in shadow, the editor round trip and the cron block; a macOS checklist covers the rest. Property tests cover the state machine (every transition is a compare-and-set and no terminal state changes) and the tier function (an unknown fact never lowers a tier). `make verify` runs ruff, pyright, pytest and markdown lint.

## Carry-forwards

Applied from the earlier sessions, in full in the private resolution lists: forgectl CF-2 (the key, null instance, GitLab finished requests, `is_bot: null`, null size, `author_association`, `details.head_sha`, `blocked_by` in the hard rule, ids over logins); pr-comment (the dry-run and `--expect-plan` flow, `--reanchor-to`, `reanchored_draft`, marker detection, the auto-approve guard); gatectl (declared head, `enforced` and `allow`, human outcomes only); review-run (the `meta.review_run` consumer rule, both exit-13 shapes, `--previous`, `cancel`, `resume`, the `rw-item` label); jobsctl (unique key, held jobs, `capacity`, `pass_env_undefined`, `kill_grace`). New requests for other tools are listed in the session's cross-tool notes.
