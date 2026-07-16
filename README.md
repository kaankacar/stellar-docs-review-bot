# stellar-docs review bot

An AI agent that reviews pull requests for a docs repo, running entirely in GitHub Actions and
acting under a bot identity. Built for `stellar/stellar-docs`; the workflow is repo-agnostic
(it reads `${{ github.repository }}`), so it drops into any repo.

> Issue triage + auto-fix (assign easy issues to Copilot) lives in a companion repo:
> [stellar-docs-issue-agent](https://github.com/kaankacar/stellar-docs-issue-agent).

This is the exact code demoed on a fork of the docs — see [the three demo PRs](#demo) below.

## How it works — one workflow, two jobs (the safety boundary)

`pr-agent.yml` is a **single workflow** that both reviews and acts, split into two jobs on
purpose:

- **`review`** — runs the model (Claude), reads the pull request, posts one signed comment,
  and *proposes* an action by applying at most one label. Its job token is `contents: read`
  and it is given **no merge/close tools**. This is the half that ingests untrusted PR text.
- **`execute`** — runs **no model**, never reads the PR prose, and acts (merge / close) purely
  from the label the reviewer applied plus the objective check status. Its token has
  `contents: write`.

Because the half that can act never reads attacker-controllable content, a malicious PR
cannot talk its way into being merged. (This mirrors GitHub Agentic Workflows' "safe outputs"
model, rebuilt in plain Actions.)

## What the reviewer decides

Every PR gets a three-pass review — **mechanics** (links/anchors, MDX validity, sidebar
collisions), **technical accuracy** (Stellar-specific: protocol/version/date correctness,
exact network passphrases, RPC-over-Horizon guidance, SDK/CLI validity, SEP/CAP numbers), and
**completeness** (does the diff fully do what it claims / resolve its issue) — then one of:

| Outcome | Label applied | What the executor does |
|---|---|---|
| Trivial + correct + all-checks-green | `auto-merge-candidate` | merges (squash) once every check is green |
| Exact duplicate / fully superseded | `triage:approve-close` | closes with a reasoned comment |
| Judgment-call close | `triage:close-candidate` | nothing — a human confirms with `triage:approve-close` |
| Needs changes / needs human review | none | nothing — left for a maintainer |

Merge/close authority is deliberately conservative and is a policy dial, not a property of the
tooling — start in propose-only mode, widen as trust builds.

## Always-current knowledge

The reviewer `git clone --depth 1`s [stellar-dev-skill](https://github.com/stellar/stellar-dev-skill)
**fresh on every run** and consults the matching `SKILL.md` (standards, data, smart-contracts,
assets, dapp, agentic-payments, zk-proofs). So its Stellar knowledge tracks the source of
truth instead of going stale in a prompt. (Note: the hosted Raven MCP is browser-OAuth only
and is **not** used in CI; the reference facts in `triage-policy.md` were compiled from it and
baked in.)

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
   `triage:close-candidate`, `triage:needs-info`.
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
