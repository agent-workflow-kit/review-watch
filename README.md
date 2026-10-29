# review-watch

Run reviews on incoming review requests unattended, as drafts you accept.

## Status

Pre-design: interface not yet specified; see docs/design.md.

## Layer

L3 (workflow)

## What it does

- Considers only PRs that request your review directly (team requests are ignored); skips draft PRs, bot-authored PRs and PRs you already reviewed yourself.
- Keys queue items on (PR, head sha, review-request instance) and runs `review-run run` for each new item, via jobsctl or as detached processes; `poll` never blocks on a review.
- Copies each review artifact into its own editable review draft; marks drafts stale when the head moves; `accept` posts through pr-comment.
- Computes a deterministic risk tier; auto-post and auto-approve go through gatectl (shadow first) and built-in hard rules.
- New commits never start a review on their own; a fresh review request or an explicit `rerun` does, up to a configurable per-PR round limit; optional daily spend cap.

## Indicative commands

Indicative, finalized in `docs/interface.md`.

```text
review-watch poll
review-watch list
review-watch show <id>
review-watch accept <id> [--reanchor|--dry-run]
review-watch reject <id>
review-watch rerun <id>
review-watch status
review-watch logs <id>
review-watch browse
```

## Dependencies

- `forgectl` (lists review requests and PR facts); binding: TBD (A/B/C), decided in this tool's design session
- `review-run` (runs reviews); binding: TBD (A/B/C), decided in this tool's design session
- `pr-comment` (posts accepted reviews; re-anchoring); binding: TBD (A/B/C), decided in this tool's design session
- `pr-watch` (optional); binding: TBD (A/B/C), decided in this tool's design session
- `gatectl` (optional: auto-post / auto-approve policies); binding: TBD (A/B/C), decided in this tool's design session
- `jobsctl` (optional: review job queue); binding: TBD (A/B/C), decided in this tool's design session

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/review-watch
```

## Documentation

- System architecture: [`../awkit-docs/architecture.md`](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- Tool design guide: [`../awkit-docs/design-guide.md`](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- This tool's design: [`docs/design.md`](docs/design.md)
- This tool's interface: [`docs/interface.md`](docs/interface.md)

## License

MIT — see LICENSE.
