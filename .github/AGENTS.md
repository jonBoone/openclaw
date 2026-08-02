# .github/AGENTS.md

Telegraph style. CI/CD, review policy, GitHub operations, and release mechanics.

## CI/CD

- CI polling: exact SHA, relevant checks only, minimal fields. Skip routine noise (`Auto response`, `Labeler`, docs agents, performance/stale). Logs only after failure/completion or concrete need. Never `gh run watch`; its 3s polling exhausts API quota. Use sparse GraphQL rollups. Filter `gh run list` by workflow/branch/commit; broad JSON lists can exceed relay caps. Exact-SHA fallback dispatches require the full 40-character SHA.
- CI waits: `node scripts/watch-pr-ci.mjs <pr> <head-sha>` — prechecks mergeable (CONFLICTING = pull_request CI cannot attach) and run attachment before polling; watchers emit every terminal state; no unbounded polls.
- Trusted-workflow release-branch CI: pass `target_ref` + `release_candidate_ref`; never `release_gate` (requires workflow head == target).
- `release-ci-summary` accepts Full Release Validation parent runs only. Diverged release-branch logs: `--first-parent` plus a bounded count.

## GitHub / PRs

- Fresh GitHub items: read `CONTRIBUTING.md`, the issue chooser/form, PR template, and `.github/CODEOWNERS`; blank issues are disabled; preserve templates and evidence requirements.
- Issue first for bugs, user-facing features, architecture/product decisions, or work needing durable discussion. Bounded maintainer-requested refactor may go direct; agent decides whether an issue adds value. PRs use the template, link context, and keep durable problem/impact/evidence sections.
- Route support to Discord and security through `SECURITY.md`. Use listed maintainer areas/`CODEOWNERS`; never guess mentions.
- Use `$openclaw-pr-maintainer` immediately for maintainer-side OpenClaw issue/PR review, triage, duplicates, labels, comments, close, land, or evidence.
- Issue/PR start: `git status -sb`; if clean, `git pull --ff-only`; if dirty, yell before pull/rebase.
- PR refs: `gh pr view/diff` or `gh api`, not web search. Prefer `gitcrawl` for maintainer discovery; missing/stale `gitcrawl` falls through to live `gh`. Verify live with `gh` before mutation.
- No unsolicited PR labels/retitles/rebases/fixups/landing. Comments/reviews ok only for reviewable findings, pre-merge proof, or close/duplicate reason after explicit request.
- Maintainer decision closes the cluster: if deciding reported behavior/proposed fix is not planned, comment+close all directly associated open issues/PRs. Close comment states: decision, why, supported alternative, and what evidence would change the decision.
- Issue/PR work: search strong related issues/PRs before final; close proven dupes/fixed siblings.
- PR superseded by `main`: if code proof shows `main` already has same-or-better behavior, comment canonical commit/PR + focused proof, then close.
- Issue/PR numbers need a short summary every time.
- PR review answer: bug/behavior, URL(s), affected surface, provenance for regressions when traceable, best-fix judgment, evidence from code/tests/CI/current or shipped behavior.
- PR reviewable findings: post them on the PR, not chat-only.
- Issue/PR final answer: last line is the full GitHub URL.
- PR verification: before merge, post land-ready work done, exact local commands, CI/Testbox run IDs, before/after proof when used, and known proof gaps.
- After PR merge/ship: concise prose recap; cover behavior, key surface, proof, and issue/PR state.
- Public GH comments: show draft in chat first unless user explicitly asked to post.
- No surprise GH writes: chat must mention every posted/updated public comment with URL.
- GH comments with backticks, `$`, or shell snippets: use heredoc/body file, not inline double-quoted `--body`.
- PR create: real body required. Template: `What Problem This Solves`, `Why This Change Was Made`, `User Impact`, and `Evidence`.
- PR create races GitHub's merge-ref computation: `gh pr create --draft`, poll `mergeable` non-null, then `gh pr ready`.
- PR create/refresh: keep PR branches takeover-ready. Enable `maintainer_can_modify` unless unsafe.
- PR artifacts/screenshots: attach to PR/comment/external artifact store. Never push to repo branches.
- Agent PR landing to `main`: use `scripts/pr` wrapper. Run `review-init`, `review-artifacts-init`, `review-validate-artifacts`, then `OPENCLAW_TESTBOX=1 scripts/pr prepare-run` and `scripts/pr merge-run`.
- Non-main PRs: do not run `scripts/pr prepare-run` or `merge-run`. Use review artifacts, exact base-head CI, then `gh pr merge --match-head-commit <verified-sha>`.

## Security / Release

- Never commit real phone numbers, videos, credentials, live config.
- Secrets: channel/provider creds in `~/.openclaw/credentials/`; model auth profiles in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.
- SecretRef failures isolate to the smallest known owning surface. Gateway refuses startup only when ingress protection cannot be established, config is structurally invalid, or owning surface is unknown.
- Dependency patches/overrides/vendor changes need explicit approval. `pnpm-workspace.yaml` patched dependencies use exact versions only.
- `pnpm-lock.yaml` is the product dependency security review surface.
- Carbon pins owner-only: do not change `@buape/carbon` unless Shadow (`@thewilloftheshadow`) asks.
- Releases/publish/version bumps need explicit approval. Use `$release-openclaw-maintainer`.
- Release versions use `YYYY.M.PATCH` (sequential monthly train, not calendar day).
- Regular beta/stable flow: two immutable identities — **Code SHA** (product validation) and **Release SHA** (diff is exactly `CHANGELOG.md`).
- Never generate changelog before Code SHA has green Full Release Validation.
- Release-SHA proof is narrow: release-note/provenance checks, npm preflight, install/update acceptance, publish readiness.
- GHSA/advisories: `$openclaw-ghsa-maintainer` / `$security-triage`. Secret scanning: `$openclaw-secret-scanning-maintainer`.
- Beta tag/version match: `vYYYY.M.PATCH-beta.N` -> npm `YYYY.M.PATCH-beta.N --tag beta`.
