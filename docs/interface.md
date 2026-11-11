# review-watch: interface

Status: designed, 2026-11-11 (interface 0.1.0-design). This document is the single normative source for review-watch's rules. `design.md` records the decisions and the internal structure and links to the headings here instead of restating them; the README, the architecture and the schemas follow it. Machine-readable shapes are in `schemas/`.

## Concepts

review-watch finds the pull requests (GitHub) and merge requests (GitLab) that ask you to review them, runs a review on each through review-run, and holds the result as a **draft** that you edit and accept. Accepting posts the review through pr-comment. Nothing is posted without a human acceptance unless an enforced gatectl policy allows it and the built-in hard rules pass. `poll` is the only command that looks for new requests; it never waits for a review.

### Identifiers

- **Repo id** and **PR reference:** as in the design guide, section 4.1. `host/<full project path>` for a repo; a PR is a URL or `host/<full project path>#N` (GitLab numbers written without `!`); output always uses `#N`. The canonical `pr` and `pr_id` come from forgectl (`pr_id` is stable across repository renames).
- **Item:** one review request. An item is keyed by `(pr_id, reviewer, instance)`, where the reviewer is the user's own account (compared by id) and `instance` is forgectl's `review_requests[].instance`. A re-request is a new instance and so a new item. A push does not change the instance, so a new head under one instance is the same item and never starts a review by itself. The first sight of `(pr_id, reviewer)` with `instance: null` starts once; later sightings of null are the same item.
- **Round:** one review of an item at one head sha. A round has its own run, draft and state. Rounds are numbered per item from 1.
- **Item id:** `rw-` followed by 8 lowercase base32 characters (`a-z`, `2-7`). A unique prefix of at least 4 characters after `rw-` is accepted (shorter exits 2, ambiguous exits 21, unknown exits 10); output always shows the full id. **Round id:** `<item id>#<n>`. A command that takes an item id accepts a round id where noted, and otherwise acts on the item's latest round.
- **Labels passed to review-run:** `rw-item=<item id>` and `rw-round=<n>`, plus the jobsctl job's own labels (Launch).
- **Time:** every stored and printed timestamp is UTC as `YYYY-MM-DDTHH:MM:SS.ffffffZ`. Durations take `0` or an integer and a unit: `ms`, `s`, `m`, `h`, `d`.
- **Actor:** `^(human|auto):[A-Za-z0-9._@-]{1,64}$`. A command that takes `--actor` defaults to `human:<me.name>` on a TTY and `auto:unknown` otherwise. Human-only actions are `accept`, `edit`, `resume`, `rerun --force` and `--override-escalation`; they need a TTY or an explicit `human:` actor.

### Untrusted text

Everything that comes from a host (titles, bodies, branch names, paths, comments) and everything a model wrote (the draft's text, `meta.review_run.summary`, `risk_reasons`) is untrusted. review-watch stores PR ids, shas, counts, enum values and the draft itself; it stores no PR title or body. It never puts untrusted text in argv or env, and never in a hook payload. Human output prints every untrusted string through one sanitizer: control characters and bidirectional marks become visible `<U+XXXX>` marks and the text is capped at 256 characters per string (the draft body view in `show --draft` and the TUI is capped per field, not per line). `--json` output carries untrusted strings verbatim, marked `x-untrusted` in the schemas.

## Items, rounds and states

### Item states

- `open`: eligible, with a round in progress, waiting or ended in `failed`, `stale` or `rejected` and no newer round.
- `skipped`: a direct request that fails a skip rule (Skip rules). It carries `skip_reason`.
- `posted`: its latest round is `posted` and no round is in progress.
- `rejected`: a human rejected it (`reject <item id>`); no round starts automatically. `rerun` reopens it.
- `closed`: the PR is merged or closed. Rounds that were not terminal become `closed`.

### Round states

`queued → running → drafted → posting → posted`, plus `escalated`, `failed`, `stale`, `rejected` and `closed`.

| State | Meaning | Terminal |
|---|---|---|
| `queued` | recorded, no run yet; `queue_reason` says why (`waiting_slot`, `spend_cap`, `held`, `submit_failed`, `retry_wait`) | no |
| `running` | a job or detached process is running (or about to run) `review-run run`; `phase` mirrors review-run's when known | no |
| `escalated` | review-run's run is `escalated`: the session asked a human, its agent is kept, a held job waits | no |
| `drafted` | the run is `done`; the draft is stored and waits for a human or a gate | no |
| `stale` | the draft's head or base is no longer the PR's; `stale_reason` is `stale_head` or `retargeted` | no |
| `posting` | `accept` or an automatic post has started and has not been confirmed done | no |
| `posted` | pr-comment posted the review (the result is stored) | yes |
| `failed` | the run failed or could not be started after its attempts; `failure` carries review-run's reason and `retryable` | yes |
| `rejected` | a human rejected the round | yes |
| `closed` | the PR merged or closed before the round ended, or the round was superseded (`closed_reason`: `pr_closed`, `pr_merged`, `superseded`) | yes |

### Transitions

- `queued → running`: a job is added (jobsctl) or a detached process is started. `queued` stays `queued` when a cap, the spend cap or a `pre-round` hold applies.
- `running → drafted`: `finalize` read the run as `done` and stored the draft. `running → failed`: the run is `failed` (review-run's `reason` and exit code), or its job ended without a run record after `max_attempts`. `running → escalated`: the run is `escalated`. `escalated → running`: `resume`. `escalated → failed`: the run expired or was canceled.
- `drafted → stale` when the head or base moved. `drafted → posting → posted` through `accept` or an automatic post. `stale → posting` only through `accept --reanchor` for `stale_head`. A `retargeted` round cannot be re-anchored.
- `drafted | stale | escalated | queued | running → rejected` through `reject`. Any non-terminal state `→ closed` when the PR is merged or closed. A stale round becomes `closed` (`superseded`) when `rerun` starts a newer round.
- `posting → posted` once pr-comment reports the post done (including `already_posted`); a failure leaves the round in `posting` with `post_error`, and running the same `accept` again is the answer to "was it posted?".
- Terminal states are immutable. Every transition is a compare-and-set on the round's stored state and `rev`, so two processes cannot both apply it.

### Skip rules

A direct request in scope (Scope) that fails a rule is stored as an item in state `skipped` with a `skip_reason`:

| `skip_reason` | Rule | Cleared by |
|---|---|---|
| `draft` | the PR is a draft | the PR becomes ready |
| `bot_author` | `author.is_bot` is true | never |
| `bot_unknown` | `author.is_bot` is null (counts as a bot) | never |
| `own_pr` | the author's account id equals your own (forgectl `whoami`) | never |
| `already_reviewed` | you reviewed it yourself (below) | the condition no longer holds |

- Team-level requests are not requests for you by name: a PR whose only request is to a team is not listed by `forgectl prs --review-requested-from me`, so it is never seen. When a snapshot shows your direct request next to a team request, only the direct one counts.
- **Already reviewed, GitHub:** a `reviews[]` entry by your account (compared by id) that is newer than `review_requests[].requested_at` (a null `requested_at` keeps the skip) and does not carry the marker as its last line. A review you posted through awkit (pr-comment's Marker detection rule, `docs/interface.md` section Marker detection) ends the item as `posted`, not `skipped`. A re-request after your review is a new instance and starts again.
- **Already reviewed, GitLab:** `review_requests[].state` is `reviewed` (your own notes that carry the marker tell a review posted through awkit from one done by hand, as above). A comment-only review leaves the request `outstanding`; an unpublished draft reads `outstanding`. Because `prs --review-requested-from` also lists finished GitLab requests, every candidate needs a `snapshot` before it is judged.
- `list` hides `skipped` items unless `--all` is given. `status` shows the count per `skip_reason`.
- A skipped item is re-evaluated on each poll from a fresh snapshot of the candidate (it is listed again only while the request is outstanding). A condition cleared makes the item `open`, and the first round is created if the round cap allows.

### Scope

- Config `hosts` (required, no default) names the hosts to poll; `include` and `exclude` are lists of repo-id globs (`*` matches within a path segment, `**` across segments; `exclude` wins; default `include: ["*"]`). A request outside the scope is not recorded; `status` counts how many were seen in the last poll.
- `me` per design guide section 5 (`me.github`, `me.gitlab`, else the authenticated user through forgectl).

### Risk tiers

The tier of a round is the maximum over independent rules, computed from the forgectl snapshot, `files` and the config. An unknown fact gives tier 3.

| Tier | Rules (any one) |
|---|---|
| 3 | a changed path matches `off_limits`; size is unknown (`additions` or `deletions` null and the file list incomplete or without counts, or `changed_files` null); `author_association` is neither a member nor a collaborator (GitHub: not `OWNER`, `MEMBER` or `COLLABORATOR`; null on GitLab counts as unknown and so tier 3 unless config `tiers.trusted_authors` names the author); `ci.state` is not `passing`, or `blocked_by` holds a `checks_*` or `other` code; the base is not the repo's default branch (`base.is_default` false or null); the head repo is a fork and `tiers.allow_forks` is false (default true); any rule input is missing |
| 2 | more than `tiers.lines[1]` changed lines (additions plus deletions, default 300), more than `tiers.files[1]` files (default 15), or a path matching `tiers.sensitive_globs` (default none) |
| 1 | more than `tiers.lines[0]` lines (default 50) or more than `tiers.files[0]` files (default 3) |
| 0 | otherwise |

- Lines come from the PR's `additions` and `deletions`; when null (GitLab) they are the sum over `files` when `files_complete` is true and every file has counts, else unknown.
- **Model output can only raise the tier:** `meta.review_run.risk` `high` makes tier 3, `medium` at least 2; `low` changes nothing. `author_association` can only make a tier stricter.
- The tier is stored on the round with its reasons (`tier_reasons`, a closed list of rule codes). It is computed when the round is created, and recomputed from a fresh snapshot when the round reaches `drafted` if an automatic action is enabled for the repo; the gate sees the higher of the two.
- The tier decides only two things: the declared `risk_tier` input of a gate policy, and the built-in limits `max_post_tier` (default 1) and `max_approve_tier` (default 0) beyond which an automatic action is not even considered.

### Hard rules

Applied before any automatic action, whatever a gate says. Any failing rule means the draft waits for a human, and `show` lists the failing rule codes in `auto_blocked_by`:

- `own_pr`, `bot_author` (including unknown): never automatic.
- `ci_not_passing`: forgectl `ci.state` is not `passing` on the PR's current head at the time of the check, or `blocked_by` holds a `checks_*` or `other` code.
- `off_limits`: the PR touches a configured off-limits path. Default list (`off_limits`): CI and build pipeline definitions (`.github/**`, `.gitlab-ci.yml`, `.gitlab/**`, `.circleci/**`, `Jenkinsfile`, `azure-pipelines.yml`, `**/Dockerfile*`, `Makefile`), agent configuration and instruction files (`.claude/**`, `.mcp.json`, `**/CLAUDE*.md`, `**/AGENTS.md`), review-ownership files (`CODEOWNERS`, `.github/CODEOWNERS`, `docs/CODEOWNERS`), and security-sensitive code and secrets (`**/auth/**`, `**/security/**`, `**/secrets/**`, `**/crypto/**`, `**/*.pem`, `**/*.key`, `.env`, `.env.*`). A per-repo `off_limits` list adds to the global list and never removes from it.
- `tier_too_high`: the tier is above `max_post_tier` (post) or `max_approve_tier` (approve).
- `meta_signal`: `meta.review_run` shows `coverage.status: partial`, `code_run.status: skipped` when output was expected (a repo with `run_pr_code` configured is not visible to review-watch; the field `code_run.status` is `skipped` only when review-run expected output), `head_moved: true`, `session.sandbox` other than `required`, `session.tools_checked: false`, or a non-zero `session.tools_unexpected`.
- `head_moved_since_draft`: the head or base is not the draft's anchor (staleness).
- `reanchored`: the draft was re-anchored (a re-anchored draft is never auto-approved; it is not auto-posted either).
- `findings_block_approve` (approve only): `meta.review_run.verdict` is not `approve`, or `severity_counts` has any `blocker` or `major`.
- A rule that cannot be evaluated because a fact is unknown fails closed.

## Detection and the poll cycle

`poll` is the only command that looks for review requests and the only one that submits reviews. A cycle:

1. Take the poll lease (design guide section 4.3). A second `poll` exits 0 with `already_running: true`.
2. Preflight: config valid, `forgectl` found at a compatible version (`tool_missing` or `tool_too_old`, exit 12), the state database usable.
3. Identity: `forgectl whoami --host <h> --json` once per host per cycle (your account id and login), unless `hosts.<h>.me` pins the login; account ids come from whoami.
4. Per host: `forgectl prs --host <h> --review-requested-from me --state open --limit 1000 --json`. Drafts are included (so they can be recorded as skipped). `complete: false` or `errors` for the host is recorded; nothing is inferred from an absence in an incomplete listing.
5. Per candidate in scope: snapshot it (`forgectl snapshot <pr> --json`) when it is new, is `skipped`, or its listed `updated_at` is later than the one stored for it; an unchanged `open` or `posted` item is not snapshotted. At most `poll.max_snapshots` snapshots per host per cycle (default 30); the rest wait for the next cycle. The snapshot gives the `instance`, request state, draft, bot, size, ci, `blocked_by` and `author_association`.
6. Match the item (Identifiers). A new key creates an item; the skip rules decide `open` or `skipped`.
7. For every item with a round in `queued`, `running`, `escalated`, `drafted`, `stale` or `posting`: `forgectl show <pr> --json` once (head, base, state). A merged or closed PR closes the item and its non-terminal rounds (`closed`); a head change marks a `drafted` round `stale` (`stale_head`), a base ref change `retargeted`.
8. Finalize: for every `running` or `escalated` round, read its run (Finalize).
9. Start: create a round for each `open` item with no round when the cap allows; submit queued rounds in `created_at` order while capacity remains (Launch).
10. Gates for rounds that reached `drafted` in this cycle (Automatic actions).
11. Fire hooks, append the cycle record to `log.jsonl`, release the lease.

- **Pacing:** `rate_limited` from forgectl pauses that host for the rest of the cycle and honors `retry_after`; review-watch has no limiter of its own beyond `poll.max_snapshots`. `forgectl` reads retry inside it.
- **Failures:** a failure on one PR, host or round never stops the others, each of which runs in its own transaction. A host failure (`auth_required`, `permission_denied`, `rate_limited`, `upstream_error`, `network_error`, an unknown or unsupported host) skips the host for the cycle and leaves its items untouched. A failed per-PR read leaves that item as it was. Nothing is marked stale, closed, skipped or expired because a listing or a read failed.
- **Repeated failures:** each host keeps a count of consecutive cycles with an error. When it reaches `poll.failure_alert_after` (default 3), `post-attention` fires once with reason `poll_failed`; it fires again only after a successful cycle.
- **Exit codes of `poll`:** 0 when the cycle completed, also when some hosts or items failed (the result lists them in `errors[]`); 12 when it could not run (invalid config at run time, `tool_missing`, `tool_too_old`, `state_unsafe`, or every configured host failed with `reason: all_hosts_failed`); 2 for usage.
- `poll --no-start` discovers, finalizes and checks staleness but submits nothing. `poll --repo <id>` limits the cycle to one repo.

## Rounds

### Creating a round

A round is created when an `open` item has no round (its first), when a human runs `rerun`, or when a fresh request instance creates a new item. Creation checks, in order, and a failed check leaves the round uncreated and reports it (automatic starts record `skipped`-style reasons in the log and `post-attention`; `rerun` exits 20):

1. **Round cap:** `max_rounds` per PR across instances, default 3 (`repos.<id>.max_rounds`). A round counts once its review session was launched (the review-run run reached phase `session`). Rounds that failed before the session (a `pre-run` veto, `tool_missing`, `tool_too_old`, an unusable host) do not count. At the cap an automatic start is not made (`round_limit`, `post-attention` once) and `rerun` exits 20 `reason: round_limit`; a human passes `rerun --force` or raises `max_rounds`; automation never does.
2. **Spend cap** (Spend cap): at the cap the round is created `queued` with `queue_reason: spend_cap`.
3. **`pre-round` hooks** (Hooks): a veto records the round `rejected`-free as not created (`hook_veto` in the log); an escalation creates it `queued` with `queue_reason: held`.
4. **Capacity** (Launch): without capacity the round is `queued` with `queue_reason: waiting_slot`.

### Launch

- **With jobsctl** (`jobs.use_jobsctl: auto` finds a `jobsctl` binary; `true` requires it; `false` never uses it): `jobsctl add --queue <jobs.queue> --unique-key review:<pr>@<head sha> --label owner=review-watch --label rw-item=<id> --label rw-round=<n> --label pr=<pr> --json [--pass-env VAR]... -- <absolute review-run> run <pr> --mode headless --label rw-item=<id> --label rw-round=<n> [--previous <run>]`. `--pass-env` names come from `jobs.pass_env` (default empty). The job's argv holds only ids and shas, never untrusted text. The result's `warnings` are read: `pass_env_undefined` is a failed submission (the round stays `queued`, `queue_reason: submit_failed`, and `post-attention` once), and `existing_job_held` links to the held job. `created: false` means an earlier job holds the key (its result is adopted). Before submitting, `jobsctl capacity --queue <jobs.queue>` `effective_free` must be above 0 (null means unbounded).
- **Without jobsctl:** a detached process (`setsid`, stdio to a private log file, working directory an empty private directory) runs the same `review-run run` command. Its pid and start time are stored; at most `max_concurrent` (default 2) such processes run at once; extra rounds wait `queued` (`waiting_slot`). Liveness is checked by pid plus start time, never by pattern.
- There is no wrapper command and the job is review-run itself: a dead job, process or `poll` leaves review-run's own run record as the truth.
- `--previous <run>` is passed for round 2 and later when `review.previous` is true and the previous round has a `done` run (any head).
- review-run does not dedupe `run`: review-watch finds an existing run for a round with `review-run list --label rw-item=<id> --label rw-round=<n> --json` before it submits, and never submits a second job for a round that has a non-failed run.

### Finalize

For each `running` or `escalated` round, `poll`, `show` and `accept` run one idempotent finalize step under the round's own lease:

1. Find the run: the stored run id, else `review-run list --label rw-item=<id> --label rw-round=<n>`.
2. No run found: if the job (jobsctl `show --unique-key`) or the detached process is gone and no run was ever recorded, count a failed attempt; when `attempt` reaches `max_attempts` (default 2) the round is `failed` (`failure.reason: no_run`), otherwise it returns to `queued` (`retry_wait`).
3. Run `running`: nothing to do (review-run finalizes its own runs; `poll` reads with `review-run show --no-refresh`, and uses plain `show` only for a run whose runner is gone).
4. Run `done`: read `review-run show <run> --draft` (the stored `review.json`), verify `draft_id`, `pr` and `anchor.head_sha` against the run object, validate it with `pr-comment draft check -`, store it as revision 1 with its sha256, copy the run's `summary` fields (verdict, severity counts, risk, coverage, `head_moved`) onto the round, set the state `drafted` and queue `post-draft`. A run whose draft fails these checks makes the round `failed` (`draft_invalid`).
5. Run `escalated`: the round becomes `escalated`; `post-attention` fires once with reason `escalated`. A held job is left for `resume` or `reject`.
6. Run `failed`: the round becomes `failed` with `failure` copied from the run (`reason`, `retryable`, `retry_after_s`, `exit_code`). When `retryable` is true and the round's attempts are below `max_attempts`, a new run is submitted for the same round after `retry_after_s` (the round returns to `queued`, `retry_wait`). `post-attention` fires once (`failed`) when the round ends `failed`, or `expired` when the run's reason is `expired`.
7. A held jobsctl job (exit 13) is resolved by `jobsctl resolve <job> --as done|failed|canceled` when the run ends or the round is rejected or closed. Rounds that end without a held job need nothing. `jobsctl` refusal `not_held` is success.
8. A round that is `rejected` or `closed` with a `running` or `escalated` run gets `review-run cancel <run> --actor auto:review-watch` (or the human's actor for `reject`); a run already terminal is a no-op.

### Spend cap

`spend.daily_cap_usd` (default `null`: off). When set, the day's spend is the sum of `cost_usd` over review-run runs of this tool's rounds that ended on the current local date (`spend.timezone` defaults to the machine's local time zone). At or above the cap, new rounds stay `queued` with `queue_reason: spend_cap`, `post-attention` fires once per day with reason `spend_cap`, and runs already started finish. A human can `rerun --force`, which skips the check for that round. A run with a null or missing `cost_usd` counts as 0 and is shown as `unknown_cost` in `status`. This needs review-run to report `cost_usd` on its run object; until it does the cap never triggers and `status` says `cost_unavailable`.

### Staleness

- Each `poll` reads every non-terminal round's PR with one `forgectl show`. A head sha different from the draft's `anchor.head_sha` marks a `drafted` round `stale` (`stale_head`, with `stale_head_sha` and `stale_since`); a base ref different from `anchor.base_ref` marks it `retargeted`. `post-attention` fires once per round with reason `stale`.
- `accept` always re-reads the PR live and passes `--expected-head` and `--expected-base` to pr-comment, so a stale flag that is out of date can never cause a wrong post.
- On a stale round: `accept --reanchor` (`stale_head` only) re-anchors through pr-comment (below), `rerun` starts a newer round (the stale one becomes `closed`, `superseded`), `reject` ends it.
- A `retargeted` round cannot be re-anchored (pr-comment needs a new draft). `accept` exits 20 with `reason: retargeted`.

## Review drafts

- The draft is a pr-comment review draft. Revision 1 is the stored copy of review-run's artifact; it is the editable, authoritative copy for posting. It keeps its `draft_id`, so a draft that already posted is never posted again and a new review is a new draft.
- Revisions are kept in `state.db` with their sha256, size and author (`human:<name>` or `revision 1`). The round records `edited: true` once a human revision exists.
- **`edit`:** see Commands. Edits to `draft_id`, `anchor`, `pr` and `schema_version` are refused; `meta` is kept as stored; the result must pass `pr-comment draft check -`. Editing a `posting` or `posted` round is refused.
- **Re-anchoring:** `accept --reanchor` runs `pr-comment review <pr> --draft-file F --reanchor-to <current head> ...`. The result's `reanchored_draft` is stored as a new revision (findings pr-comment could not map are in its `unanchored`); the round's anchor head moves to the new head and the round records `reanchored: true`. A re-anchored round is never gated again and never auto-approved.

## Automatic actions

Defaults: all off. `auto_post.enabled` and `auto_approve.enabled` are per repo (`repos.<id>`), each with a `policy` (defaults `review-post` and `review-approve`).

1. When a round reaches `drafted` and an automatic action is enabled for its repo, the hard rules are evaluated. A failing rule stops here (the draft waits; `auto_blocked_by` lists the codes).
2. `gatectl check <policy> --repo <repo> --input - --timeout <gate.timeout>` runs once per round per policy. The input is a JSON object of declared fields only: `repo`, `object_type: "pr"`, `object_id` (the `pr`), `action` (`review` or `approve`), `head_sha`, `base_ref`, `risk_tier`, `tier_reasons`, `verdict`, `severity_counts`, `coverage_status`, `additions`, `deletions`, `changed_files`, `author_association`, `ci_state`, `round`, `edited`. The policy should list `repo` and `head_sha` in `input.require`. No PR text and no model text is in it. The hook timeout caller rule applies: when review-watch runs under a hook-style limit, `gate.timeout` must be at least 30 seconds below it.
3. Only a result with `"enforced": true` and `"decision": "allow"` triggers the action. Shadow, `off`, `not_applicable`, an exit 11 or 13 (deny or escalate), an exit 1 or 2 (state unknown or a config error: recorded as `gate_error`, no action), a missing gatectl, and a missing `decision_id` all leave the draft for a human. The decision id, the would-be decision and the `enforced` flag are stored on the round (`post_gate`, `approve_gate`).
4. The post gate runs first. On allow, review-watch runs `pr-comment review <pr> --draft-file F --dry-run --actor auto:<policy> --decision-id <id> --expected-head <head> --expected-base <base> --json`, reads `plan.plan_sha256`, then `pr-comment review ... --yes --expect-plan <sha> --actor auto:<policy> --decision-id <id>`. The round goes `posting` then `posted`.
5. The approve gate runs only after the post gate allowed, the approve hard rules pass and the draft was never re-anchored. On allow, the post in step 4 adds `--approve` and both decision ids. pr-comment itself requires `allow_auto_approve: true` for the repo in its own config. If only the post gate allowed, the review is posted without `--approve`.
6. To approve an already posted draft, a human uses `pr-comment approve` directly; review-watch never calls `review --approve` for a draft that was posted, and never approves from a `request_changes` draft (`approve_conflicts_event`).
7. A stale or re-anchored round is not gated again; a `rerun` creates a new round with a new gate call.
8. A revert detector is not review-watch's concern: a revert or incident is filed by whoever detects it with `gatectl outcome <id> disagree --signal revert`.

### Gate outcomes

When a human acts on a round that has a gate decision id, review-watch files `gatectl outcome <id> agree|disagree --actor human:<name> --signal human` once per decision id (`changed: false` is success). An automatic action files nothing, because an automatic `agree` never counts.

| Gate's decision | Human action | Outcome filed |
|---|---|---|
| `allow` (would-be) | `accept` of an unedited round | `agree` |
| `allow` | `accept` of an edited round, or `reject` | `disagree` |
| `deny` or `escalate` | `accept` of an unedited round | `disagree` (over-strict, counted only) |
| `deny` or `escalate` | `reject` | `agree` |

Outcomes are filed at `accept` and `reject`, using the round's `edited` flag at that time; `rerun`, a superseded round and a closed PR file nothing. When gatectl is not installed or the call fails, the failure is logged and the human action still proceeds.

## Commands

All commands take `--json`, `--config PATH`, `--verbose`, `--quiet`, `--actor`, `--decision-id` (repeatable), `--version`. Commands that change external state take `--dry-run`. `--yes` skips a confirmation that review-watch itself would ask for; it never overrides a refusal. Ids: an item id or, where noted, a round id.

### `poll`

`review-watch poll [--repo <id>] [--no-start] [--json]`

Runs the cycle in Detection and the poll cycle. Exit 0 or 12. `--json` prints the poll report: `{started_at, finished_at, hosts:[{host, listed, complete, snapshots, error}], discovered, new_items, skipped, out_of_scope, finalized, stale, closed, started, queued, errors:[{host, pr, code, message}], already_running}`.

### `list`

`review-watch list [--state S]... [--repo <id>] [--pr <pr>] [--all] [--limit N] [--json]`

Lists items with their latest round: `{items:[...], truncated}`. Default order: states needing a human first (`drafted`, `escalated`), then `stale`, `failed`, `running`, `queued`, `posted`, `rejected`, `closed`; then newest. `skipped` items appear only with `--all` or `--state skipped`. Default limit 100. Does not contact a host.

### `show`

`review-watch show <item id | round id> [--round N] [--draft] [--revision N] [--no-refresh] [--json]`

Prints the item and the selected round (the latest by default): states, tier and reasons, gate decisions, `auto_blocked_by`, run id and phase, staleness, revisions, failure, attempts, hook records and the commands now possible (`next_actions`). Without `--no-refresh` a `running` or `escalated` round is finalized first (idempotent). `--draft` prints the stored draft revision as pr-comment's JSON (untrusted, no envelope). Does not contact a host except through finalize.

### `edit`

`review-watch edit <round id | item id> [--draft-file <path>|-] [--base-revision N]`

Human-only. Without `--draft-file`, writes the latest revision to a private temporary file (mode 0600 in the state directory), opens `$EDITOR` (`$VISUAL` first), reads it back, deletes it, and stores a valid change as a new revision. With `--draft-file` the file or stdin is the new draft (stdin on a TTY exits 2). The result must pass `pr-comment draft check -` and keep `draft_id`, `anchor`, `pr` and `schema_version` unchanged. `--base-revision` refuses (exit 20 `revision_changed`) when the latest revision is not N. Exit 20 for a round that is not `drafted`, `stale` or `escalated`: `reason: not_editable`. An unchanged draft stores nothing (`changed: false`).

### `accept`

`review-watch accept <round id | item id> [--reanchor] [--dry-run] [--approve] [--revision N] [--override-escalation=<scope>] [--yes] [--expect-plan <sha256>]`

Human-only. Posts the round's draft through pr-comment.

1. The round must be `drafted` (or `stale` with `--reanchor`, or `posting` to finish an interrupted post); otherwise exit 20 (`not_drafted`, `stale_head`, `retargeted`, `escalated`).
2. Finalize first if needed. Read the PR live (`forgectl show`). A head different from `anchor.head_sha` without `--reanchor` exits 20 `stale_head` (`details.head_sha`); a different base exits 20 `retargeted`.
3. Run `pr-comment review <pr> --draft-file <revision> --dry-run [--reanchor-to <head>] [--approve] --actor human:<name> --expected-head <head> --expected-base <base> --json`. Print the plan (findings, moved and unanchored findings with reasons, neutralized text counts, the footer) and `plan_sha256`. `--dry-run` stops here, exit 0.
4. Ask once on a TTY (`Post? [y/N]`). Without a TTY, without `--yes`, exit 13 `confirmation_required`; nothing is recorded (there is no pending store: the caller re-runs after a human decision). A TUI confirms in its own dialog and passes `--yes`.
5. Mark the round `posting`, run `pr-comment review ... --yes --expect-plan <sha> --actor human:<name>`, store its result (`review_id`, `posted_inline`, `moved`, `landed`, `already_posted`, `plan_sha256`) and mark `posted`. A `reanchored_draft` in the result is stored as a new revision first.
6. pr-comment's exits pass through: 11 vetoed (round stays `drafted`), 13 `escalated` (`escalated_by` in the envelope; a human re-runs with `--override-escalation=<scope>`), 20 refusals (`stale_head`, `stale_base`, `plan_changed`, `auto_approve_disabled`, `not_open`, `self_approval`, `drafts_present`, `pending_review_exists`), 12 failures with their `landed` and `retry` fields. A failure after the write may have started leaves the round `posting`; running the same `accept` again is the check (pr-comment's `already_posted` is the answer) and never makes a new draft.
7. `--approve` passes through only for a `comment` draft (a `request_changes` draft exits 2 `approve_conflicts_event`).
8. After a successful post, file the gate outcomes (Gate outcomes) and fire `post-post`.

Exit codes: 0 posted (or `already_posted`), 2, 10, 11, 12, 13, 20 (`not_drafted`, `stale_head`, `retargeted`, `escalated`, `revision_changed`, `round_active`, plus pr-comment's reasons).

### `reject`

`review-watch reject <round id | item id> [--reason-file <path>|-] [--yes]`

Human-only. Ends the round (`rejected`) and, given an item id, rejects the item (state `rejected`, no automatic round). A live run is canceled with `review-run cancel`; a held job is resolved; open rounds of the item are `rejected`. Confirms on a TTY unless `--yes`. Files the gate outcome (Gate outcomes). `--reason-file` is stored (untrusted, capped at 1 KiB).

### `rerun`

`review-watch rerun <item id> [--force] [--dry-run]`

Human-only. Adds a round at the PR's current head to an item whose latest round is `posted`, `stale`, `failed`, `rejected` or `closed` with the PR still open (a `stale` round becomes `closed`, `superseded`). Exit 20 `round_active` while a round is `queued`, `running`, `escalated`, `drafted` or `posting`; exit 20 `round_limit` at the cap; exit 20 `spend_cap` at the spend cap. `--force` skips the round and spend caps and is human-only; `pre-round` hooks still run. `rerun` of a `rejected` item reopens it.

### `resume`

`review-watch resume <round id | item id> [--prompt-file <path>|-] [--yes]`

Human-only. For an `escalated` round, calls `review-run resume <run> --prompt-file F --actor human:<name>`; the answer is untrusted and travels in a file. For a round held by a `pre-run` hook escalation (it has no run), reruns `review-run run ... --override-escalation` with the round's original labels. Exit 20 `not_escalated`. The round returns to `running`; `poll` finalizes it. A run that escalates again stays `escalated`; there is never an automatic resume.

### `status`

`review-watch status [--json]`

Counts per item and round state; `skipped` per reason; today's runs and spend (`spend.today_usd`, `cap_usd`, `unknown_cost`, or `cost_unavailable`); rounds waiting for a slot or the spend cap; the last poll (`started_at`, `finished_at`, per-host error and consecutive failure count); out-of-scope requests seen; and, when gatectl is configured, `gatectl stats --policy <p> --repo <r> --json` for each enabled policy (`recommend`, `upper_bound`, decision and outcome counts), or `gatectl: not_configured`. Does not contact a host.

### `logs`

`review-watch logs <item id | round id> [--round N] [--run] [--limit N]`

Prints the item's events from `log.jsonl` (state changes, gate calls, hook results, errors); with `--run`, passes through `review-run show <run> --no-refresh` for each of the round's runs (sanitized). Human output only unless `--json`.

### `cron`

`review-watch cron install|uninstall [--every <duration>] [--schedule "<cron fields>"] [--dry-run] [--json]`

Writes or removes one managed block, following the golden-block convention of pr-watch and gatectl: line 1 `# review-watch managed v1`, line 2 `PATH=<dirs>` (the directories of the sibling tools), line 3 `<schedule> . '<env_file>' && '<absolute review-watch>' poll --quiet` (the `. '<env_file>' &&` part only with `cron.env_file`), last line `# review-watch managed end`. The default schedule is `*/5 * * * *` (`cron.schedule` or `--schedule` overrides it). A block starts at a line beginning `# review-watch managed` and ends at the end line, or, when the end line is absent (a block written by `awkit cron install`), at the next blank line or the next `# ... managed` line; the awkit adapter writes the same shape. Install is idempotent. Output `{action, dry_run, changed, block}`.

### `export`

`review-watch export [--log] [--since <time>] [--json]`

Prints `audit.jsonl` (or `log.jsonl` with `--log`) records as JSON lines, or one array with `--json`.

### `prune`

`review-watch prune [--older-than <duration>] [--dry-run] [--json]`

Deletes items whose state is `closed`, `rejected`, `posted` or `skipped` and whose last change is older than `retention.closed` (default 90 days), with their rounds, revisions and events. Never touches an item with a non-terminal round. Output `{older_than, results:[{item, result, reason}], warnings, dry_run}`. Exit 0.

### `browse`

See TUI.

## Global flags

`--json`, `--config <path>`, `--verbose`, `--quiet`, `--dry-run` (commands that change external state: `accept`, `reject`, `rerun`, `resume`, `edit`, `cron`, `prune`, `poll` (previews what it would start), `--no-start` behavior), `--yes`, `--actor`, `--decision-id` (repeatable, at most 16, `^[A-Za-z0-9._:@/-]{1,128}$`), `--hook <name>=<cmd>` (repeatable; `poll`, `accept`, `reject`, `rerun`, `resume`), `--version`. A flag given twice exits 2. Write-only flags follow the command.

## Exit codes

| Exit | Meaning |
|---|---|
| 0 | success (`poll` cycle completed, also with some errors; `accept` posted or `already_posted`) |
| 1 | internal error |
| 2 | usage (including `approve_conflicts_event` and `invalid_draft`) |
| 10 | not found (item, round, revision, draft) |
| 11 | vetoed (a `pre-round` hook, or pr-comment's or forgectl's veto passed through by `accept`) |
| 12 | failed for a reportable reason (`tool_missing`, `tool_too_old`, `all_hosts_failed`, a lower tool's failure, `state_unsafe`) |
| 13 | needs a human (`confirmation_required`, an escalation of a lower tool, `resume` or `accept` without a human) |
| 20 | refused, with a `reason` |
| 21 | ambiguous id prefix |

There are no tool-specific codes above 21. Refusal reasons (exit 20): `not_drafted`, `not_editable`, `not_escalated`, `round_active`, `round_limit`, `spend_cap`, `stale_head`, `retargeted`, `revision_changed`, `human_required`, `item_closed`, and the reasons passed through from pr-comment (`stale_base`, `plan_changed`, `auto_approve_disabled`, `auto_approve_reanchored`, `not_open`, `self_approval`, `drafts_present`, `pending_review_exists`, `not_resolvable`, `too_large`, `unknown_host`) and from review-run (`live`, `lower_refused`).

## Output by exit code

- `--json` prints at most one document on stdout. Exit 0 prints the command's result object. Any non-zero exit prints the error envelope, except `poll` exit 12 `all_hosts_failed`, which also prints the poll report, and `accept --dry-run` exit 20, which prints the plan beside the refusal.
- Errors go to stderr in human mode. Without `--json`, `poll` prints one line per notable event and a summary line.

### Error envelope

`{"error": {"code", "message", "reason", "details", "retry_after", "lower_tool", "lower_code", "lower_reason", "escalated_by", "landed", "retry", "decision_ids"}}`. `code` is one of `usage`, `config_invalid`, `not_found`, `ambiguous`, `vetoed`, `escalated`, `confirmation_required`, `refused`, `failed`, `tool_missing`, `tool_too_old`, `state_unsafe`, `internal`. Only `code` and `message` are required. `lower_tool`, `lower_code` and `lower_reason` identify a sibling's failure that passed through; `escalated_by` and `landed` and `retry` are copied from pr-comment's envelope unchanged.

### Result objects

Item, round, revision, gate record, `next_actions`, poll report, status and the per-command results are described in `schemas/cli/` and `schemas/defs.json`. A **round** carries: `id`, `item`, `n`, `state`, `queue_reason`, `head_sha`, `base_ref`, `tier`, `tier_reasons`, `run_id`, `run_ids`, `phase`, `attempt`, `job_id`, `draft_id`, `revision` (latest), `edited`, `reanchored`, `stale_reason`, `stale_since`, `escalated_since`, `failure`, `post_gate`, `approve_gate`, `auto_blocked_by`, `meta` (verdict, severity counts, risk, coverage status, `head_moved`, sandbox, `tools_checked`, `tools_unexpected`), `post` (the stored pr-comment result), `closed_reason`, `created_at`, `drafted_at`, `posted_at`. An **item** carries: `id`, `pr`, `pr_id`, `repo`, `instance`, `state`, `skip_reason`, `rounds` (count), `latest_round`, `first_seen_at`, `last_seen_at`.

## Hooks

Hooks follow design guide section 8: `sh -c` in their own process group, JSON on stdin, `REVIEW_WATCH_HOOK`, `REVIEW_WATCH_ID`, `REVIEW_WATCH_OBJECT_TYPE=round` in the environment, a 30-second default timeout per hook (configurable), no template substitution. Payloads (`schemas/hooks/<hook>.json`) carry `tool`, `hook`, `object_type: "round"`, `object_id` (the round id), `repo`, `labels` (`rw-item`, `rw-round`, `pr`) and the fields below. They never carry PR text, a title, a draft body, a summary or any model text. A tool that stores no pending confirmations has no `post-pending`.

- **`pre-round`**: after the scope, skip, cap and spend checks and before the job or process is created. Payload: `pr`, `item`, `n`, `head_sha`, `tier`, `tier_reasons`, `trigger` (`new_request`, `first_sight`, `rerun`, `skip_cleared`), `forced`. A hook may veto (exit other than 0 and 13: the round is not created, `hook_veto` in the log, exit 11 for `rerun`) or escalate (exit 13: the round is created `queued`, `queue_reason: held`, `post-attention` fires; a human continues with `rerun --override-escalation`, human-only, which runs the other pre-hooks again). A printed `decision_id` is recorded on the round. A timeout counts as a veto.
- **`post-draft`**: a round reached `drafted`. Payload: `pr`, `item`, `n`, `head_sha`, `draft_id`, `tier`, `verdict`, `severity_counts`, `coverage_status`, `auto_blocked_by`, `post_gate_decision` and `approve_gate_decision` (`allow|deny|escalate|null`) with their `enforced` flags.
- **`post-attention`**: something needs a human. Payload adds `reason` in `escalated`, `stale`, `failed`, `expired`, `round_limit`, `spend_cap`, `held`, `poll_failed` and `details` (`failure_reason`, `retryable`, `stale_reason`, `host`, `consecutive_failures`; never text). Fires once per cause (per round, or per host for `poll_failed`).
- **`post-post`**: a round reached `posted`. Payload: `actor`, `decision_ids`, `review_id`, `head_sha`, `automatic`, `reanchored`, `edited`.
- Post hooks run after the transition committed, detached, at most once, and their failures are logged and never fatal. `post-draft` and `post-attention` carry `event_id` (`<round id>-<hook>-<n>`) for deduplication.
- `--hook <name>=<cmd>` adds a hook for one invocation (timeout 30 s, async for post hooks).

## Config

Location and precedence follow design guide section 5 (`~/.config/review-watch/config.yaml`, drop-ins in `config.d/`, flags > `repos.<id>` > environment > global > defaults). Schema: `schemas/config.json`; every key with its default is in `config.example.yaml`. Invalid config fails fast with the key and the problem (exit 2 `config_invalid`, `details.key`).

| Key | Default | Notes |
|---|---|---|
| `hosts` | none (required) | hosts to poll; a missing or empty list exits 2 |
| `include`, `exclude` | `["*"]`, `[]` | repo-id globs; `exclude` wins |
| `me` | per section 5 | `name`, `github`, `gitlab` |
| `max_rounds` | 3 | per PR, across instances; per repo override |
| `max_concurrent` | 2 | only without jobsctl |
| `max_attempts` | 2 | runs per round |
| `jobs.use_jobsctl` | `auto` | `auto`, `true`, `false` |
| `jobs.queue` | `reviews` | jobsctl queue; needs `kill_grace` of at least 120 s (125 s with `run_pr_code`) |
| `jobs.pass_env` | `[]` | names for `--pass-env`; values come from jobsctl's env file |
| `review.skill` | `null` | review-run's default skill |
| `review.mode` | `headless` | |
| `review.previous` | `true` | pass `--previous` for round 2 and later |
| `tiers.lines` | `[50, 300]` | per-repo override may only be lower |
| `tiers.files` | `[3, 15]` | per-repo override may only be lower |
| `tiers.sensitive_globs` | `[]` | per-repo values add to the global list |
| `tiers.trusted_authors` | `[]` | author logins treated as members on hosts without associations |
| `tiers.allow_forks` | `true` | `false` makes a fork PR tier 3 |
| `off_limits` | the list in Hard rules | per-repo values add to the global list |
| `max_post_tier`, `max_approve_tier` | 1, 0 | per-repo override may only be lower |
| `auto_post.enabled`, `auto_post.policy` | `false`, `review-post` | per repo |
| `auto_approve.enabled`, `auto_approve.policy` | `false`, `review-approve` | per repo; pr-comment needs `allow_auto_approve` too |
| `gate.timeout` | `60s` | `gatectl check --timeout` |
| `spend.daily_cap_usd`, `spend.timezone` | `null`, local | optional |
| `poll.max_snapshots` | 30 | per host and cycle |
| `poll.failure_alert_after` | 3 | consecutive failing cycles per host |
| `retention.closed` | `90d` | `prune` |
| `cron.schedule`, `cron.env_file` | `*/5 * * * *`, `null` | |
| `forgectl_cmd`, `review_run_cmd`, `pr_comment_cmd`, `gatectl_cmd`, `jobsctl_cmd` | the tool names | binding A; the value is an argv list run without a shell, absolute paths recommended under cron |
| `hooks.<name>` | `{}` | lists of `{cmd, timeout, async}` |
| `repos.<id>` | `{}` | per-repo overrides of `include`/`exclude`, `max_rounds`, `tiers`, `max_post_tier`, `max_approve_tier`, `off_limits`, `auto_post`, `auto_approve`, `review.skill` |

A per-repo value for a safety setting (`tiers.lines`, `tiers.files`, `max_post_tier`, `max_approve_tier`) may be only stricter than the global one; a looser value is an invalid config. There is no `token_env`: credentials belong to forgectl and the lower tools.

## Environment variables

Scalar config keys read `REVIEW_WATCH_<KEY>` (dashes and nesting joined with `__`, for example `REVIEW_WATCH_SPEND__DAILY_CAP_USD`). Lists, `hooks`, `repos` and the tool commands cannot be set from the environment. Hook variables: `REVIEW_WATCH_HOOK`, `REVIEW_WATCH_ID`, `REVIEW_WATCH_OBJECT_TYPE`.

## Audit and log

- **`audit.jsonl`** (`schemas/audit.json`; public, append-only): one record pair, `attempted` then the outcome sharing an `attempt_id`, for each external action: an `accept` or an automatic post (target PR, `head_sha`, `draft_id`, `plan_sha256`, actor, decision ids, pr-comment's result with its review id), a `reject` that cancels a run or resolves a held job, a `rerun`, a `resume` (with `override_escalation`), and each `gatectl outcome` filed. Fields: `ts`, `actor`, `decision_id` (a list when several), `target`, `action`, `inputs` (hashes: head, `draft_id`, `draft_sha256`, config hash), `result`, `attempt_id`. `poll`, discovery, finalizing and state changes go to the log only. `awkit audit` pairs these records with pr-comment's and forgectl's by target, action and time.
- **`log.jsonl`** (`schemas/log.json`): state transitions (`item_state`, `round_state` with the reason), poll cycles, gate calls, hook results, finalize steps, warnings. No text from a host or a model.

## Files

State directory `~/.local/state/review-watch/` (respecting `XDG_STATE_HOME`): `state.db` (SQLite, WAL, private; tables `items`, `rounds`, `revisions`, `runs`, `events`, `leases`, `hosts`; additive migrations only), `audit.jsonl`, `log.jsonl`, `tmp/` (private files for editing and process logs, mode 0700, files 0600, emptied on use and by `prune`). The store carries a schema version; a newer one exits 12 `state_unsafe` with a hint, and it never downgrades.

## Labels

review-watch has no labels of its own. It passes `rw-item` and `rw-round` to review-run and to jobsctl, and `owner=review-watch` and `pr=<pr>` to jobsctl. No other tool may give those keys meaning.

## TUI

`review-watch browse [--focus <item id | round id>] [--result-file <path>] [--all]`. A Textual TUI over the CLI actions; without a TTY it exits 2. It runs as a foreground child of another TUI, takes no lease and refreshes with `list` and `show --no-refresh`.

- **Layout:** the left pane lists items sorted by need (`drafted` and `escalated`, then `stale`, `failed`, `running`, `queued`, the rest; `skipped` only with `--all` or the `a` key). The right pane shows the selected round: tier and reasons, verdict, severity counts, the gate decision, `auto_blocked_by`, staleness and the draft's findings by file, all through the sanitizer.
- **Keys:** `q` quit, `?` help, `/` filter, `enter` open a finding, `r` refresh, `a` accept (a dialog shows the `--dry-run` plan and `plan_sha256`, then runs `accept --yes --expect-plan`), `A` accept with `--reanchor`, `e` edit (suspends the TUI and runs `edit`), `x` reject, `R` rerun, `u` resume, `l` logs.
- **Result file:** `--result-file` writes `{"intent": "accept|edit|reject|rerun|resume|open|back", "selected": ["rw-abcd2345#2"]}` on exit. Without the flag nothing is written. Every action is also a command.

## Consumer notes

- A run is read with `review-run show` and `list --label`; review-watch never reads another tool's `state.db` and never imports it.
- review-run's exit 13 has two shapes: an error envelope with `code: escalated` and `hook` (a `pre-run` hook escalation: no run exists) and a run object with `state: escalated` (the session escalated). review-watch tells them apart with `--json`.
- `meta.review_run` fields other than the closed enums are claims. `verdict`, `risk` and `summary` never authorize anything; `risk` can only raise the tier.
- The draft's `body`, finding titles and bodies are untrusted: they reach the host only through pr-comment, which neutralizes them.
- A review posted by hand after review-watch started its round is not detected until the next poll's snapshot; the posted review through pr-comment still succeeds, and the earlier hand review does not cancel the round.

## Upgrades

- The state database migrates forward with additive changes only; an older tool refuses a newer store.
- Added `--json` fields are not breaking; a consumer ignores unknown fields. A change to a field's meaning, a removed field, an exit code or a hook payload key is a breaking change and follows the versioning rule in design guide section 13.
- Pinned sibling versions: forgectl and review-run and pr-comment must be the same major and minor as the schemas copied into review-watch's tests; a mismatch exits 12 `tool_too_old` (or `lower_contract_mismatch` when a field is missing).

## Residual risks

- A prompt-injected review session can steer its own draft; the draft is untrusted, reviewed by a human or an enforced gate before it is posted, and neutralized by pr-comment.
- A gate's `allow` is a decision about the declared input, not proof that the revision was judged: `gatectl show <id>` returns the declared head, which review-watch verifies equals the draft's head before acting.
- `forgectl` `ci.state: passing` does not say every expected check registered; `blocked_by` and the hard rules reduce but do not remove that.
- `poll` sees a PR again only while forgectl lists the request as outstanding, so a request that is cleared before the next poll and re-requested within the same interval can look like one request.
- Detached processes without jobsctl have no process-level isolation beyond their own session group.
- The spend cap needs review-run's `cost_usd`; until then it cannot trigger.

## Platform notes

Linux, WSL and macOS. review-watch needs no tmux and no notifier. Cron has a minimal environment: use `cron install`, which writes `PATH`, and keep the sibling tools' directories on it. A review needs bubblewrap and socat on Linux and WSL (review-run's sandbox); review-watch itself runs no sandboxed process.
