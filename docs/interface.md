# review-watch: interface

Status: designed, 2026-11-11 (interface 0.1.0-design). This document is the single normative source for review-watch's rules. `design.md` records the decisions and the internal structure and links to the headings here instead of restating them; the README, the architecture and the schemas follow it. Machine-readable shapes are in `schemas/`.

## Concepts

review-watch finds the pull requests (GitHub) and merge requests (GitLab) that ask you to review them, runs a review on each through review-run, and holds the result as a **draft** that you edit and accept. Accepting posts the review through pr-comment. Nothing is posted without a human acceptance unless an enforced gatectl policy allows it and the built-in hard rules pass. `poll` is the only command that looks for new requests; it never waits for a review.

Only `poll`, `accept`, `rerun` (one `forgectl show`), `resume`, `reject` (the cancel) and `show` (through finalize) contact a host or a sibling tool, apart from `edit` (`pr-comment draft check`) and `logs --run` (`review-run show`). `list` reads `state.db`; `status` reads `state.db` and `gatectl stats`.

### Identifiers

- **Repo id** and **PR reference:** as in the design guide, section 4.1. `host/<full project path>` for a repo; a PR is a URL or `host/<full project path>#N` (GitLab numbers written without `!`); output always uses `#N`. The canonical `pr` and `pr_id` come from forgectl (`pr_id` is stable across repository renames). On every listing, snapshot and show, review-watch matches by `pr_id` and updates the item's stored `pr` and `repo` (logged as `item_renamed`); a read whose `pr_id` differs from the item's is refused as `lower_contract_mismatch` for that item.
- **Item:** one review request. An item is keyed by `(pr_id, reviewer, instance)`, where the reviewer is the user's own account (compared by id) and `instance` is forgectl's `review_requests[].instance` (its grammar is forgectl's; this tool caps it at 128 characters). A re-request is a new instance and so a new item. A push does not change the instance, so a new head under one instance is the same item and never starts a review by itself. A null `instance` matches the pair's newest item that is not `closed`. A non-null instance first seen for a pair whose newest item has `instance: null` is adopted by that item (stored, no new item) when its `requested_at` is null or not later than the item's `first_seen_at`; otherwise it is a new request. The first sight of `(pr_id, reviewer)` with `instance: null` starts once; later sightings of null are the same item.
- **GitLab re-request:** GitLab writes no new instance when the same set of reviewers is sent again. review-watch stores the last seen `review_requests[].state` per item; on GitLab a change from `reviewed` to `outstanding` on a `posted` item or a `skipped` (`already_reviewed`) item is a re-request: the item adds a round with trigger `re_request` (subject to `max_rounds`). After a comment-only review the state stays `outstanding`, so a re-sent request then is not detectable; the user runs `rerun`.
- **Round:** one review of an item at one head sha. A round has its own run, draft and state. Rounds are numbered per item from 1.
- **Item id:** `rw-` followed by 8 lowercase base32 characters (`a-z`, `2-7`); the `rw-` is optional in input. A unique prefix of at least 4 characters after `rw-` is accepted (shorter exits 2, ambiguous exits 21, unknown exits 10); output always shows the full id. **Round id:** `<item id>#<n>`; a prefix applies to the item part. A command that takes an item id accepts a round id where noted, and otherwise acts on the item's latest round. `--round` together with a round id exits 2.
- Run, job, draft and decision ids, `instance` and `pr_id` follow the siblings' grammars (review-run's `rr-` ids, jobsctl's `jb-` ids, pr-comment's draft ids, gatectl's decision ids); the schemas in `schemas/defs.json` copy them, pinned by the siblings' repository versions.
- **Labels passed to review-run:** `rw-item=<item id>` and `rw-round=<n>`, plus the jobsctl job's own labels (Launch).
- **Time:** every stored and printed timestamp is UTC as `YYYY-MM-DDTHH:MM:SS.ffffffZ`. Durations take `0` or an integer and a unit: `ms`, `s`, `m`, `h`, `d`.
- **Actor:** `^(human|auto):[A-Za-z0-9._@-]{1,64}$`. A command that takes `--actor` defaults to `human:<me.name>` on a TTY and `auto:unknown` otherwise. Human-only actions are listed under Human-only actions.

### Human-only actions

Human-only commands are `accept`, `edit`, `reject`, `rerun`, `resume`, and the flag `--force`. They need a TTY or an explicit `--actor human:<name>`; otherwise they exit 13 with `code: confirmation_required` and `reason: human_required` (design guide 4.4). Commands that ask (`accept`, `reject`, `rerun`, `resume`) ask once on a TTY; without a TTY and without `--yes` they exit 13 with `reason: confirmation_required`. The human checks are advisory against a same-user process that claims `--actor human:<name>` (a hook or script can run `accept --yes`); the audit records `actor_source` and `stdin_tty` for every such action.

### Path matching

One matcher serves every path glob (`off_limits`, `tiers.sensitive_globs`, `auto_approve.paths`). It is not `fnmatch`.

- Patterns are split on `/`. `*` and `?` never cross `/`. A `**` segment matches zero or more directories.
- A pattern without `/` matches the last path segment at any depth. A pattern with `/` is anchored at the repo root.
- Both sides are compared after ASCII case folding. Every changed file's `path` and, for a rename, its `previous_path` are tested.
- Repo-id globs (Scope) use the same segment rules without the any-depth rule.

| Pattern | Path | Matches |
|---|---|---|
| `.github/**` | `.github/workflows/ci.yml` | yes |
| `.github/**` | `docs/.github/x.yml` | no (anchored) |
| `CODEOWNERS` | `docs/CODEOWNERS` | yes (any depth) |
| `**/auth/**` | `src/Auth/login.py` | yes (case folded) |
| `*.pem` | `certs/server.pem` | yes |
| `Dockerfile*` | `services/api/Dockerfile.prod` | yes |
| `.env.*` | `config/.env.local` | yes |
| `a/*.txt` | `a/b.txt` | yes |
| `a/*.txt` | `a/c/b.txt` | no |
| `a/**/b.txt` | `a/b.txt` | yes (`**` matches zero directories) |

### Untrusted text

Everything that comes from a host (titles, bodies, branch names, paths, comments) and everything a model wrote (the draft's text, `meta.review_run.summary`, `risk_reasons`) is untrusted. review-watch stores PR ids, shas, counts, enum values and the draft itself; it stores no PR title or body. It never puts untrusted text in argv or env, and never in a hook payload. Human output prints every untrusted string through one sanitizer: control characters and bidirectional marks become visible `<U+XXXX>` marks and the text is capped at 256 characters per string (the draft body view in `show --draft` and the TUI is capped per field, not per line). The TUI and, when it uses Rich, the human output of `logs` render every untrusted string as plain text (markup off or escaped). Stored revisions, `show --draft`, `--json` output and the edit file are serialized with ASCII escapes, so bidirectional and C1 characters appear as `\uXXXX`. `--json` output carries untrusted strings verbatim, marked `x-untrusted` in the schemas. Identifiers read from siblings and hooks (`decision_id`, `run_id`, `job_id`, `review_id`, `instance`, `pr_id`) are validated against their `defs.json` grammars (`instance` and `pr_id` capped at 128 characters); a mismatch is `lower_contract_mismatch` (a hook's bad `decision_id` is dropped with a warning). Every SQL statement binds its values.

## Items, rounds and states

### Item states

- `open`: eligible; no round yet, or its latest round is not terminal, or ended `failed` or `rejected`.
- `skipped`: a direct request that fails a skip rule (Skip rules). It carries `skip_reason`; `skip_reason` is non-null exactly when `state` is `skipped`.
- `posted`: its latest round is `posted` and no round is in progress.
- `rejected`: a human rejected it (`reject <item id>`); no round starts automatically. `rerun` reopens it.
- `closed`: the PR is merged or closed. Rounds that were not terminal become `closed`.

### Item transitions

| From | To | Cause |
|---|---|---|
| `open` | `skipped` | a skip rule applies (including `hook_veto`) |
| `skipped` | `open` | the rule is cleared (Skip rules) |
| `open` | `posted` | the latest round is `posted` |
| `posted` | `open` | a new round: a re-request (trigger `re_request`) or `rerun` |
| `open`, `posted` | `rejected` | `reject <item id>` |
| `rejected` | `open` | `rerun` |
| any state | `closed` | the PR is merged or closed (also for a `failed` or `rejected` item) |
| `closed` | `open` | the PR is open again and listed: no automatic round, `rerun` is available |

Closure is detected for every item that is not `closed`. An item whose PR the host listing did not return in this cycle and that has no non-terminal round gets one `forgectl show` at most once per `poll.recheck_interval` (default `1h`). `not_found` or `permission_denied` three times in a row sets the item flag `unreachable` (shown by `list` and `status`, never a state change). Nothing is closed on a failed read.

### Round states

`queued → running → drafted → posting → posted`, plus `escalated`, `failed`, `stale`, `rejected` and `closed`.

| State | Meaning | Terminal |
|---|---|---|
| `queued` | recorded, no run yet; `queue_reason` says why (`waiting_slot`, `daily_limit`, `held`, `submit_failed`, `retry_wait`); `not_before` holds a retry until its time | no |
| `running` | a job (or an inline `review-run run --no-wait`) started a run; `phase` mirrors review-run's when known | no |
| `escalated` | a run escalated (its agent is kept, a held job waits), or review-run's `pre-run` hook escalated before any run existed (`escalation_kind: pre_run`) | no |
| `drafted` | the run is `done`; the draft is stored and waits for a human or a gate | no |
| `stale` | the draft's head or base is no longer the PR's; `stale_reason` is `stale_head` or `retargeted` | no |
| `posting` | a post was started and its outcome is not known yet; `post_intent` is stored (Post intent and recovery) | no |
| `posted` | pr-comment posted the review (the result is stored) | yes |
| `failed` | the last attempt failed, or a failure that is not retryable; `failure` carries the reason | yes |
| `rejected` | a human rejected the round | yes |
| `closed` | the PR merged or closed before the round ended, or the round was superseded (`closed_reason`: `pr_closed`, `pr_merged`, `superseded`) | yes |

A round's `head_sha` is the PR head at creation, refreshed from the cycle's `forgectl show` while the round is `queued`, set to `anchor.head_sha` when the draft is stored and moved to the new head after a re-anchor. `revision` is null before `drafted`.

### Transitions

- `queued → running`: a job is added (jobsctl) or `review-run run --no-wait` returned a run (Launch). A round stays `queued` when the daily limit, a `pre-round` hold or a retry wait applies.
- `running → drafted`: finalize read the run as `done` and stored the draft. `running → queued` (`retry_wait`) for a retryable failure with attempts left; `running → failed` otherwise (the run is `failed` with a failure that is not retryable or the attempts are used up, or no run came out of the launch: Run outcomes). `running → escalated`: the run is `escalated`. `escalated → running`: `resume` (session escalation). `escalated → queued`: `resume` of a `pre_run` escalation. `escalated → failed`: the run expired or was canceled.
- A retry never passes through `failed`: the failed run's id stays in `run_ids`, its failure is stored as `last_failure`, `retry_after_s` (null means 0) becomes `not_before`, and `attempt` is the number of runs started for the round. Only `poll` resubmits.
- `drafted → stale` when the head or base moved. `drafted → posting → posted` through `accept` or an automatic post. `stale → posting` only through `accept --reanchor` for `stale_head`. A `retargeted` round cannot be re-anchored.
- `posting → posted`, `posting → drafted` or `posting → stale` (nothing landed), `posting → rejected` (a human, no live holder), `posting → closed` with `post_unconfirmed: true`: all in Post intent and recovery.
- `drafted | stale | escalated | queued | running → rejected` through `reject`. Any non-terminal state `→ closed` when the PR is merged or closed. A stale round becomes `closed` (`superseded`) when `rerun` starts a newer round.
- Terminal states are immutable. Every transition is a compare-and-set on the round's stored state and `rev`, so two processes cannot both apply it. A command that fails to take a round's lease or loses a compare-and-set re-reads the round once and re-evaluates its preconditions; if it is still blocked it exits 20 `round_busy` (with `details.state`), except `show`, which prints the stored row with `finalize: busy`. Nothing waits for a lease.

### Skip rules

A direct request in scope (Scope) that fails a rule is stored as an item in state `skipped` with a `skip_reason`:

| `skip_reason` | Rule | Cleared by |
|---|---|---|
| `draft` | the PR is a draft | the PR becomes ready |
| `bot_author` | `author.is_bot` is true | never |
| `bot_unknown` | `author.is_bot` is null (counts as a bot) | never |
| `own_pr` | the author's account id equals your own (forgectl `whoami`) | never |
| `already_reviewed` | you reviewed it yourself (below) | the condition no longer holds |
| `hook_veto` | a `pre-round` hook vetoed the round (Hooks) | a new head, a new instance or `rerun` |

- Team-level requests are not requests for you by name: a PR whose only request is to a team is not listed by `forgectl prs --review-requested-from me`, so it is never seen. When a snapshot shows your direct request next to a team request, only the direct one counts.
- **Already reviewed, GitHub:** a listed request is always outstanding (a submitted review clears it), so the rule applies only when `requested_at` is null and a review by your account (compared by id) exists. **GitLab:** `review_requests[].state` is `reviewed`; a comment-only review leaves the request `outstanding`, and an unpublished draft reads `outstanding`.
- For an item that has no round, any review of yours counts (marker or not) and the item is `skipped` (`already_reviewed`). For an item with a round, reviews and notes carrying the marker are this tool's own and are ignored by the rule. On GitLab the notes come from the snapshot's `threads` (the newest 100 of your own notes are checked with pr-comment's Marker detection rule, referenced here, not restated). There is no `posted` item without a round.
- A snapshot whose `incomplete` names `review_requests` creates no item and changes no item's skip decision; one that names `reviews` (GitHub) or `threads` (GitLab marker notes) makes no `already_reviewed` decision (the item is left as it was).
- `list` hides `skipped` items unless `--all` is given. `status` shows the count per `skip_reason`.
- A skipped item is re-evaluated from a fresh snapshot by the Snapshot selection rules. A condition cleared makes the item `open`, and the first round is created if the round cap allows. GitLab lists finished requests too; their `state` decides. Items skipped for `bot_author`, `bot_unknown` or `own_pr` are never snapshotted again.

### Scope

- Config `hosts` (required, no default) names the hosts to poll; `include` and `exclude` are global lists of repo-id globs (Path matching segment rules; `exclude` wins; default `include: ["**"]`). A request outside the scope is not recorded; `status` counts how many were seen in the last poll.
- `me` per design guide section 5 (`me.github`, `me.gitlab`, else the authenticated user through forgectl).

### Risk tiers

The tier of a round is the maximum over independent rules, computed from the forgectl snapshot, `files` and the config. An unknown fact gives tier 3.

| Tier | Rules (any one) |
|---|---|
| 3 | `off_limits`, `size_unknown`, `files_incomplete`, `author_untrusted`, `ci_not_passing`, `base_not_default`, `fork_not_allowed`, `input_missing`, `model_risk_high` (codes and triggers in Rule codes) |
| 2 | `lines_high`, `files_high`, `sensitive_path`, `model_risk_medium` |
| 1 | `lines_medium`, `files_medium` |
| 0 | otherwise |

- Lines come from the PR's `additions` and `deletions`; when null (GitLab) they are the sum over `files` when `files_complete` is true and every file has counts, else unknown.
- **Model output can only raise the tier:** `meta.review_run.risk` `high` makes tier 3, `medium` at least 2; `low` changes nothing. `author_association` can only make a tier stricter.
- The tier is computed twice. At round creation it is computed for the `pre-round` payload and display and kept as `tier_at_creation`. When an automatic action is enabled for the repo, it is computed again at `drafted` from facts bound to the draft's head (Fact binding); that is the only tier the gate and `max_*_tier` see, and it is the round's `tier`. Each tier is stored with its reasons (`tier_reasons`).
- The tier decides only two things: the declared `risk_tier` input of a gate policy, and the built-in limits `max_post_tier` (default 1) and `max_approve_tier` (default 0) beyond which an automatic action is not even considered.
- `tiers.trusted_authors` entries are logins compared case-insensitively, applied only where `author_association` is null (GitLab). With defaults every GitLab PR is tier 3 (`author_untrusted`), so GitLab is never automatic unless `tiers.trusted_authors` names the author.

### Hard rules

Applied before any automatic action, whatever a gate says. Any failing rule means the draft waits for a human, and `show` lists the failing rule codes in `auto_blocked_by`:

- `own_pr` (account ids are compared, else logins case-insensitively, as forgectl does), `bot_author` (including unknown, and `is_bot` null): never automatic.
- `ci_not_passing`: forgectl `ci.state` is not `passing` on the PR's current head at the time of the check, or `blocked_by` holds a `checks_*` or `other` code.
- `off_limits`: the PR touches a configured off-limits path (Path matching). Default list (`off_limits`): CI and build definitions (`.github/**`, `.gitlab-ci.yml`, `.gitlab/**`, `.circleci/**`, `Jenkinsfile`, `azure-pipelines.yml`, `Dockerfile*`, `docker-compose*`, `Makefile`, `GNUmakefile`), hooks and submodules (`.husky/**`, `.pre-commit-config.yaml`, `.gitmodules`), agent configuration and instruction files (`**/.claude/**`, `.mcp.json`, `CLAUDE*.md`, `AGENTS.md`), review ownership (`CODEOWNERS`), and security-sensitive code and secrets (`**/auth/**`, `**/security/**`, `**/secrets/**`, `**/crypto/**`, `*.pem`, `*.key`, `.env`, `.env.*`). A per-repo `off_limits` list adds to the global list and never removes from it. The list is the veto for both actions.
- `tier_too_high`: the tier is above `max_post_tier` (post) or `max_approve_tier` (approve).
- `meta_signal`: the round's `meta`, copied from revision 1's `meta.review_run` (review-run's stored `review.json`, checked against the run; not from the run `summary`, which lacks the session fields), shows `coverage.status: partial`, `code_run.status: skipped` (review-run uses `skipped` only when a code run was configured; `not_configured` and `failed` do not fail the rule), `head_moved: true` or null, `session.sandbox` other than `required`, `session.tools_checked: false`, a non-zero `session.tools_unexpected`, or any of these fields missing.
- `head_moved_since_draft`: the head or base is not the draft's anchor (staleness).
- `reanchored`: the draft was re-anchored (never auto-posted and never auto-approved).
- `edited`: a round with a human revision is never posted or approved automatically.
- `facts_unbound`: a fact the rules need is not bound to the draft's head (Fact binding). Fails closed.
- `repo_changed`: the item's `repo` changed after the round was created (a per-repo override or a gatectl row may no longer match).
- `findings_block_approve` (approve only): `meta.review_run.verdict` is not `approve`, `severity_counts` has any `blocker` or `major`, `severity_counts.unspecified` is above 0, or `coverage.skipped` is not empty.
- `automatic_request_changes`: an automatic post requires the draft's `event` to be `comment`. `event` and `verdict` both go into the gate input.
- `approve_path_not_allowed` (approve only): an automatic approve requires every changed path (and every `previous_path`) to match the repo's `auto_approve.paths`; empty means no automatic approve in that repo.
- `approve_author_not_trusted` (approve only): an automatic approve requires the PR author's account id in the global `auto_approve.trusted_author_ids`. `author_association` and logins never suffice for approve (they still feed the tier for posts).
- `gate_warning` and `gate_error` are recorded by the gate flow (Automatic actions).
- A rule that cannot be evaluated because a fact is unknown fails closed.

### Rule codes

One vocabulary serves `tier_reasons`, `auto_blocked_by` and `skip_reason`. The prose lists and the schema enums are the same sets.

| Code | Appears in | Trigger |
|---|---|---|
| `off_limits` | tier 3, `auto_blocked_by` | a changed path matches `off_limits` |
| `size_unknown` | tier 3 | `additions` or `deletions` null with the file list incomplete or without counts, or `changed_files` null |
| `files_incomplete` | tier 3 | the file list is incomplete (`files_complete: false`); size is unknown |
| `author_untrusted` | tier 3 | `author_association` is neither a member nor a collaborator (GitHub: not `OWNER`, `MEMBER` or `COLLABORATOR`); null (GitLab) unless `tiers.trusted_authors` names the author |
| `ci_not_passing` | tier 3, `auto_blocked_by` | `ci.state` is not `passing`, or `blocked_by` holds a `checks_*` or `other` code |
| `base_not_default` | tier 3 | `base.is_default` is false or null |
| `fork_not_allowed` | tier 3 | the head repo is a fork and `tiers.allow_forks` is false |
| `input_missing` | tier 3 | any rule input is missing (also a failed `files` read at creation) |
| `model_risk_high` | tier 3 | `meta.review_run.risk` is `high` |
| `lines_high` | tier 2 | more than `tiers.lines[1]` changed lines (default 300) |
| `files_high` | tier 2 | more than `tiers.files[1]` files (default 15) |
| `sensitive_path` | tier 2 | a path matches `tiers.sensitive_globs` (default none) |
| `model_risk_medium` | tier 2 | `risk` is `medium` |
| `lines_medium` | tier 1 | more than `tiers.lines[0]` lines (default 50) |
| `files_medium` | tier 1 | more than `tiers.files[0]` files (default 3) |
| `own_pr`, `bot_author` | `auto_blocked_by`; `skip_reason` | as in Skip rules and Hard rules |
| `tier_too_high`, `meta_signal`, `head_moved_since_draft`, `reanchored`, `edited`, `facts_unbound`, `repo_changed`, `findings_block_approve`, `automatic_request_changes`, `approve_path_not_allowed`, `approve_author_not_trusted`, `gate_warning`, `gate_error` | `auto_blocked_by` | as in Hard rules and Automatic actions |
| `draft`, `bot_unknown`, `already_reviewed`, `hook_veto` | `skip_reason` | as in Skip rules |

## Detection and the poll cycle

`poll` is the only command that looks for review requests and the only one that submits reviews. A cycle:

1. Take the poll lease (design guide section 4.3). A second `poll` exits 0 with `already_running: true` (zero counts, `started_at` equal to `finished_at`).
2. Preflight: config valid, `forgectl` found at a compatible version (`tool_missing` or `tool_too_old`, exit 12), the state database usable.
3. Identity: `forgectl whoami --host <h> --json` once per host per cycle (your account id and login), unless `hosts.<h>.me` pins the login; account ids come from whoami.
4. Per host: `forgectl prs --host <h> --review-requested-from me --state open --limit 1000 --json`. Drafts are included (so they can be recorded as skipped). `complete: false` or `errors` for the host is recorded; nothing is inferred from an absence in an incomplete listing.
5. Match each listed candidate to the stored items by `pr_id`, choose the snapshots (Snapshot selection) and take them (`forgectl snapshot <pr> --json`). The snapshot gives the `instance`, request state, draft, bot, size, ci, `blocked_by` and `author_association`.
6. Match the item (Identifiers). A new key creates an item; the skip rules decide `open` or `skipped`.
7. Finalize: for every `running` or `escalated` round, read its run (Finalize). Finalize runs before any head comparison, so a draft whose anchor is already behind becomes `stale` in the same cycle before any gate runs.
8. For every item with a round in `queued`, `running`, `escalated`, `drafted`, `stale` or `posting`: `forgectl show <pr> --json` once (head, base, state). A merged or closed PR closes the item and its non-terminal rounds (`closed`); a head change marks a `drafted` round `stale` (`stale_head`), a base ref change `retargeted`; when both moved the reason is `retargeted`. Also check the items the listing did not return (Item transitions).
9. Retry automatic `posting` rounds (Post intent and recovery).
10. Start: create a round for each `open` item with no round when the cap allows; submit queued rounds in `created_at` order while capacity remains (Launch).
11. Gates for every `drafted` round that has no recorded gate result for the policy, whoever finalized it (Automatic actions).
12. Fire hooks, append the cycle record to `log.jsonl`, release the lease.

- **Pacing:** `rate_limited` from forgectl pauses that host for the rest of the cycle and honors `retry_after`; review-watch has no limiter of its own beyond `poll.max_snapshots`. `forgectl` reads retry inside it.
- **Failures:** a failure on one PR, host or round never stops the others, each of which runs in its own transaction. A host failure (`auth_required`, `permission_denied`, `rate_limited`, `upstream_error`, `network_error`, an unknown or unsupported host) skips the host for the cycle and leaves its items untouched. A failed per-PR read leaves that item as it was. Nothing is marked stale, closed, skipped or expired because a listing or a read failed.
- **Repeated failures:** each host keeps a count of consecutive cycles with an error. When it reaches `poll.failure_alert_after` (default 3), `post-attention` fires once with reason `poll_failed`; it fires again only after a successful cycle.
- **Exit codes of `poll`:** 0 when the cycle completed, also when some hosts or items failed (the result lists them in `errors[]`); 12 when it could not run (`tool_missing`, `tool_too_old`, `state_unsafe`, or every configured host failed with `reason: all_hosts_failed`, printing only the error envelope with `details.hosts: [{host, code}]`); 2 for usage and for invalid config (`config_invalid`, like every command).
- `poll --no-start` discovers and finalizes and checks staleness but submits and gates nothing; it is the preview. `poll --repo <id>` limits the cycle to one repo.

### Snapshot selection

Per host and cycle, at most `poll.max_snapshots` (default 30) snapshots are taken, in this order:

1. candidates with no item, oldest `updated_at` first;
2. items whose listed `head.sha`, `draft` or `updated_at` differs from the stored listing values, and, on GitHub, every listed item whose stored state is `posted` (a submitted review clears a GitHub request, so being listed again means a new request);
3. `skipped` items with reason `draft` only when the listing's `draft` is now false, `already_reviewed` and `hook_veto` items, and `open` or `posted` items without a non-terminal round, whose last snapshot is older than `poll.recheck_interval` (default `1h`), oldest last snapshot first.

Candidates beyond the budget wait; the poll report and `status` show `snapshots_deferred` per host. `updated_at` is only one trigger among three, and the recheck interval covers missed changes.

## Timeouts and leases

Each call to a sibling has a fixed timeout, and a timeout kills the exact child's process group:

| Call | Timeout |
|---|---|
| forgectl reads | 120 s |
| `review-run list`, `show`, `cancel` | 300 s |
| `review-run run --no-wait` | 30 min |
| `review-run resume --no-wait` | 300 s |
| `pr-comment draft check` | 300 s |
| `pr-comment review` | 600 s |
| `gatectl check` | `gate.timeout` plus 30 s |
| `gatectl outcome`, `stats` | 60 s |
| jobsctl | 60 s |
| hooks | per config |

A timed-out call is a failure of that object only. Child stdout is capped at 32 MiB. The `poll` lease and the round leases last 15 minutes and are renewed by the holder after every host, round and sibling call; a holder that finds its lease taken stops at the next boundary.

## Rounds

### Creating a round

A round is created when an `open` item has no round (its first), when a human runs `rerun`, or when a fresh request instance or a `re_request` creates a new round. Round creation reads `forgectl files <pr> --json` once for the creation tier; a failed read gives tier 3 with reason `input_missing`. Creation checks, in order, and a failed check leaves the round uncreated and reports it (automatic starts record the reason in the log and `post-attention`; `rerun` exits 20):

1. **Round cap:** `max_rounds` per PR across instances, default 3 (`repos.<id>.max_rounds`). A round counts once `session_launched` is true, that is, once a run of the round has a non-null `agent_id`. Rounds that failed before a session (a `pre-run` veto, `tool_missing`, `tool_too_old`, an unusable host) do not count. At the cap an automatic start is not made (`round_limit`, `post-attention` once per item, reset when a round is created) and `rerun` exits 20 `reason: round_limit`; a human passes `rerun --force` or raises `max_rounds`; automation never does.
2. **Daily limit** (Daily limit): at the limit the round is created `queued` with `queue_reason: daily_limit`.
3. **`pre-round` hooks** (Hooks), run exactly once per round, here, after the scope, skip and cap checks and before capacity: a veto records the item `skipped` with `skip_reason: hook_veto` and no round is created (`hook_veto` in the log); an escalation creates the round `queued` with `queue_reason: held` and fires `post-attention`.
4. **Capacity** (Launch): without capacity the round is `queued` with `queue_reason: waiting_slot`.

`trigger` values: `first_sight` (an item created by the first poll that saw this PR and reviewer), `new_request` (a new instance for a PR and reviewer already known), `re_request` (GitLab re-request), `skip_cleared`, `rerun`. `n` in the payload is the number the round will get.

### Launch

- **Argv:** `review-run run <pr> --json --mode headless [--skill <repos.<id>.review.skill or review.skill>] --label rw-item=<id> --label rw-round=<n> [--previous <run>] --actor auto:review-watch`, plus `--no-wait` when review-watch calls it itself (without jobsctl). Every launch passes `--json`.
- **With jobsctl** (`jobs.use_jobsctl: auto` finds a `jobsctl` binary; `true` requires it; `false` never uses it): `jobsctl add --queue <jobs.queue> --unique-key review:<round id> --label owner=review-watch --label rw-item=<id> --label rw-round=<n> --label pr=<pr> --max-retries 0 --kill-grace 125s --cwd <state dir>/jobs --json [--pass-env VAR]... -- <absolute review-run> run <pr> --json ...`. `<state dir>/jobs` is an empty private directory created at startup. The key is within jobsctl's 256-character limit; it is per round, so a job can only be this round's. `--pass-env` names come from `jobs.pass_env` (default empty); jobs get only jobsctl's inherited variables (by default `HOME`, so `gh` and `glab` logins work) and `jobs.pass_env` for token variables. The job's argv holds only ids and shas, never untrusted text.
- The job id is stored in the same transaction as the submission result and every later read is `jobsctl show <job id> --json`; `show --unique-key` is used only to recover a job whose id was not stored after a crash. `created: false` adopts the job. A `pass_env_undefined` warning means the job was already created: review-watch cancels it (`jobsctl cancel <job id>`), leaves the round `queued` (`queue_reason: submit_failed`) and fires `post-attention` `submit_failed` once per round. `existing_job_held` links to the held job.
- Each cycle submits at most `effective_free` jobs, re-read before each submission (`jobsctl capacity --queue <jobs.queue>`; null means unbounded, and then `max_concurrent` bounds it).
- **Without jobsctl:** `poll` runs `review-run run <pr> --no-wait --json ...` itself as a bounded synchronous call (30-minute timeout). review-run returns once the session's agent is stored, with the run object (exit 0, `state: running`) or, before any run exists, the error envelope (exits 2, 10, 11, 12, 13, 20). From then on the session is supervised by agent-spawn and finalized by review-watch's refreshing `review-run show` (Finalize). `max_concurrent` (default 2) counts this tool's rounds whose run is `running` (no PIDs are stored). `poll` blocks for each start during review-run's pre-fetch and optional code run; starts per cycle are bounded by free capacity. There are no detached processes.
- There is no wrapper command: a dead job or `poll` leaves review-run's own run record as the truth.
- **`--previous`:** when `review.previous` is true and the PR has a `done` run of this tool (any item, found with `review-run list --pr <pr> --state done --label rw-round --json`, newest first), its id is passed as `--previous`, after `review-run show <run> --no-refresh --json` confirms it still exists; exit 10 drops the flag and logs a `warning` with code `previous_unavailable`. review-run's retention (30 days by default) bounds how far back `--previous` reaches.
- review-run does not dedupe `run`: review-watch finds an existing run for a round with `review-run list --label rw-item=<id> --label rw-round=<n> --json` before it submits, and never submits a second job for a round that has a non-failed run.

### Run outcomes

review-watch learns the outcome of a launch from three sources: the run found by its labels, the inline envelope (without jobsctl), and, with jobsctl, the job read by its stored `job_id` (`jobsctl show <job id> --json`: `status`, `hold_reason`, the current attempt's `exit.code`). One table maps them to the round:

| Observation | Round | Counts toward `max_rounds` | Retried |
|---|---|---|---|
| a run exists | follow the run (Finalize) | when any run of the round has a non-null `agent_id` | per the run's `retryable` |
| no run; exit 11 (envelope `vetoed`, or job `failed` with exit 11) | `failed`, `pre_run_vetoed` | no | no |
| no run; exit 13 (envelope `escalated` with `hook`, or job `held` with `exit_13`) | `escalated`, `escalation_kind: pre_run`, `run_id: null` | no | no (`resume`) |
| no run; exit 12 envelope | `failed`, the envelope's `reason` (`tool_missing`, `tool_too_old`, `state_unsafe`, `lower_failed`, ...) | no | only when `details.retryable` is true |
| no run; exit 20 envelope (`not_open`, `unknown_host`, `lower_refused`) | `failed`, that reason (`not_open` also triggers the item's closure check) | no | no |
| no run; exit 2 or 10 envelope | `failed`, `launch_usage` (`post-attention`) | no | no |
| no run; job ended with another exit, or job gone with no run | `failed`, `no_run` with `failure.exit_code` | no | when the exit was 12, up to `max_attempts` |

`session_launched` is a stored round flag, set when finalize first sees a run of the round with a non-null `agent_id` (review-run keeps `phase` only while running). With jobsctl the job status alone tells gone (terminal), held and running; nothing is read by PID. A job's exit code is read only when no run exists; with a run, the run is the truth. The `failure.reason` set is review-run's failed reasons (`prefetch_failed`, `stale_head`, `too_large`, `launch_failed`, `canceled`, `session_failed`, `unreported`, `killed`, `timeout`, `lost`, `runner_lost`, `agent_gone`, `expired`, `implicit_done`, `artifact_missing`, `artifact_mismatch`, `skill_missing`, `artifact_invalid`, `internal`), its no-run reasons (`tool_missing`, `tool_too_old`, `no_tmux`, `state_unsafe`, `schema_newer`, `plugin_changed`, `lower_failed`, `not_open`, `unknown_host`, `lower_refused`), pinned to review-run's schema version, and review-watch's own `no_run`, `draft_invalid`, `pre_run_vetoed`, `launch_usage` and `lower_contract_mismatch`. The failure object also carries `exit_code` and `lower_reason`.

### Finalize

For each `running` or `escalated` round, `poll`, `show` and `accept` run one idempotent finalize step under the round's own lease:

1. Find the run: the stored run id, else `review-run list --label rw-item=<id> --label rw-round=<n>`. Read it with plain `review-run show <run> --json` (it refreshes). review-run's own lease makes a concurrent refresh safe. This also finalizes a run whose session ended, including one a human answered through `agent-spawn resume --mode interactive`. Expiry by `escalation_ttl` is a review-run `reconcile` step, so run `review-run cron install` alongside review-watch's cron.
2. No run found: the table in Run outcomes decides the round.
3. Run `done`: read `review-run show <run> --draft` (the stored `review.json`), verify `draft_id`, `pr` and `anchor.head_sha` against the run object, validate it with `pr-comment draft check -`, store it as revision 1 with its sha256, copy the round's `meta` from revision 1's `meta.review_run`, set the state `drafted` and queue `post-draft`. A run whose draft fails these checks makes the round `failed` (`draft_invalid`).
4. Run `escalated`: the round becomes `escalated` (`escalation_kind: session`); `post-attention` fires once for the episode with reason `escalated`. A held job is left for `resume` or `reject`.
5. Run `failed`: when `retryable` is true and the round's attempts are below `max_attempts` (default 2), the round moves `running → queued` (`queue_reason: retry_wait`), the failure is stored as `last_failure` and `retry_after_s` becomes `not_before`; otherwise the round becomes `failed` with `failure` copied from the run. `post-attention` (`failed`) fires only when the round enters `failed`.
6. A held jobsctl job (exit 13) is resolved by `jobsctl resolve <job> --as done|failed|canceled` when the run ends or the round is rejected or closed. `jobsctl` refusal `not_held` is success.
7. A round that is `rejected` or `closed` with a `running` or `escalated` run gets `review-run cancel <run> --actor auto:review-watch` (or the human's actor for `reject`); a run already terminal is a no-op.

### Daily limit

`limits.rounds_per_day` (default 50, `null` is off) is counted at submission over this tool's rounds started on the current local day (`limits.timezone`, default the machine's zone). At the limit, new rounds stay `queued` with `queue_reason: daily_limit`, `post-attention` fires once per day (`daily_limit`), started runs finish, and `rerun --force` passes it for that round. It is a flood bound against request spam, not spend accounting; a dollar cap is deferred to a 1.x release that follows review-run reporting a `cost_usd`.

### Staleness

- Finalize runs before heads are compared. Each `poll` reads every non-terminal round's PR with one `forgectl show`. A head sha different from the draft's `anchor.head_sha` marks a `drafted` round `stale` (`stale_head`, with `stale_head_sha` and `stale_since`); a base ref different from `anchor.base_ref` marks it `retargeted`; when both moved, `retargeted`. `post-attention` fires once per round with reason `stale`.
- `accept` always re-reads the PR live and passes `--expected-head` to pr-comment (which has no `--expected-base`). Base protection is pr-comment's pin on `anchor.base_ref` (exit 20 `stale_base`), which review-watch maps to the round's `retargeted` handling (round `stale`, `stale_reason: retargeted`), plus its own `forgectl show` check. A stale flag that is out of date can never cause a wrong post.
- On a stale round: `accept --reanchor` (`stale_head` only) re-anchors through pr-comment (below), `rerun` starts a newer round (the stale one becomes `closed`, `superseded`), `reject` ends it.
- A `retargeted` round cannot be re-anchored (pr-comment needs a new draft). `accept` exits 20 with `reason: retargeted`.

## Review drafts

- The draft is a pr-comment review draft. Revision 1 is the stored copy of review-run's artifact; it keeps its `draft_id`, so a draft that already posted is never posted again and a new review is a new draft.
- Revisions are kept in `state.db` as `{n, sha256, size, author, created_at}`; `author` is `human:<name>`, or null for revision 1 (review-run's artifact). The round records `edited: true` while the latest revision's sha256 differs from revision 1's (a revert clears it).
- **`edit`:** see Commands. Edits to `draft_id`, `anchor`, `pr`, `schema_version` and `meta` are refused; the result must pass `pr-comment draft check -`. Only `drafted` and `stale` rounds are editable.
- **Re-anchoring:** `accept --reanchor` runs `pr-comment review <pr> --draft-file F --reanchor-to <current head> ...`. The result's `reanchored_draft` is stored as a new revision (findings pr-comment could not map are in its `unanchored`); the round's anchor head moves to the new head and the round records `reanchored: true`. A re-anchored round is never gated again, never auto-posted and never auto-approved, and files no gate outcome.

### Post intent and recovery

`posting` means a post was started and its outcome is not known yet. A round enters it by the compare-and-set that also stores its `post_intent`: `{revision, revision_sha256, approve, reanchor_to, actor, decision_ids, plan_sha256, automatic, started_at, holder}`. `holder` is the PID plus the process start time plus a 15-minute expiry renewed while the call runs. Every later attempt uses exactly this intent.

Every `pr-comment review` call uses the same flags except the last ones:

| Call | Argv |
|---|---|
| dry run | `pr-comment review <pr> --draft-file <revision file> --dry-run [--reanchor-to <head>] [--approve] --actor <actor> [--decision-id <id>]... --expected-head <head> --json` |
| post | `pr-comment review <pr> --draft-file <revision file> [--reanchor-to <head>] [--approve] --actor <actor> [--decision-id <id>]... --expected-head <head> --yes --expect-plan <sha256> --json` |

`--expected-head` is the anchor head or the re-anchor target, always passed. There is no `--expected-base`. The pr-comment result maps to the round as follows:

| pr-comment result | Round |
|---|---|
| exit 0 (including `already_posted: true`) | `posted`; store the result |
| exit 12 with `landed: true` (`quick_action_ran`, `approved_stale_head`, forgectl's `audit_failed`) | `posted`; store `post.warnings: [<code>]` |
| exit 2, 11, 13, 20, or 12 with `landed: false` and `retry: same_command` | back to `drafted` (to `stale` for `stale_head` or `stale_base`; `stale_base` gives `stale_reason: retargeted`); `post_error` stored; nothing landed |
| exit 12 with `landed: null` (`outcome_unknown`), or `retry: new_key`, or `retry: none` with `landed: false` (`host_rejected` with leftover drafts) | stays `posting`; `post_error` stored |

`post_error` carries `{code, reason, landed, retry, at}`. The stored success result has no `landed` (pr-comment's success result has none).

Resolving a `posting` round:

1. A caller that finds a live `holder` exits 20 `round_busy`. Without a live holder, `poll` retries a round whose intent is `automatic` with the identical command while the stored `post_error.retry` is `same_command`, at most `max_attempts` times; when it stops (another `retry`, a refusal, or a head that moved), the round stays `posting` and `post-attention` fires once with reason `post_failed`.
2. `accept` on a `posting` round takes no `--approve`, `--reanchor` or `--revision` (exit 2 `usage` if given). It reads the PR live and runs the normal dry-run, confirm and post flow from the intent's revision and `approve`. When the head moved and the intent has no `approve`, it adds `--reanchor-to <current head>` (the `review` dedupe key ignores the anchor, so a landed review reports `already_posted` and nothing posts twice). When the head moved and the intent has `approve`, it exits 20 `stale_head` with `next_actions: [reject]` (an approval key includes the head, so a retry would approve a head no one reviewed).
3. `reject` of a `posting` round without a live holder ends it `rejected` with `post_unconfirmed: true`. A PR that closes or merges under a `posting` round makes it `closed` with `post_unconfirmed: true` (pr-comment refuses `not_open` before any lookup, so no answer is possible); no `post-post` fires.

## Automatic actions

Defaults: all off. `auto_post.enabled` and `auto_approve.enabled` are per repo (`repos.<id>`) or global (the default for every repo that does not set its own), each with a `policy` (defaults `review-post` and `review-approve`). `auto_approve.enabled` without `auto_post.enabled` for the same repo is a config error.

For a `drafted` round with an automatic action enabled, in this order:

1. **Fact binding and hard rules** (Fact binding, Hard rules). A failing rule stops here (the draft waits; `auto_blocked_by` lists the codes).
2. **Post gate:** `gatectl check <policy> --repo <repo> --input - --timeout <gate.timeout in whole seconds, rounded up> --actor auto:review-watch --json`; gatectl's `--timeout` is a plain number of seconds, so `60s` would exit 2. review-watch's own wait for the call is `gate.timeout` plus 30 s (a caller that wires `poll` into a hook must keep the hook's own timeout above that). The input is a JSON object of declared fields only: `repo`, `object_type: "pr"`, `object_id` (the `pr`), `action` (`review` or `approve`), `head_sha` (the anchor head), `base_ref`, `risk_tier`, `tier_reasons`, `event`, `verdict`, `severity_counts`, `coverage_status`, `additions`, `deletions`, `changed_files`, `author_association`, `ci_state`, `round`, `edited`. The policy should list `repo` and `head_sha` in `input.require`. No PR text and no model text is in it.
3. **Approve gate:** only if the post gate allowed, auto-approve is enabled and the approve hard rules pass.
4. One `pr-comment review --dry-run` with the final flags (`--approve` only if both gates allowed; the actor is `auto:<post policy>`, decision ids passed post first, approve second), reads `plan.plan_sha256`, then the same call with `--yes --expect-plan <sha>`. The post uses revision 1 and its stored sha256, recorded in `post_intent` by the same compare-and-set that moves `drafted → posting`; an `edit` racing it loses with `round_busy` or `not_editable`. pr-comment itself requires `allow_auto_approve: true` for the repo in its own config. If only the post gate allowed, the review is posted without `--approve`.
5. A pr-comment refusal or failure in this flow (`auto_approve_disabled`, `auto_approve_reanchored`, `plan_changed`, a veto, an escalation) returns the round to `drafted` per Post intent and recovery, is not retried in the cycle, and fires `post-attention` `post_failed`.

Rules for the gate results:

- Only `enforced: true`, `decision: allow`, a `decision_id` and an empty `warnings` act. Any warning (`definition_changed`, `repo_mismatch`, `repo_invalid`, `config_problem`, `output_truncated`) means no action (`auto_blocked_by: gate_warning`). Shadow, `off`, `not_applicable`, an exit 11 or 13 (deny or escalate), and a missing `decision_id` leave the draft for a human. An exit 1, 2 or 12, a parse failure or a missing gatectl is `gate_error` (no decision recorded, `auto_blocked_by: gate_error`).
- A gate runs in `poll` only, for every `drafted` round that has no recorded gate result for that policy, whoever finalized it. A `gate_error` is retried on the next three cycles, then left for the human.
- Gate records store `{policy, decision_id, decision, enforced, state, warnings, error, attempts, checked_at, outcome}`. Shadow, `off` and `not_applicable` are stored with `enforced: false` (and `decision: null` for `off`).
- A re-anchored round is not gated again; a `rerun` creates a new round with a new gate call.
- To approve an already posted draft, a human uses `pr-comment approve` directly; review-watch never calls `review --approve` for a draft that was posted, and never approves from a `request_changes` draft (`approve_conflicts_event`).
- A revert detector is not review-watch's concern: a revert or incident is filed by whoever detects it with `gatectl outcome <id> disagree --signal revert`.

### Fact binding

The gate's tier and the hard rules use facts bound to `anchor.head_sha`: `forgectl show` (head, base, `is_default`, fork), `forgectl snapshot` (`ci` with `ci.head_sha` equal to the anchor, association, bot, request state) and `forgectl files <pr> --json` (its `head_sha` equal to the anchor). Any mismatch, `complete: false` on a part used, `files_complete: false` or a failed read fails closed with the hard rule `facts_unbound` (an incomplete file list is "size unknown" for the tier and fails `off_limits`). The file list is used in memory and not stored (no host text is kept). The gate input's `head_sha` is `anchor.head_sha`.

### Gate outcomes

Outcomes are filed per gate record that has a `decision_id`, as `gatectl outcome <id> agree|disagree --actor human:<name> --signal human`, once per decision id (`changed: false` is success):

| Gate | Human action | Gate's decision | Outcome filed |
|---|---|---|---|
| post | `accept` of an unedited round | `allow` | `agree` |
| post | `accept` of an edited round, or `reject` | `allow` | `disagree` |
| post | `accept` of an unedited round | `deny` or `escalate` | `disagree` (over-strict, counted only) |
| post | `reject` | `deny` or `escalate` | `agree` |
| approve | `accept --approve` (unedited) | `allow` | `agree` |
| approve | `accept --approve` (unedited) | `deny` or `escalate` | `disagree` |
| approve | `reject` | `allow` | `disagree` |
| approve | `reject` | `deny` or `escalate` | `agree` |

The approve gate's outcome is filed only when the human's `accept` carried `--approve` or the human rejected. Nothing is filed for a re-anchored round, for an `accept` that completes an automatic intent, for `rerun`, a superseded or closed round, or a record without a `decision_id`. The round's `edited` flag at that time decides. When gatectl is not installed or the call fails, the failure is logged and the human action still proceeds.

## Commands

### Flags

| Command | Flags |
|---|---|
| `poll` | `[--repo <id>] [--no-start]` |
| `list` | `[--state S]... [--repo <id>] [--pr <pr>] [--all] [--limit N]` |
| `show` | `<item id \| round id> [--round N] [--draft] [--no-refresh]` |
| `edit` | `<round id \| item id> [--draft-file <path>\|-] [--base-revision N]` |
| `accept` | `<round id \| item id> [--reanchor] [--dry-run] [--approve] [--override-escalation=<scope>] [--yes] [--expect-plan <sha256>]` |
| `reject` | `<round id \| item id> [--reason-file <path>\|-] [--dry-run] [--yes]` |
| `rerun` | `<item id> [--force] [--dry-run] [--yes]` |
| `resume` | `<round id \| item id> [--prompt-file <path>\|-] [--dry-run] [--yes]` |
| `status` | none |
| `logs` | `<item id \| round id> [--round N] [--run] [--limit N]` |
| `cron` | `install\|uninstall [--schedule "<cron fields>"] [--dry-run]` |
| `export` | `[--log] [--since <time>]` |
| `prune` | `[--older-than <duration>] [--dry-run]` |
| `browse` | `[--focus <item id \| round id>] [--result-file <path>] [--all]` |

Every command takes `--json`, `--config <path>`, `--verbose`, `--quiet`, `--actor`, `--decision-id` (repeatable, at most 16, `^[A-Za-z0-9._:@/-]{1,128}$`) and `--version`. `--hook <name>=<cmd>` (repeatable) is taken by `poll`, `accept`, `reject`, `rerun` and `resume`. `--dry-run` is taken only by the commands that change external state: `accept`, `reject`, `rerun`, `resume`, `cron`, `prune` (not `poll` and not `edit`). `--yes` skips a confirmation that review-watch itself would ask for; it never overrides a refusal. A flag given twice exits 2. Ids: an item id or, where noted, a round id.

### `poll`

`review-watch poll [--repo <id>] [--no-start] [--json]`

Runs the cycle in Detection and the poll cycle. Exit 0 or 12. `--json` prints the poll report: `{started_at, finished_at, hosts:[{host, listed, complete, snapshots, snapshots_deferred, error}], discovered, new_items, skipped, out_of_scope, finalized, stale, closed, started, queued, errors:[{host, pr, code, message}], already_running}`. Every counter is for this cycle: `discovered` (listed candidates in scope), `new_items`, `skipped` (items that became skipped), `out_of_scope`, `finalized`, `stale`, `closed`, `started`, `queued` (rounds left queued). With `already_running: true` the counts are zero and `started_at` equals `finished_at`.

### `list`

`review-watch list [--state S]... [--repo <id>] [--pr <pr>] [--all] [--limit N] [--json]`

Lists items with their latest round: `{items:[...], truncated}`. `--state` matches the item state for `skipped`, `posted`, `rejected` and `closed`, and otherwise the latest round's state (`open` for an item with no round). Order: states needing a human first (`drafted`, `escalated`), then `stale`, `failed`, `running`, `queued`, `posting`, `posted`, `rejected`, `closed`; then the latest round's last change, newest first, then item id. `skipped` items appear only with `--all` or `--state skipped`. Default limit 100. Does not contact a host.

### `show`

`review-watch show <item id | round id> [--round N] [--draft] [--no-refresh] [--json]`

Prints the item and the selected round (the latest by default): states, tier and reasons, gate decisions, `auto_blocked_by`, run id and phase, staleness, revisions, failure, attempts, hook records and the commands now possible (`next_actions`). Without `--no-refresh` a `running` or `escalated` round is finalized first (idempotent); a round whose lease is held is printed as stored, with `finalize: busy`. `--draft` prints the latest stored draft revision as pr-comment's JSON (untrusted, no envelope, ASCII escapes); history is in `logs`. Does not contact a host except through finalize.

### `edit`

`review-watch edit <round id | item id> [--draft-file <path>|-] [--base-revision N]`

Human-only. Without `--draft-file`, writes the latest revision to a private temporary file (mode 0600 in the state directory, created with `O_CREAT|O_EXCL|O_NOFOLLOW`, ASCII escapes), opens `$VISUAL`, else `$EDITOR` (split with `shlex`, run without a shell; neither set exits 12 `tool_missing` with `details.tool: editor`), reads it back, deletes it, and stores a valid change as a new revision. A non-zero editor exit stores nothing. An invalid result keeps the private temporary file, prints its path and exits 2 `invalid_draft`. With `--draft-file` the file or stdin is the new draft (stdin on a TTY exits 2). The result must pass `pr-comment draft check -`, keep `draft_id`, `anchor`, `pr` and `schema_version` unchanged and keep `meta` byte-identical to revision 1 (a changed `meta` is refused with `invalid_draft`). At most 50 revisions per round and 8 MiB per revision. `--base-revision` refuses (exit 20 `revision_changed`) when the latest revision is not N. Applies to `drafted` and `stale` rounds only (an `escalated` round has no draft): exit 20 `reason: not_editable` otherwise. The revision is stored under a compare-and-set that bumps the round's `rev`. `edit` has no `--dry-run`. An unchanged draft stores nothing (`changed: false`).

### `accept`

`review-watch accept <round id | item id> [--reanchor] [--dry-run] [--approve] [--override-escalation=<scope>] [--yes] [--expect-plan <sha256>]`

Human-only. Posts the latest revision of the round's draft through pr-comment.

1. The round must be `drafted` (or `stale` with `--reanchor`, or `posting` to finish an interrupted post, see Post intent and recovery); otherwise exit 20 (`not_drafted` with `details.state`, `stale_head`, `retargeted`, `round_busy`). An `escalated` round exits 20 `not_drafted`.
2. Finalize first if needed. Read the PR live (`forgectl show`). A head different from `anchor.head_sha` without `--reanchor` exits 20 `stale_head` (`details.head_sha`); a different base exits 20 `retargeted`.
3. Run the dry run of the Post intent and recovery table (`--expected-head`, no `--expected-base`; `--reanchor-to <head>` and `--approve` when requested; actor `human:<name>`). Print the plan (findings, moved and unanchored findings with reasons, neutralized text counts, the footer) and `plan_sha256`. `--dry-run` stops here, exit 0. A refused dry run prints only the error envelope.
4. Ask once on a TTY (`Post? [y/N]`). Without a TTY, without `--yes`, exit 13 `confirmation_required`; nothing is recorded (there is no pending store: the caller re-runs after a human decision). A TUI confirms in its own dialog and passes `--yes`.
5. Mark the round `posting` with its `post_intent` (compare-and-set), run the post call with `--yes --expect-plan <sha>`, store its result (`review_id`, `posted_inline`, `moved`, `already_posted`, `plan_sha256`) and mark `posted`. A `reanchored_draft` in the result is stored as a new revision first.
6. pr-comment's exits map to the round by the Post intent and recovery table: 11 vetoed and 13 `escalated` (`escalated_by` in the envelope; a human re-runs with `--override-escalation=<scope>`) return the round to `drafted`; 20 refusals (`stale_head`, `stale_base` (round `stale`, `retargeted`), `plan_changed`, `auto_approve_disabled`, `not_open`, `self_approval`, `drafts_present`, `pending_review_exists`) likewise; 12 failures keep their `landed` and `retry` fields. A post whose outcome is unknown leaves the round `posting`.
7. `--approve` passes through only for a `comment` draft (a `request_changes` draft exits 2 `approve_conflicts_event`).
8. After a successful post, file the gate outcomes (Gate outcomes) and fire `post-post`.

Exit codes: 0 posted (or `already_posted`), 2, 10, 11, 12, 13, 20 (`not_drafted`, `stale_head`, `retargeted`, `revision_changed`, `round_active`, `round_busy`, plus pr-comment's reasons).

### `reject`

`review-watch reject <round id | item id> [--reason-file <path>|-] [--dry-run] [--yes]`

Human-only. Ends the round (`rejected`) and, given an item id, rejects the item (state `rejected`, no automatic round). A live run is canceled with `review-run cancel`; a held job is resolved; open rounds of the item are `rejected`. A `posting` round without a live holder ends `rejected` with `post_unconfirmed: true`. Confirms on a TTY unless `--yes`. Files the gate outcomes (Gate outcomes). `--reason-file` is stored (untrusted, capped at 1 KiB). Every `reject` is audited.

### `rerun`

`review-watch rerun <item id> [--force] [--dry-run] [--yes]`

Human-only. Adds a round at the PR's current head to an item whose latest round is `posted`, `stale`, `failed`, `rejected` or `closed` with the PR still open (a `stale` round becomes `closed`, `superseded`). Exit 20 `round_active` while a round is `queued` (including `held`), `running`, `escalated`, `drafted` or `posting`; exit 20 `round_limit` at the cap; exit 11 `vetoed` when a `pre-round` hook vetoes; exit 13 `escalated` when a `pre-round` hook escalates (the round is created `held`). `--force` skips the round cap and the daily limit and is human-only; `pre-round` hooks still run. `rerun` of a `rejected` item reopens it.

### `resume`

`review-watch resume <round id | item id> [--prompt-file <path>|-] [--dry-run] [--yes]`

Human-only. One continuation for every hold and escalation; the result carries `kind: session|pre_run|pre_round`.

- A session escalation: calls `review-run resume <run> --prompt-file F --no-wait --json --actor human:<name>`; the answer is untrusted and travels in a file. The round returns to `running` and `poll` finalizes it. `resume` never blocks for a session.
- A `pre_run` escalation (no run exists): resolves the held job (if any) with `jobsctl resolve <job> --as canceled`, then submits a fresh launch for the same round through the normal path with `--override-escalation --actor human:<name>` added (inline without jobsctl, a new job otherwise); the round goes `queued`.
- A `held` round (a `pre-round` escalation, `kind: pre_round`): re-runs the other `pre-round` hooks, treats the escalating hook's verdict as allow (design guide 4.4), records `override_escalation` in the audit and moves the round to `queue_reason: waiting_slot`.

Exit 20 `not_escalated` for any other round. A run that escalates again stays `escalated`; there is never an automatic resume.

### `status`

`review-watch status [--json]`

Counts per item and round state; `skipped` per reason; `today` (`rounds_started` and the `limit`); rounds waiting (`waiting_slot`, `daily_limit`, `held`); the last poll (`started_at`, `finished_at`, per-host error, consecutive failure count and `snapshots_deferred`); out-of-scope requests seen; `unmatched_repo_keys` (the `repos.<id>` keys that matched no seen repo in the last 30 days); and, when gatectl is configured, `gatectl stats --policy <p> --repo <r> --json` for each enabled policy (`recommend`, `upper_bound`, decision and outcome counts), or `gatectl: not_configured`. Does not contact a host.

### `logs`

`review-watch logs <item id | round id> [--round N] [--run] [--limit N]`

Prints the item's events from `log.jsonl` (state changes, gate calls, hook results, errors), oldest first, at most `--limit` (default 200); with `--run`, passes through `review-run show <run> --no-refresh` for each of the round's runs (sanitized). Human output only unless `--json`.

### `cron`

`review-watch cron install|uninstall [--schedule "<cron fields>"] [--dry-run] [--json]`

Writes or removes one managed block, following pr-watch's convention. The block starts at `# review-watch managed` and ends at `# review-watch managed end`: line 1 `# review-watch managed v1`, line 2 `PATH=<dirs>`, line 3 `<schedule> . '<env_file>' && '<absolute review-watch>' poll --quiet` (the `. '<env_file>' &&` part only with `cron.env_file`), last line `# review-watch managed end`. A block without its end line is never replaced or removed (exit 2 `usage`, with a hint to remove it by hand). The default schedule is `*/5 * * * *` (`cron.schedule` or `--schedule` overrides it); a schedule is five fields of `[0-9*/,-]` or month and day names. `PATH` is the de-duplicated directories of the resolved absolute `*_cmd[0]` values plus `~/.local/bin`, `/usr/local/bin`, `/usr/bin` and `/bin`. `cron.env_file`, every path and every `PATH` entry is refused (exit 2 `config_invalid`) when it holds a control character, `'`, `%` or a newline; the generated block is parsed back and compared before it is written. No `crontab` binary exits 12 `tool_missing`. Install is idempotent. Output `{action, dry_run, changed, replaced, block, warnings}` (install) with `block` a list of lines, and `{action, dry_run, changed, block}` (uninstall); `warnings` holds `stray_poll_line` when a `review-watch poll` line exists outside the block.

### `export`

`review-watch export [--log] [--since <time>] [--json]`

Prints `audit.jsonl` (or `log.jsonl` with `--log`) records as JSON lines, or one array with `--json`. `--since` takes an RFC 3339 time or a duration.

### `prune`

`review-watch prune [--older-than <duration>] [--dry-run] [--json]`

Deletes an item with its rounds and events only when the item is `closed` (PR merged or closed) and its last change is older than `retention.closed` (default 90 days). For `posted`, `rejected` and `skipped` items older than the retention it deletes draft revision blobs other than revision 1's hash record and keeps the item and round rows, so the request key, the round count for `max_rounds` and a rejection survive. It removes `tmp/` files only when their round is terminal or they are older than 24 hours. Never touches a non-terminal round. Output `{older_than, results:[{item, result, reason}], warnings, dry_run}` with `result` one of `deleted`, `trimmed`, `would_delete`, `would_trim`, `skipped`. Exit 0.

### `browse`

See TUI.

## Exit codes

| Exit | Meaning |
|---|---|
| 0 | success (`poll` cycle completed, also with some errors; `accept` posted or `already_posted`) |
| 1 | internal error |
| 2 | usage (including `usage`, `approve_conflicts_event`, `invalid_draft` and `config_invalid`) |
| 10 | not found (item, round, revision, draft) |
| 11 | vetoed (a `pre-round` hook, or pr-comment's or forgectl's veto passed through by `accept`) |
| 12 | failed for a reportable reason (`tool_missing`, `tool_too_old`, `state_unsafe`, `all_hosts_failed`, `lower_contract_mismatch`, `lower_failed`) |
| 13 | needs a human (`confirmation_required` with `human_required` or `confirmation_required`, an escalation of a lower tool or a `pre-round` hook) |
| 20 | refused, with a `reason` |
| 21 | ambiguous id prefix |

There are no tool-specific codes above 21. Refusal reasons (exit 20): `not_drafted`, `not_editable`, `not_escalated`, `round_active`, `round_busy`, `round_limit`, `daily_limit`, `stale_head`, `retargeted`, `revision_changed`, `item_closed`, and the reasons passed through from pr-comment (`stale_base`, `plan_changed`, `auto_approve_disabled`, `auto_approve_reanchored`, `not_open`, `self_approval`, `drafts_present`, `pending_review_exists`, `not_resolvable`, `too_large`, `unknown_host`) and from review-run (`live`, `lower_refused`). `human_required` is an exit 13 reason, not an exit 20 refusal. `round_limit` and `daily_limit` are both a refusal reason and an attention reason.

## Output by exit code

- `--json` prints at most one document on stdout. Exit 0 prints the command's result object. Any non-zero exit prints only the error envelope (a refused `accept --dry-run` prints no plan; `poll` exit 12 prints no report).
- Errors go to stderr in human mode. Without `--json`, `poll` prints one line per notable event and a summary line.

### Error envelope

`{"error": {"code", "message", "reason", "details", "retry_after", "lower_tool", "lower_code", "lower_reason", "escalated_by", "hook", "landed", "retry", "decision_ids"}}`. `code` is one of `usage`, `config_invalid`, `not_found`, `ambiguous`, `vetoed`, `escalated`, `confirmation_required`, `refused`, `failed`, `tool_missing`, `tool_too_old`, `state_unsafe`, `internal`. Only `code` and `message` are required; `reason` is required when `code` is `refused`. `lower_contract_mismatch` is `code: failed`, `reason: lower_contract_mismatch`, with `lower_tool`. `lower_tool`, `lower_code` and `lower_reason` identify a sibling's failure that passed through; `escalated_by`, `hook`, `landed` and `retry` are copied from pr-comment's or review-run's envelope unchanged. `details` is an open object (`head_sha`, `key`, `required`, `found`, `install`, `state`, `tool`, `path`, `hosts`).

### Result objects

A **round** carries: `id`, `item`, `n`, `state`, `queue_reason`, `head_sha`, `base_ref`, `tier`, `tier_at_creation`, `tier_reasons`, `run_id`, `run_ids`, `phase` (review-run's `prefetch`, `code_run`, `session`, `finalize`, or null), `attempt`, `session_launched`, `job_id`, `draft_id`, `revision` (latest, null before `drafted`), `edited`, `reanchored`, `stale_reason`, `stale_head_sha`, `stale_since`, `escalated_since`, `escalation_kind`, `not_before`, `last_failure`, `failure` (`reason`, `retryable`, `retry_after_s`, `exit_code`, `lower_reason`), `post_gate`, `approve_gate` (gate records, Automatic actions), `auto_blocked_by`, `meta` (verdict, severity counts, risk, coverage status, `code_run_status`, `head_moved`, sandbox, `tools_checked`, `tools_unexpected`), `post` (the stored pr-comment result: `review_id`, `posted_inline`, `moved`, `already_posted`, `plan_sha256`, `warnings`), `post_intent`, `post_error`, `post_unconfirmed`, `hook_decision_ids`, `reject_reason`, `closed_reason`, `created_at`, `drafted_at`, `posted_at`. An **item** carries: `id`, `pr`, `pr_id`, `repo`, `instance`, `state`, `skip_reason`, `vetoed_head_sha`, `hook_decision_ids`, `unreachable`, `rounds` (count), `latest_round` (the round object or null), `first_seen_at`, `last_seen_at`. A **revision** is `{n, sha256, size, author, created_at}`. A **hook record** is `{hook, ts, exit_code, duration_ms, timed_out, decision_ids}`. `next_actions` is a list of `accept`, `accept_reanchor`, `edit`, `reject`, `rerun`, `rerun_force`, `resume`.

Per-command results (shapes in `schemas/cli/`):

| Command | Result |
|---|---|
| `poll` | the poll report |
| `list` | `{items, truncated}` |
| `show` | `{item, round, revisions, hooks, next_actions, finalize}` (`finalize`: `done`, `busy`, `skipped` or null) |
| `edit` | `{round, revision, changed, sha256, size}` |
| `accept` | `{round, post, automatic: false, reanchored, outcomes}`; with `--dry-run` `{dry_run, round, plan, plan_sha256}` |
| `reject` | `{target, item, rounds, run_canceled, job_resolved, outcomes, dry_run, changed}` |
| `rerun` | `{item, round, superseded, reopened, forced, dry_run}` |
| `resume` | `{round, run_id, kind, dry_run}` |
| `status` | `{items, rounds, skipped, today, waiting, last_poll, out_of_scope, unmatched_repo_keys, gatectl}` |
| `logs` | `{item, events, runs}` |
| `cron` | as in `cron` |
| `export` | an audit record, a log record, or an array of either |
| `prune` | `{older_than, results, warnings, dry_run}` |
| `browse` | the result file (TUI) |

## Hooks

Hooks follow design guide section 8: `sh -c` in their own process group, JSON on stdin, `REVIEW_WATCH_HOOK`, `REVIEW_WATCH_ID`, `REVIEW_WATCH_OBJECT_TYPE` in the environment, a 30-second default timeout per hook (configurable; a duration, a bare integer means seconds), no template substitution. `async` is refused on `pre-round` by the config loader. Payloads (`schemas/hooks/<hook>.json`) carry `tool`, `hook`, `object_type`, `object_id`, `repo`, `labels` (`key=value` strings: `rw-item`, `rw-round`, `pr`) and the fields below. `object_type` is `round`, `item` (for `post-attention` `round_limit` and `daily_limit`, and a `pre-round` veto log record) or `host` (`poll_failed`), with `object_id` and `REVIEW_WATCH_OBJECT_TYPE` and `REVIEW_WATCH_ID` set to match; `daily_limit` uses `object_type: item` with `object_id` the item of the first round it held that day. They never carry PR text, a title, a draft body, a summary or any model text. A tool that stores no pending confirmations has no `post-pending`.

- **`pre-round`**: runs exactly once per round, at creation, after the scope, skip and cap checks and before capacity (a held or waiting round is never re-asked). Payload: `pr`, `item`, `n` (the number the round will get), `head_sha`, `tier`, `tier_reasons`, `trigger` (`first_sight`, `new_request`, `re_request`, `skip_cleared`, `rerun`), `forced`. A hook may veto (exit other than 0 and 13): the item is recorded `skipped` with `skip_reason: hook_veto` and the hook's `decision_id` on the item (`hook_decision_ids`); the veto is re-asked only when the listed head differs from the vetoed head (`vetoed_head_sha`), a new instance arrives, or a human runs `rerun` (exit 11 `vetoed`). A hook may escalate (exit 13): the round is created `queued` with `queue_reason: held`, `post-attention` fires (`escalated`, `details.kind: pre_round`), and `poll` never submits a `held` round. A human continues it with `resume <round>`. A printed `decision_id` is recorded. A timeout counts as a veto. `pre-round` is not delivered later and has no `event_id`.
- **`post-draft`**: a round reached `drafted`. Payload: `event_id`, `pr`, `item`, `n`, `head_sha`, `draft_id`, `tier`, `verdict`, `severity_counts`, `coverage_status`. It fires from the `drafted` transition's outbox, whoever finalized; gate decisions are read with `show`.
- **`post-attention`**: something needs a human; one hook with a closed `reason` (Attention reasons). Payload: `event_id`, `pr`, `item`, `n`, `head_sha` (null where there is no round), `reason`, `details` (`kind`, `failure_reason`, `retryable`, `stale_reason`, `host`, `consecutive_failures`; never text).
- **`post-post`**: a round reached `posted`. Payload: `event_id`, `pr`, `item`, `n`, `actor`, `decision_ids`, `review_id`, `head_sha`, `automatic`, `reanchored`, `edited`.
- Post hooks run after the transition committed, detached, and are delivered at least once through the outbox, deduplicated by `event_id` (`<object id>-<hook>-<n>`, `n` the cause's sequence); every post hook carries `event_id`. Their failures are logged and never fatal.
- `--hook <name>=<cmd>` adds a hook for one invocation (timeout 30 s, async for post hooks).

### Attention reasons

One table lists every `post-attention` reason with its dedupe key and reset:

| Reason | Dedupe key | Reset | Notes |
|---|---|---|---|
| `escalated` | the round plus the escalation episode (review-run's `escalated_run`, or `pre_run`, or `pre_round`) | a new episode fires again | `details.kind` is `session`, `pre_run` or `pre_round` |
| `stale` | the round | none | |
| `failed` | the round | none | `details.failure_reason` includes `expired` |
| `round_limit` | the item | when a round is created | object type `item` |
| `daily_limit` | the day | the next local day | object type `item` (the item of the first round held that day) |
| `poll_failed` | the host | after a successful cycle | object type `host` |
| `submit_failed` | the round | none | a jobsctl submission failed (`pass_env_undefined`) |
| `post_failed` | the round | none | an automatic post stopped (`posting` kept, or a refusal returned the round to `drafted`) |

## Config

Location and precedence follow design guide section 5 (`~/.config/review-watch/config.yaml`, drop-ins in `config.d/`, flags > `repos.<id>` > environment > global > defaults). Schema: `schemas/config.json`; every key with its default is in `config.example.yaml`. Invalid config fails fast with the key and the problem (exit 2 `config_invalid`, `details.key`), for every command.

| Key | Default | Notes |
|---|---|---|
| `hosts` | none (required) | a list of host names, or a map `host: {me: <login>}`; a missing or empty value exits 2 |
| `include`, `exclude` | `["**"]`, `[]` | repo-id globs (Path matching); `exclude` wins; global only |
| `me` | per section 5 | `name`, `github`, `gitlab` |
| `max_rounds` | 3 | per PR, across instances; per repo override (free) |
| `max_concurrent` | 2 | rounds with a running run counted by this tool (without jobsctl, or when jobsctl reports no limit); per repo free |
| `max_attempts` | 2 | runs per round; per repo free |
| `jobs.use_jobsctl` | `auto` | `auto`, `true`, `false` |
| `jobs.queue` | `reviews` | jobsctl queue; review-watch passes `--kill-grace 125s` and `--max-retries 0` |
| `jobs.pass_env` | `[]` | names for `--pass-env`; values come from jobsctl's env file |
| `review.skill` | `null` | review-run's default skill; per repo override |
| `review.previous` | `true` | pass the PR's newest done review as `--previous` |
| `tiers.lines` | `[50, 300]` | per-repo override may only be lower or equal elementwise |
| `tiers.files` | `[3, 15]` | per-repo override may only be lower or equal elementwise |
| `tiers.sensitive_globs` | `[]` | per-repo values add to the global list |
| `tiers.trusted_authors` | `[]` | logins compared case-insensitively, applied only where `author_association` is null; global only |
| `tiers.allow_forks` | `true` | `false` makes a fork PR tier 3; a per-repo value may only be `false` |
| `off_limits` | the list in Hard rules | per-repo values add to the global list |
| `max_post_tier`, `max_approve_tier` | 1, 0 | per-repo override may only be lower or equal |
| `auto_post.enabled`, `auto_post.policy` | `false`, `review-post` | global (the default for every repo) or per repo |
| `auto_approve.enabled`, `auto_approve.policy` | `false`, `review-approve` | global or per repo; needs `auto_post.enabled` for the same repo; pr-comment needs `allow_auto_approve` too |
| `auto_approve.paths` | `[]` | per repo only; path globs (Path matching); an automatic approve needs every changed path to match; empty means no automatic approve in that repo |
| `auto_approve.trusted_author_ids` | `[]` | global only; account ids; an automatic approve needs the PR author's id here |
| `gate.timeout` | `60s` | a duration of 1 s to 86400 s, passed to `gatectl check --timeout` in whole seconds |
| `limits.rounds_per_day` | `50` | rounds started per local day; `null` is off |
| `limits.timezone` | `null` (local) | an IANA zone name |
| `poll.max_snapshots` | 30 | per host and cycle |
| `poll.recheck_interval` | `1h` | the recheck interval of Snapshot selection and closure checks |
| `poll.failure_alert_after` | 3 | consecutive failing cycles per host |
| `retention.closed` | `90d` | `prune`; closed items; other items lose old revisions only |
| `cron.schedule`, `cron.env_file` | `*/5 * * * *`, `null` | |
| `forgectl_cmd`, `review_run_cmd`, `pr_comment_cmd`, `gatectl_cmd`, `jobsctl_cmd` | the tool names | binding A; the value is an argv list run without a shell, absolute paths recommended under cron |
| `hooks.<name>` | `{}` | lists of `{cmd, timeout, async}`; `async` is refused on `pre-round` |
| `repos.<id>` | `{}` | keys are canonical repo ids (exact, as forgectl returns them); per-repo overrides of `max_rounds`, `max_attempts`, `max_concurrent`, `tiers` (`lines`, `files`, `sensitive_globs`, `allow_forks`), `max_post_tier`, `max_approve_tier`, `off_limits`, `auto_post`, `auto_approve` (`enabled`, `policy`, `paths`), `review.skill` |

There is no `token_env`: credentials belong to forgectl and the lower tools.

### Config validation

Cross-field checks the config loader makes beyond the schema (ranges are the schema's):

- A per-repo `tiers.lines`, `tiers.files`, `max_post_tier` or `max_approve_tier` that is looser than the global value (elementwise) is invalid; `tiers.lines[0]` and `tiers.files[0]` may not exceed their `[1]`.
- A per-repo `tiers.allow_forks` may only be `false`. `include`, `exclude`, `tiers.trusted_authors` and `auto_approve.trusted_author_ids` are global only. `off_limits` and `tiers.sensitive_globs` add.
- `auto_approve.enabled` without `auto_post.enabled` for the same repo is invalid.
- `gate.timeout` above 86400 s is invalid. `async` on a `pre-round` hook is invalid.
- Safety keys (`max_*_tier`, `tiers.*`, `off_limits`, `auto_*`, `gate.*`, `limits.*`) cannot be set from the environment. `max_rounds`, `max_attempts` and `max_concurrent` per repo are free (cost, not safety).
- `repos.<id>` keys that match no seen repo in the last 30 days are listed by `status` (`unmatched_repo_keys`), not refused.

## Environment variables

Scalar config keys read `REVIEW_WATCH_<KEY>` (dashes and nesting joined with `__`, for example `REVIEW_WATCH_LIMITS__ROUNDS_PER_DAY`), except the safety keys listed in Config validation. Lists, `hooks`, `repos` and the tool commands cannot be set from the environment. Hook variables: `REVIEW_WATCH_HOOK`, `REVIEW_WATCH_ID`, `REVIEW_WATCH_OBJECT_TYPE`.

## Audit and log

- **`audit.jsonl`** (`schemas/audit.json`; public, append-only): one record pair, `attempted` then the outcome sharing an `attempt_id` (`phase: attempted|outcome`), for each external action: an `accept` or an automatic post (`auto_post`), every `reject` (a human decision that feeds the track record), a `rerun`, a `resume` (with `override_escalation`), and each `gatectl outcome` filed (`gate_outcome`). Fields: `ts`, `phase`, `actor`, `decision_id` (a list when several), `target` (the canonical `pr`; the round id is in `inputs.round`), `action` (`accept`, `auto_post`, `reject`, `rerun`, `resume`, `gate_outcome`), `inputs` (`head_sha`, `draft_id`, `draft_sha256`, `plan_sha256`, `config_sha256`, `round`, `override_escalation`, `actor_source`, `stdin_tty`), `result` (null for `attempted`), `attempt_id`. `poll`, discovery, finalizing and state changes go to the log only. `awkit audit` pairs these records with pr-comment's and forgectl's by target, action and time.
- **`log.jsonl`** (`schemas/log.json`): events `poll_cycle`, `item_state`, `round_state` (with the reason), `round_not_created` (reasons `round_limit` and `hook_veto`), `gate_call`, `hook_result`, `finalize`, `host_failed`, `gate_outcome`, `item_renamed`, `warning`. No text from a host or a model.

## Files

State directory `~/.local/state/review-watch/` (respecting `XDG_STATE_HOME`): `state.db` (SQLite, WAL, private; tables `items`, `rounds`, `revisions`, `runs`, `events`, `leases`, `hosts`; additive migrations only), `audit.jsonl`, `log.jsonl`, `jobs/` (the empty working directory for jobsctl jobs), `tmp/` (private files for editing, mode 0700, files 0600 created with `O_CREAT|O_EXCL|O_NOFOLLOW`; `prune` removes them when their round is terminal or they are older than 24 hours). `state_unsafe` is review-run's rule: a state root or `tmp/` that is a symlink, not owned by the user, or group or other writable. The store carries a schema version; a newer one exits 12 `state_unsafe` with a hint, and it never downgrades.

## Labels

review-watch has no labels of its own. It passes `rw-item` and `rw-round` to review-run and to jobsctl, and `owner=review-watch` and `pr=<pr>` to jobsctl. No other tool may give those keys meaning.

## TUI

`review-watch browse [--focus <item id | round id>] [--result-file <path>] [--all]`. A Textual TUI over the CLI actions; it needs the `tui` extra (`uv tool install 'review-watch[tui]'`); without it, exit 12 `tool_missing`. Without a TTY it exits 2. It runs as a foreground child of another TUI, takes no lease and refreshes with `list` and `show --no-refresh`.

- **Layout:** the left pane lists items sorted by need (`drafted` and `escalated`, then `stale`, `failed`, `running`, `queued`, the rest; `skipped` only with `--all` or the `a` key). The right pane shows the selected round: tier and reasons, verdict, severity counts, the gate decision, `auto_blocked_by`, staleness and the draft's findings by file, all through the sanitizer and rendered as plain text (no markup). Gate decisions, tier, verdict and state are shown in widgets that never hold untrusted text.
- **Keys:** `q` quit, `?` help, `/` filter, `enter` open a finding, `r` refresh, `a` accept (a dialog shows the `--dry-run` plan and `plan_sha256`, then runs `accept --yes --expect-plan`), `A` accept with `--reanchor`, `e` edit (suspends the TUI and runs `edit`), `x` reject, `R` rerun, `u` resume, `l` logs.
- **Result file:** `--result-file` writes `{"intent": "open|back|done", "selected": ["rw-abcd2345#2"], "done": [{"action", "round", "exit_code"}]}` on exit 0 only (`schemas/cli/browse-result.json`). Actions taken in the TUI are listed in `done`. Without the flag nothing is written. Every action is also a command.

## Consumer notes

- A run is read with `review-run show` (refreshing, so it finalizes finished runs) and `list --label`; review-watch never reads another tool's `state.db` and never imports it.
- review-run's exit 13 has two shapes: an error envelope with `code: escalated` and `hook` (a `pre-run` hook escalation: no run exists) and a run object with `state: escalated` (the session escalated). review-watch tells them apart with `--json`, which every launch passes.
- `meta.review_run` fields other than the closed enums are claims. `verdict`, `risk` and `summary` never authorize anything; `risk` can only raise the tier.
- The draft's `body`, finding titles and bodies are untrusted: they reach the host only through pr-comment, which neutralizes them.
- A review posted by hand after review-watch started its round is not detected until the next poll's snapshot; the posted review through pr-comment still succeeds, and the earlier hand review does not cancel the round.

## Upgrades

- The state database migrates forward with additive changes only; an older tool refuses a newer store.
- Added `--json` fields are not breaking; a consumer ignores unknown fields. A change to a field's meaning, a removed field, an exit code or a hook payload key is a breaking change and follows the versioning rule in design guide section 13.
- Pinned sibling versions: forgectl and review-run and pr-comment must be the same major and minor as the schemas copied into review-watch's tests; a mismatch exits 12 `tool_too_old` (or `lower_contract_mismatch` when a field is missing). The schemas carry no version field; the pin is by the siblings' repository versions.

## Residual risks

- A prompt-injected review session can steer its own draft; the draft is untrusted, reviewed by a human or an enforced gate before it is posted, and neutralized by pr-comment.
- A gate's `allow` is a decision about the declared input, not proof that the revision was judged; the head is pinned by `--expected-head` and the declared input.
- `forgectl` `ci.state: passing` does not say every expected check registered; `blocked_by` and the hard rules reduce but do not remove that.
- `poll` sees a PR again only while forgectl lists the request as outstanding, so a request that is cleared before the next poll and re-requested within the same interval can look like one request. On GitLab a re-sent request after a comment-only review keeps the state `outstanding` and is not detectable; run `rerun`.
- The human-only checks are advisory against a same-user process that claims `--actor human:<name>`; the audit records `actor_source` and `stdin_tty`.
- The daily limit is a count of started rounds, not spend accounting.

## Platform notes

Linux, WSL and macOS. review-watch needs no tmux and no notifier. Cron has a minimal environment: use `cron install`, which writes `PATH`, and keep the sibling tools' directories on it. Also run `review-run cron install`, so review-run's `reconcile` applies `escalation_ttl` expiry. A review needs bubblewrap and socat on Linux and WSL (review-run's sandbox); review-watch itself runs no sandboxed process. Without jobsctl, `poll` blocks for each start while review-run fetches the PR and runs the optional code run.
