# review-watch

Run reviews on incoming review requests unattended, as drafts you accept.

## Status

Designed (interface 0.1.0-design): the interface is specified in [`docs/interface.md`](docs/interface.md) and the schemas are in `schemas/`; not yet implemented.

## Layer

L3 (workflow)

## What it does

- Finds the PRs that ask you to review them by name (team-level requests are ignored) on the hosts you list, and records each request as an item. It skips draft PRs, bot-authored PRs (an unknown author type counts as a bot), your own PRs, PRs you already reviewed yourself and PRs whose author is not a member, collaborator or trusted author (GitHub `OWNER`, `MEMBER` or `COLLABORATOR`; GitLab a non-fork head; or a login in `tiers.trusted_authors`), unless `start.external_authors: review`; skipped items are kept with their reason and re-checked, and a human `rerun` starts one.
- Starts a review for each new request through review-run (a jobsctl job, or `review-run run --no-wait` called by `poll` under an internal cap); `poll` never waits for a review. A new head under the same request never starts a review by itself; a fresh request, a re-request after a posted review or an explicit `rerun` does, up to a per-PR round limit (default 3).
- Copies each finished review into an editable draft (`edit` keeps revisions), flags it stale when the head moves, and posts it through pr-comment when you `accept` (with `--reanchor` for a moved head). Every post shows pr-comment's exact plan first.
- Computes a deterministic risk tier from PR facts; model output can only raise it. Auto-post and auto-approve are off by default, go through gatectl (shadow first) and are always limited by built-in hard rules (CI passing on the current head, no off-limits paths, never your own or a bot PR).
- A daily round limit (`limits.rounds_per_day`, default 50, `null` is off) bounds how many reviews start per day, against request spam; rounds over the limit wait for the next day. A dollar cap is planned for a later release, after review-run reports cost.
- GitLab merge requests are tier 3 unless `tiers.trusted_authors` names the author, so with the defaults GitLab is never automatic.

## Commands

```text
review-watch poll [--repo ID] [--no-start]
review-watch list [--state S] [--repo ID] [--pr PR] [--all] [--limit N]
review-watch show <id> [--round N] [--draft] [--no-refresh]
review-watch edit <id> [--draft-file F] [--base-revision N]
review-watch accept <id> [--reanchor] [--dry-run] [--approve] [--yes]
review-watch reject <id> [--reason-file F] [--yes]
review-watch rerun <id> [--force] [--yes]
review-watch resume <id> [--prompt-file F] [--yes]
review-watch status
review-watch logs <id> [--run] [--limit N]
review-watch cron install|uninstall [--schedule "..."]
review-watch export [--log] [--since T]
review-watch prune [--older-than D]
review-watch browse
```

Every command that prints data takes `--json`. `accept`, `edit`, `reject`, `rerun` and `resume` are human-only: without a TTY they need an explicit `--actor human:<name>`, and the ones that ask need `--yes` or a TTY (otherwise exit 13). `resume` and `rerun` only change state; `poll` launches. `poll --no-start` is the preview of a cycle (it discovers and finalizes but starts nothing); `--dry-run` previews `accept`, `reject`, `rerun`, `resume`, `cron` and `prune`. `browse` needs the `tui` extra. Exit codes follow the design guide (0 success, 2 usage, 10 not found, 11 vetoed, 12 failed, 13 needs a human, 20 refused with a `reason`, 21 ambiguous); the full list is in [`docs/interface.md`](docs/interface.md).

## Configuration

`~/.config/review-watch/config.yaml` (plus `config.d/*.yaml`); every key is documented in [`config.example.yaml`](config.example.yaml). The one required key is `hosts`: nothing runs until you name the hosts to watch. Credentials belong to forgectl; review-watch reads none. Scalar keys can be set with `REVIEW_WATCH_<KEY>` environment variables.

## Hooks

`pre-round` (may veto, which skips the request, or escalate, which holds the round until `resume`), `post-draft`, `post-attention` (one hook with a `reason`: escalated, stale, failed, round limit, daily limit, poll failed, submit failed, post failed) and `post-post`. Payloads carry ids, shas, counts and enums, never PR or model text. Post hooks are delivered at least once; deduplicate by `event_id`. Wire them to your notifier.

## Dependencies

- `forgectl` (lists review requests and PR facts); binding A (hard)
- `review-run` (runs reviews); binding A (hard)
- `pr-comment` (validates drafts, re-anchors and posts); binding A (hard)
- `gatectl` (optional: auto-post and auto-approve policies, track records); binding A, without it nothing is automatic
- `jobsctl` (optional: the review job queue); binding A, without it `poll` starts each review with `review-run run --no-wait` under an internal cap. Jobs get only jobsctl's inherited variables (by default `HOME`, so `gh` and `glab` logins work); name token variables in `jobs.pass_env`.

review-run needs bubblewrap and socat on Linux and WSL for its sandbox. A missing required tool exits with its install command.

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/review-watch
# for the terminal UI (review-watch browse):
uv tool install 'review-watch[tui]'
```

Then run `review-watch cron install` to poll every five minutes, and `review-run cron install` too: review-run's `reconcile` applies the expiry of escalated runs. If you wire `poll` into a hook with its own timeout, allow at least `poll.deadline` (default 20 minutes) plus 30 minutes.

## Documentation

- System architecture: [`../awkit-docs/architecture.md`](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- Tool design guide: [`../awkit-docs/design-guide.md`](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- This tool's design: [`docs/design.md`](docs/design.md)
- This tool's interface: [`docs/interface.md`](docs/interface.md)
- Implementation plan: [`docs/plan.md`](docs/plan.md)

## License

MIT — see LICENSE.
