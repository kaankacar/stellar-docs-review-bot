# stellar-docs PR review bot

An AI pull-request reviewer that runs entirely in **GitHub Actions**. On every PR it reviews
the diff against the repo, verifies Stellar ecosystem facts against **live sources** (the
Stellar Raven MCP + official docs), and posts one signed review with a clear recommendation —
plus, when enabled, applies labels, auto-merges trivial+correct changes, and can push small
fixes itself. Runs under the `github-actions[bot]` identity. Repo-agnostic (reads
`${{ github.repository }}`), so it drops into any repo.

## How it works — one workflow, two jobs (the safety boundary)
- **`review`** (model: `claude-fable-5`) reads the PR, runs a three-pass review (mechanics →
  technical accuracy → completeness), and *proposes* an action via at most one label. It has
  **no merge/close power**. This is the half that ingests untrusted PR content.
- **`execute`** (no model) takes the action — merge / close — but only from the label plus
  the objective check status, and never reads the PR prose. A malicious PR therefore can’t
  talk its way into being merged.
- **`fix` / `fix-fork`** — for small, unambiguous fixes the reviewer flags, the bot pushes the
  fix to the PR branch (same-repo) or posts one-click suggestions (forks). Every result is
  re-reviewed before any merge.

## Live fact-checking (the differentiator)
The reviewer doesn’t rely on memorized facts. For ecosystem claims it can’t settle from the
repo (protocol activation dates/status, release versions, deprecations), it checks **live**:
1. **Stellar Raven MCP** first (`mcp__raven__search` / `execute`) — dated, citable Stellar
   sources;
2. **web** fallback (stellar.org, developers.stellar.org, github.com);
3. it also clones the current `stellar-dev-skill` per run for topic best-practices.
All external and PR content is treated as untrusted data.

## Behavior is a dial (safe by default)
Everything defaults to **comment + label only** (a “consultant” — no merge/close) so results
can be judged before any autonomy is enabled. Merge/close are gated (trivial + correct + all
checks green for merge; ironclad cases only for close) and can be turned up as trust builds.

## Deploying it — what an admin needs to set
1. Copy `.github/workflows/pr-agent.yml` and `.github/triage-policy.md` into the target repo.
2. Repo **secrets**:
   - `ANTHROPIC_API_KEY` — an **organization API key** (the right choice for a shared repo,
     rather than an individual’s subscription token). The workflow currently authenticates
     with `claude_code_oauth_token`; for an org key, swap that input to
     `anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}` in the three `claude-code-action`
     steps.
   - `RAVEN_MCP_TOKEN` — bearer for the Stellar Raven MCP (live fact-checking).
3. Create the labels the policy uses: `P1`, `P2`, `auto-merge-candidate`,
   `triage:approve-close`, `triage:close-candidate`, `triage:needs-info`, `pr:autofix`.
4. Merge/close autonomy additionally needs the workflow token to be allowed to merge on the
   protected branch — a repo/org permission an admin controls.

## Proof it works (live demos on a fork)
- **Raven catch:** a PR asserting a non-existent “Protocol 28 (live on Mainnet)” →
  NEEDS-CHANGES, the bot citing *“checked via Raven corpus — no dated source for any Protocol
  28 exists.”* https://github.com/kaankacar/stellar-docs/pull/34
- **Auto-merged** a clean trivial fix: https://github.com/kaankacar/stellar-docs/pull/28
- **Caught & blocked** a subtle wrong network-passphrase edit:
  https://github.com/kaankacar/stellar-docs/pull/27
- **Deferred to a human** on an unverifiable claim:
  https://github.com/kaankacar/stellar-docs/pull/26

## The rulebook
All behavior lives in [`.github/triage-policy.md`](.github/triage-policy.md) — the single
source of truth the reviewer reads at runtime. Change policy via PR; no code change needed.
