# review-watch

Run reviews on incoming review requests unattended, as drafts you accept.

## Status

Designed (interface 0.1.0-design): the interface is specified in [`docs/interface.md`](docs/interface.md) and the schemas are in `schemas/`; not yet implemented.

## Layer

L3 (workflow)

## What it does

- Finds the PRs that ask you to review them by name (team-level requests are ignored) on the hosts you list, and records each request as an item. It skips draft PRs, bot-authored PRs (an unknown author type counts as a bot), your own PRs and PRs you already reviewed yourself; skipped items are kept with their reason and re-checked.
- Starts a review for each new request through review-run (a jobsctl job, or a detached process under an internal cap); `poll` never waits for a review. A new head under the same request never starts a review by itself; a fresh request or an explicit `rerun` does, up to a per-PR round limit (default 3).
- Copies each finished review into an editable draft (`edit` keeps revisions), flags it stale when the head moves, and posts it through pr-comment when you `accept` (with `--reanchor` for a moved head). Every post shows pr-comment's exact plan first.
- Computes a deterministic risk tier from PR facts; model output can only raise it. Auto-post and auto-approve are off by default, go through gatectl (shadow first) and are always limited by built-in hard rules (CI passing on the current head, no off-limits paths, never your own or a bot PR).
- Optional daily spend cap (off by default).

## Commands

```text
review-watch poll [--repo ID] [--no-start]
review-watch list [--state S] [--repo ID] [--pr PR] [--all]
review-watch show <id> [--round N] [--draft]
review-watch edit <id>
review-watch accept <id> [--reanchor] [--dry-run] [--approve]
review-watch reject <id>
review-watch rerun <id> [--force]
review-watch resume <id> [--prompt-file F]
review-watch status
review-watch logs <id> [--run]
review-watch cron install|uninstall
review-watch export | prune | browse
```

Every command that prints data takes `--json`. Exit codes follow the design guide (0 success, 2 usage, 10 not found, 11 vetoed, 12 failed, 13 needs a human, 20 refused with a `reason`, 21 ambiguous); the full list is in [`docs/interface.md`](docs/interface.md).

## Configuration

`~/.config/review-watch/config.yaml` (plus `config.d/*.yaml`); every key is documented in [`config.example.yaml`](config.example.yaml). The one required key is `hosts`: nothing runs until you name the hosts to watch. Credentials belong to forgectl; review-watch reads none. Scalar keys can be set with `REVIEW_WATCH_<KEY>` environment variables.

## Hooks

`pre-round` (may veto or escalate), `post-draft`, `post-attention` (one hook with a `reason`: escalated, stale, failed, expired, round limit, spend cap, held, poll failed) and `post-post`. Payloads carry ids, shas, counts and enums, never PR or model text. Wire them to your notifier.

## Dependencies

- `forgectl` (lists review requests and PR facts); binding A (hard)
- `review-run` (runs reviews); binding A (hard)
- `pr-comment` (validates drafts, re-anchors and posts); binding A (hard)
- `gatectl` (optional: auto-post and auto-approve policies, track records); binding A, without it nothing is automatic
- `jobsctl` (optional: the review job queue); binding A, without it reviews run as detached processes under an internal cap

review-run needs bubblewrap and socat on Linux and WSL for its sandbox. A missing required tool exits with its install command.

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/review-watch
```

Then run `review-watch cron install` to poll every five minutes.

## Documentation

- System architecture: [`../awkit-docs/architecture.md`](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- Tool design guide: [`../awkit-docs/design-guide.md`](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- This tool's design: [`docs/design.md`](docs/design.md)
- This tool's interface: [`docs/interface.md`](docs/interface.md)
- Implementation plan: [`docs/plan.md`](docs/plan.md)

## License

MIT — see LICENSE.
