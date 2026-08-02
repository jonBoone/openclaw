# test/AGENTS.md

Telegraph style. Test and validation rules for `test/`, `*.test.ts`, `*.e2e.test.ts`, and QA scenarios.

## Test Conventions

- Vitest. Colocated `*.test.ts`; e2e `*.e2e.test.ts`; example models `sonnet-4.6`, `gpt-5.6-luna`; test GPT with Luna preferred; use Sol when capability matters; no GPT-4.x agent-smoke defaults.
- Test where the bugs live: boundaries, not internals. Coverage behind mocks proves the mocks; one test through the real transport/dispatch seam outranks many stub-backed branch tests.
- Prefer invariant assertions (every input accounted for; every action ends in a visible outcome or recorded non-outcome) over enumerating happy paths.
- Inject faults — network, provider, ordering, restart — instead of asserting only success shapes. Changes to delivery, dispatch, or session paths need at least one boundary-level proof (harness or live), not only unit tests of the changed function.
- Shared-state/order failures: reproduce original execution order, repair the writer or lifecycle owner, and add boundary regression coverage. Use tracked environment helpers; never mask producer leaks with consumer-only environment overrides.
- Prefer behavior tests over workflow/docs string greps. Put operator policy reminders in AGENTS/docs.
- A test asserting on files owned by lane X belongs in lane X's suite. A cross-lane assertion may never be selected by PR change classification, so it passes PR CI and first breaks on `main` full runs.
- QA scenario sources are YAML only: `qa/scenarios/index.yaml` and `qa/scenarios/<theme>/*.yaml`. Do not add fenced `qa-scenario`/`qa-flow` Markdown files under `qa/scenarios/`.
- Clean timers/env/globals/mocks/sockets/temp dirs/module state; `--isolate=false` safe.
- Tests asserting resolver/root-containment paths: `fs.realpath` mkdtemp/tmp roots first. macOS `os.tmpdir()` is a `/var` -> `/private/var` symlink; prod resolvers return canonical paths, so raw mkdtemp assertions pass on Linux CI but fail on Mac.
- Explicit `vi.mock` factories must export every binding prod touches, including error classes used in `instanceof` checks; `vi.importActual` the defining module for those instead of stub classes.
- Prefer injection and narrow `*.runtime.ts` mocks over broad barrels or `openclaw/plugin-sdk/*`.
- Do not edit baseline/inventory/ignore/snapshot/expected-failure files to silence checks without explicit approval.
- Do not run independent `pnpm test`/Vitest commands concurrently in one worktree; Vitest cache races with `ENOTEMPTY`. Group one command or use distinct `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH`.
- Never edit source/test files while a Vitest run is in flight in the same checkout; mid-collection reads produce phantom failures and 120s timeouts. Wait for the run to finish, then edit.
- Vitest rejects Jest `--runInBand`; use `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test` for serial proof. Test workers max 16.
- Live: `OPENCLAW_LIVE_TEST=1 pnpm test:live`; verbose `OPENCLAW_LIVE_TEST_QUIET=0`.
- Live gateway tests: session-owned dev gateway only — isolated `OPENCLAW_STATE_DIR` + free port. Never bind the operator's real gateway port (default 18789) while their gateway runs.
- Never stop/restart/kickstart a gateway service you did not start (launchd/systemd/tmux) or edit its live `~/.openclaw` state/config; that is the operator's running instance — explicit per-task operator approval required.
- Realistic data: copy the state/DB into your dev state dir and test the copy. In-place migration of a live gateway's state needs explicit operator approval.
- Guide: `docs/reference/test.md`.

## Test Commands

- Trusted-source focused local proof: `node scripts/run-vitest.mjs <path-or-filter>`.
- Remote or normal-checkout proof: `pnpm test <path-or-filter> [vitest args...]`, `pnpm test:changed`, `pnpm test:serial`, or `pnpm test:coverage`.
- Never raw `vitest`; if unavoidable, `vitest run ...` (bare `vitest` starts local watch mode and never exits).
- No `--repeat`; use a bounded shell loop around the focused repo test command.
- Local agent test execution is allowed only for trusted source and one/few focused files when the existing dependency install is ready. In a Codex worktree or linked/sparse checkout, use `node scripts/run-vitest.mjs <path-or-filter>`; never direct local `pnpm test*`, and never reconcile dependencies merely to keep proof local.
- Extension tests: `pnpm test:extensions`, `pnpm test extensions`, `pnpm test extensions/<id>`.

## Validation

- Use `$openclaw-testing` for test/CI choice and `$crabbox` for remote/full/E2E proof.
- Proof routing: source trust first, proof size second. Trusted source runs one/few focused tests, `git diff --check`, targeted formatting, and cheap static probes locally when the existing dependency install is ready. Heavy proof — full suites, changed gates with typecheck/lint fan-out, builds, Docker, packaging, E2E, live, cross-OS, or anything computationally intensive — goes to Crabbox/Testbox; trusted maintainer heavy proof defaults to Blacksmith Testbox. If the remote backend is unavailable (broker/DNS/network/lease failure), trusted-source proof falls back to local execution — including heavier suites and gates — instead of blocking; note the fallback and reason in the proof summary.
- Untrusted (contributor/fork) source: never run its scripts, tests, checks, wrappers, config, or package hooks locally, regardless of proof size, and never fall back to local. Use secretless fork CI or sanitized direct AWS Crabbox, never a credential-hydrated Testbox. Maintainer approval of credentialed execution after review makes it trusted; an explicit owner/maintainer instruction to land named, reviewed PRs is that approval — do not ask twice.
- Sanitized AWS Crabbox procedure: launch an installed trusted Crabbox binary from a clean trusted `main` checkout; fetch only the remote PR via `--fresh-pr`; never execute a wrapper, config, or command from the untrusted local checkout. Upload trusted `scripts/crabbox-untrusted-bootstrap.sh` from clean `main` — it proves the remote IMDSv2 IAM credentials endpoint returns 404, verifies the reviewed head SHA, unsets `NODE_OPTIONS`, installs pinned Node/pnpm, verifies the package-manager pin, isolates `HOME`, installs dependencies, then runs the requested test. Before warmup: unset `CRABBOX_AWS_INSTANCE_PROFILE` and all `CRABBOX_TAILSCALE*` overrides; fail closed unless resolved `aws.instanceProfile` is empty; force `--network public --tailscale=false`, clear exit-node/LAN flags; require `crabbox inspect` to report public networking with no Tailscale state before any script. Use a newly warmed lease bound to one reviewed head SHA with `CRABBOX_ENV_ALLOW=CI` and `--no-hydrate`; never reuse a trusted/previously hydrated lease or carry an untrusted lease across head revisions — stop and rewarm when the SHA changes. No repo `OPENCLAW_*` allowlist, existing auth profile, instance role, tailnet/LAN access, moving PR head, or ambient Node preload may reach untrusted execution.
- Leases: do not pre-warm at task start. Acquire the backend lazily at the first heavy proof, reuse that one lease (sync the current checkout for every run), then stop it before handoff. If local proof fans out or becomes expensive, stop it and acquire the remote box.
- Testbox mechanics: warm from the task checkout; ownership is checkout-path scoped; `--reclaim` only for intentional transfer, and it does not retarget the remote checkout — never cross repos. One lease, one active command; never sync/reclaim during a run; base/head changed means stop and rewarm — never override stale lease checks. Warmup must print a lease id; silent success is unusable — verify before reuse, else fall back to one-shot `run`. Wrapper reuse requires its local SSH key; missing after restart/handoff means warm fresh. Direct lease: `blacksmith testbox run`; Crabbox wrapper reuse needs a wrapper-created lease. Status/stop: `blacksmith testbox status|stop --id <tbx_id>` — id is not positional, no status `--json` flag. Delegated runs reject `--fresh-pr` and `--stop-after`; sync current checkout, workflow owns lifecycle. Compound commands: `bash -lc`, never `sh -lc`; job env uses Bash `declare`. Testbox owns Chromium; never pass Crabbox `--browser` to `provider=blacksmith-testbox`.
- Crabbox mechanics: a Crabbox request means real scenario proof — install/update/call/repro the user path, not just copied tests run remotely. Final timing JSON = proof complete; if portal sync hangs after it, interrupt the wrapper only. Wrapper `stop` has no `--timing-json`; use `node scripts/crabbox-wrapper.mjs stop --provider <provider> --id <id>`. Sparse-sync temp checkout may claim a kept Testbox; repo-path reuse needs `--reclaim`. Dirty-sync generator proof: compare hashes before/after; `git diff` includes the synced patch.
- Visual proof: use Crabbox, set up like a user, then screenshot-verify. No harness/bypass/shortcut unless explicitly asked.
- In Codex or linked worktrees, direct local `pnpm test*`, `pnpm check*`, `pnpm crabbox:run`, and `scripts/committer` can trigger pnpm dependency reconciliation or install prompts. Prefer `node` wrappers locally and Crabbox/Testbox for pnpm-gated proof.
- Repo-native PR worktree may omit `node_modules`; prove remotely, then use `git commit --no-verify`, not `scripts/committer`.
- Release-branch formatting: Testbox or existing binary; never local `pnpm exec` reconciliation. Targeted local format/lint: existing `./node_modules/.bin/*`; never `pnpm exec` reconciliation.
- Parallel agents share the checkout; never switch its branch while sibling work runs.
- QA CLI `--output-dir` must be repo-relative.
- Before handoff/push: prove touched surface. Before landing to `main`: proof matches actual risk. Bounded behavior-neutral refactor: focused tests/checks enough; no issue proof or full/broad suite by default.
- Release-branch full validation: freeze the product-complete **Code SHA**, then use `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH`; no raw dispatch without `target_context_ref`.
- Pre-land/pre-commit code changes: mandatory fresh `$autoreview` until no accepted/actionable findings remain. Do not land code on CI, ClawSweeper, prior review comments, or your own manual review alone unless user explicitly opts out or scope is truly trivial/docs-only. If findings want refactor, refactor; no ugly fixes. Autoreview staged/uncommitted diff: `--mode uncommitted`; there is no `dirty` or `staged` mode.
- If proof is blocked, say exactly what is missing and why.
- Do not land related failing format/lint/type/build/tests. If unrelated on latest `origin/main`, say so with scoped proof.
- Landing PR onto red `main` (unrelated breakage blocks the merge gate): fix the breakage in the same landing PR; note it in the PR body; never land onto red or bypass the gate. Prefer the smallest correct fix (e.g. register a missing source file, add a dropped export).
- Docs/changelog-only and CI/workflow metadata-only: `git diff --check` plus relevant docs/workflow sanity; escalate only if scripts/config/generated/package/runtime behavior changed.
- Prompt snapshots: CI truth is Linux Node 24. If macOS local passes but CI drifts, reproduce/generate in Linux before rerun.
