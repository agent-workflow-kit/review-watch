# review-watch: design

Status: Seeded from awkit architecture v1.4; not yet designed.

Section references (§N) point to `../awkit-docs/architecture.md` unless marked "design guide" (`../awkit-docs/design-guide.md`).

## Role and layer

Run reviews on incoming review requests unattended, as drafts you accept.

Layer 3 (workflow). Must work fully standalone; no upward awareness.

## Responsibilities

From §5.3 (`review-watch`), verbatim:

Watches for review requests, runs reviews unattended and holds each result as a draft you accept.

- Commands: `poll`, `list`, `show <id>` (a queue item), `accept <id> [--reanchor|--dry-run]`, `reject <id>`, `rerun <id>`, `status` (summary of the queue and today's runs, plus gate track records from `gatectl stats` when gatectl is configured), `logs <id>` (run log), `browse`.
- Uses: forgectl, review-run, pr-comment; optionally pr-watch, gatectl and jobsctl.
- Notes:
  - The behaviors below (direct-request trigger, skips, risk tiers, shadow mode, staleness, round limit) are generic design decisions of this project; prior art is cited in review-watch's design doc.
  - Considers only PRs that request your review directly; team-level requests are ignored. Skips draft PRs, bot-authored PRs and PRs you already reviewed yourself (a review by your own account carrying the `<!-- pr-comment:v1 -->` marker was posted through awkit and does not count).
  - Keys its queue items on (PR, head sha, review-request instance) and enqueues only for a new item. It runs `review-run run <pr> --label rw-item=<id>` via jobsctl (queue `reviews`, `--unique-key review:<pr>@<sha>` as a second safety net) or, without jobsctl, as detached processes; `poll` never blocks on a review. `poll` resolves held review jobs once the run is done or failed, or the item is rejected.
  - Copies the review-run artifact into its own review draft, which is the editable, authoritative copy for posting. Marks review drafts stale when the head moves; `accept --reanchor` re-anchors through `pr-comment review --reanchor-to <sha>`. `accept` is a human action and passes `--yes --actor human:<user>` to pr-comment.
  - New commits never start a review on their own; a fresh review request or an explicit `rerun` does, up to a configurable per-PR round limit.
  - **Risk tier** is computed deterministically from PR facts (paths, size, author, repo rules, CI); model output can move a PR to a stricter tier, never a looser one. Auto-post and auto-approve go through gatectl (shadow first, section 12.3); an auto-approve policy must include a script check that ignores model output. It passes `--approve` to pr-comment only when an enforced auto-approve gate allows and the hard rules pass.
  - **Built-in hard rules** (the public minimum, applied regardless of gate decisions): never auto-approve your own PRs or bot PRs; never auto-post or auto-approve unless CI has passed on the PR's current head; never auto-act on a PR that touches an off-limits path (a configurable glob list whose default covers security-sensitive code, CI and build pipeline definitions, agent configuration and instruction files, and review-ownership files).
  - **Gate outcomes:** when you act on a gated item, it records `gatectl outcome <id> agree` if your action matches the gate's (would-be) decision and `disagree` otherwise (e.g. the gate would auto-post and you edit or reject the draft).
  - Has an internal concurrency cap so it works without jobsctl; its built-in hard rules let it work without gatectl. Optional daily spend cap, off by default (section 20).
  - Hooks (indicative): `post-draft`, `post-pending`.

## Uses

- `forgectl` (lists review requests and PR facts); binding: TBD (A/B/C), decided in this tool's design session
- `review-run` (runs reviews); binding: TBD (A/B/C), decided in this tool's design session
- `pr-comment` (posts accepted reviews; re-anchoring); binding: TBD (A/B/C), decided in this tool's design session
- `pr-watch` (optional); binding: TBD (A/B/C), decided in this tool's design session
- `gatectl` (optional: auto-post / auto-approve policies); binding: TBD (A/B/C), decided in this tool's design session
- `jobsctl` (optional: review job queue); binding: TBD (A/B/C), decided in this tool's design session

## Key flow: §10.1 Review request (flow A, priority slice)

Verbatim from the architecture:

0. When jobsctl is used, `jobsctl run` runs from cron (or as a loop).
1. Cron runs `review-watch poll`.
2. forgectl lists PRs that request your review directly; draft PRs, bot-authored PRs and PRs you already reviewed yourself are skipped.
3. For each new queue item (PR, head sha, review-request instance), review-watch enqueues a jobsctl job (queue `reviews`, `--unique-key review:<pr>@<sha>`) or, without jobsctl, spawns a detached process under its internal cap, running `review-run run <pr> --mode headless --label rw-item=<id>`.
4. review-run pre-fetches diff and metadata into a fenced prompt file. If `run_pr_code: true`, it first runs the throwaway-checkout sequence of section 13.2 rule 9 and appends the fenced command output to that file. It then calls `agent-spawn launch --mode headless --kind review --cwd session-dir --skill review-run:review-pr --plugin-dir <bundled> --worktree none --prompt-file <file> --permission-profile <bundled file> --env-allowlist-only <vars> --label owner=review-run --wait` (section 13.1). The session never has a checkout.
5. The session reports done with the review artifact. review-run validates it against the review-draft schema, stores the run with its own copy of the artifact, and tears down the agent; review-watch copies the artifact into its review draft; its `post-draft` hook calls notice-route (Slack/Home Assistant).
6. You accept (CLI or TUI). If the head moved, the review draft is stale (`rerun`, or `accept --reanchor` to re-anchor onto the new head via pr-comment). Otherwise pr-comment posts one review (COMMENT, or REQUEST_CHANGES per the draft) with inline comments and an attribution footer, and records an audit entry.
7. Optional: gatectl decides auto-post / auto-approve, in shadow mode first, within the hard rules (section 5.3).
8. A new review starts only on a fresh review request or an explicit `rerun`, up to the configured per-PR round limit.
9. If the review session escalates, the run is `escalated`, its job is held and its agent is kept. Once the agent is resumed and the run is done or failed (or the item is rejected), `poll` resolves the held job.

## Relevant cross-cutting sections

- §3.2, §3.3: layering rules and edge bindings
- §9.3: post-exit policies and held jobs
- §10.1: review request flow (flow A)
- §10.4: attention and notifications (flow D)
- §12: autonomy and gates: public defaults (§12.1), gates and shadow mode (§12.3), confirmations and `--yes` (§12.4)
- §13.2: prompt injection: fencing, transport, trust tiers
- §14: audit, attribution footers and the `<!-- pr-comment:v1 -->` marker
- §15: notifications (`post-draft`, `post-pending`)
- design guide §4.3, §4.4, §10, §12: concurrency and cron; confirmations; TUI; safety

## Constraints (decided)

From §20 (`review-watch`):

queue items are keyed on (PR, head sha, review-request instance). The spend cap uses the local timezone; spend is the sum of headless `total_cost_usd`; promoted interactive time is not counted.

## Open topics

From §20 (`review-watch`):

review state machine (queued → running → drafted → posting → posted → closed, plus failed), risk tier rules, rounds, staleness, shadow mode, config.

## Decisions

| Decision | Rationale | Date |
|---|---|---|

## Questions for the design session

1. What is the review state machine (queued → running → drafted → posting → posted → closed, plus failed), including transitions and persistence?
2. What are the risk tier rules, and which PR facts feed them?
3. What counts as a round, and how is the per-PR limit applied?
4. How is staleness detected and surfaced?
5. How does shadow mode work end to end, and how are gate track records surfaced in `status`?
6. What is the config shape (repos, off-limits paths, caps, spend cap)?

## Prior art and references

The risk-tier and shadow-mode ideas above are common practice; these public sources informed them. The values this tool ships with (round limit, caps, tier rules, off-limits globs, thresholds for recommending a gate) are its own conservative defaults and are expected to be tuned per user and per repo.

- **Risk-stratified review automation.** Chris Adams et al., "Automating Low-Risk Code Review at Meta: RADAR, Risk Calibration, and Review Efficiency", arXiv:2605.30208 [cs.SE], 2026. <https://arxiv.org/abs/2605.30208>. Describes a staged funnel (eligibility gates, static heuristics, a learned diff risk score, LLM review, deterministic validation) that automates only low-risk changes and routes the rest to people, with a tunable risk threshold. This tool borrows the general shape: cheap deterministic checks first, model output that can only make the outcome stricter, and automation limited to the lowest tier. It does not implement a learned risk score.
- **Change-level risk prediction.** Yasutaka Kamei et al., "A Large-Scale Empirical Study of Just-in-Time Quality Assurance", IEEE Transactions on Software Engineering 39(6), 2013. <https://doi.org/10.1109/TSE.2012.70>. Background for scoring a change from its own facts (size, files touched, history) rather than from the whole codebase.
- **Shadow mode before enforcement.** Running a new decision path alongside the existing one and comparing results before it is allowed to act (often called a dark launch or shadow deployment) is general release practice; see Pete Hodgson, "Feature Toggles (aka Feature Flags)", martinfowler.com. <https://martinfowler.com/articles/feature-toggles.html>. Here the comparison is between a gate's would-be decision and what the user actually did (gatectl track records, §12.3).
