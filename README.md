# stellar-docs review bot

An AI agent that reviews pull requests for a docs repo, running entirely in GitHub Actions and
acting under a bot identity. Built for `stellar/stellar-docs`; the workflow is repo-agnostic
(it reads `${{ github.repository }}`), so it drops into any repo.

> Issue triage + auto-fix (assign easy issues to Copilot) lives in a companion repo:
> [stellar-docs-issue-agent](https://github.com/kaankacar/stellar-docs-issue-agent).

This is the exact code demoed on a fork of the docs — see [the three demo PRs](#demo) below.

## How it works — one workflow, split jobs (the safety boundary)

`pr-agent.yml` is a **single workflow** that both reviews and acts, split into jobs on
purpose:

- **`review`** — runs the model (Claude), reads the pull request, posts one signed comment,
  and *proposes* actions by applying labels. Its job token is `contents: read`
  and it is given **no merge/close tools**. This is the half that ingests untrusted PR text.
- **`execute`** — runs **no model**, never reads the PR prose, and acts (merge / close) purely
  from the label the reviewer applied plus the objective check status. Its token has
  `contents: write`.
- **`fix` / `fix-fork`** — the [self-fix](#self-fix) stage: when the reviewer labels a PR
  `pr:autofix`, these apply the small fixes it listed (same-repo PR: a bot commit pushed to
  the PR branch; fork PR: one-click suggested changes). They never merge or close, and
  whatever they produce goes back through `review` before any merge.

Because the half that can act never reads attacker-controllable content, a malicious PR
cannot talk its way into being merged. (This mirrors GitHub Agentic Workflows' "safe outputs"
model, rebuilt in plain Actions.)

## What the reviewer decides

Every PR gets a three-pass review — **mechanics** (links/anchors, MDX validity, sidebar
collisions), **technical accuracy** (Stellar-specific: protocol/version/date correctness,
exact network passphrases, RPC-over-Horizon guidance, SDK/CLI validity, SEP/CAP numbers), and
**completeness** (does the diff fully do what it claims / resolve its issue) — then one of:

| Outcome | Label applied | What happens next |
|---|---|---|
| Trivial + correct + all-checks-green | `auto-merge-candidate` | executor merges (squash) once every check is green |
| Needs only small safe fixes (typo/link/wording) | `pr:autofix` (alongside a NEEDS-CHANGES verdict) | executor ignores it — the [self-fix](#self-fix) stage applies the fixes, then a fresh review decides |
| Exact duplicate / fully superseded | `triage:approve-close` | executor closes with a reasoned comment |
| Judgment-call close | `triage:close-candidate` | nothing — a human confirms with `triage:approve-close` |
| Needs changes / needs human review | none | nothing — left for a maintainer |

Merge/close authority is deliberately conservative and is a policy dial, not a property of the
tooling — start in propose-only mode, widen as trust builds.

## Self-fix

When the *only* problems the review finds are small, unambiguous, low-risk fixes — a typo,
wording, a broken link or anchor, an obvious factual correction (the same bar the companion
issue agent uses for auto-fix) — the reviewer doesn't just ask for them: it applies the
`pr:autofix` label and lists every fix (`path:line` — current → corrected) under a
`### Proposed self-fix` heading in its comment. A separate job then applies **exactly that
list**, nothing more:

- **Same-repo PR** — the `fix` job checks out the PR branch, applies the listed edits,
  commits as the bot, and pushes. The push re-triggers the reviewer (`synchronize`), which
  typically now finds the PR clean and labels it `auto-merge-candidate` — so a fixable PR can
  flow review → fix → re-review → merge without a human, while every merged byte still passed
  a clean review plus green checks. (App-token pushes re-trigger workflows; a plain
  `GITHUB_TOKEN` push doesn't, so without the App the re-review waits for the next push or a
  manual run.)
- **Fork PR** — the bot never pushes to a fork. The `fix-fork` job posts the same fixes as
  GitHub **suggested changes** — a review of inline `suggestion` blocks the author applies in
  one click — falling back to a single comment carrying the suggestion blocks when a fix
  can't be anchored to a diff line. This job deliberately holds no `contents: write` token.

Guard rails: the fix jobs run only on the reviewer's `pr:autofix` label (and clear it when
done); they treat all PR text as untrusted data and apply only the review's listed fixes;
they have no tools that execute repo code; they never merge, close, or approve. A loop guard
skips the fix job whenever the newest commit on the PR is already the bot's own — one fix
round per human push. Anything larger, ambiguous, or opinionated never gets the label and
stays with a human.

## Always-current knowledge

The reviewer `git clone --depth 1`s [stellar-dev-skill](https://github.com/stellar/stellar-dev-skill)
**fresh on every run** and consults the matching `SKILL.md` (standards, data, smart-contracts,
assets, dapp, agentic-payments, zk-proofs). So its Stellar knowledge tracks the source of
truth instead of going stale in a prompt.

For facts neither the baked policy nor the skills can settle — protocol activation
status/dates, release tags/versions, deprecations — the reviewer can also hit the **live
web**: `WebSearch` plus `WebFetch` scoped to stellar.org, developers.stellar.org, and
github.com, citing URL + access date and treating fetched pages as untrusted data. (The
hosted Raven MCP is browser-OAuth only and is **not** used in CI; the reference facts in
`triage-policy.md` were compiled from it and baked in. Live web is the CI stand-in until
Raven has a service token.)

## The rulebook

All behavior lives in [`.github/triage-policy.md`](.github/triage-policy.md) — the single
source of truth both agents read at runtime. Change policy via PR; no code change needed.

## Bot identity — one branded bot instead of github-actions[bot] (optional)

By default the review posts as **github-actions[bot]**. To have it (and the companion issue
agent) act as a single branded **GitHub App** identity like `stellar-docs-bot[bot]`, run the
included one-command setup — two browser clicks total (Create, then Install):

```bash
./setup-github-app.sh   # see SETUP.md
```

It registers the app via GitHub's manifest flow, captures the App ID + private key
automatically (nothing to copy/paste), and sets them as the `APP_ID` / `APP_PRIVATE_KEY`
repo secrets. Each job then mints a short-lived installation token with
`actions/create-github-app-token@v3`, scoped down per job — the `review` job's token still
cannot merge, preserving the safety boundary. If the secrets are absent the workflow falls
back to `GITHUB_TOKEN` and behaves exactly as before.

## Install

1. Copy `.github/workflows/*.yml` and `.github/triage-policy.md` into your repo.
2. Create the labels the policy uses: `P1`, `P2`, `auto-merge-candidate`, `triage:approve-close`,
   `triage:close-candidate`, `triage:needs-info`, and `pr:autofix` (e.g.
   `gh label create "pr:autofix" --color FBCA04 --description "Reviewer found small safe fixes it can apply itself"`).
3. Add auth as a repo secret — either `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) or
   swap the workflows to an org `ANTHROPIC_API_KEY`.
4. Optional: `REPO=owner/name ./setup-github-app.sh` for the branded bot identity (see
   [SETUP.md](SETUP.md)); skip it and the bot posts as `github-actions[bot]`.
5. Open a PR and watch it review within a couple of minutes.

## Demo

Run on a fork of the docs, all actions taken under the `github-actions[bot]` identity:

- **Auto-merged** a clean trivial fix (reviewed → labeled → waited for green CI → merged).
- **Caught & blocked** a subtle error — a PR "updating" the Testnet network passphrase to a
  new year; the bot knew the passphrase is a fixed constant and flagged the inconsistency, so
  it withheld merge (NEEDS-CHANGES).
- **Deferred to a human** on an unverifiable "X is now live" ecosystem claim — neither merged
  nor blocked, just asked for a maintainer's eye.
