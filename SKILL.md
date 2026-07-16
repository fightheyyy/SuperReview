---
name: superreview
description: >-
  Use for code review of a pull request, branch, commit range, staged diff, or
  working-tree changes when the user says "review this PR", "review and fix",
  "inspect these changes", "检查这个 PR", "审一下这组改动", or explicitly invokes
  SuperReview. Make the main Codex agent personally establish the review boundary
  and confirm evidence-backed findings before delegating each finding as a
  bounded atomic repair, then personally accept or reject every patch and
  re-review the final effective diff. Keep generic implicit review report-only;
  use the repair loop for explicit `$superreview` or an explicit fix request.
  Compose with SuperDev architecture contracts and SuperGoal child-goal lifecycle
  rules when those skills are active.
---

# SuperReview

Run a reviewer-owned repair loop for pull requests and local change sets. Keep
review judgment, finding quality, patch acceptance, final verification, and the
final verdict with the main agent. Use subagents only as bounded repair workers
after the main agent has confirmed actionable findings.

## Model and Intelligence Policy

Enforce this role-specific model split before substantive review work:

- Run the main reviewer on `gpt-5.6-sol` with **Ultra**. Use this profile for
  task analysis, boundary discovery, the complete initial review, finding and
  priority decisions, repair acceptance, final verification, and the verdict.
- Run every repair worker through the `superreview-repair` custom agent on
  `gpt-5.6-sol` with **Extra High** reasoning (`model_reasoning_effort =
  "xhigh"`).
- Do not let Ultra's proactive delegation bypass the ownership invariant. The
  main agent must still finish the initial review and freeze findings before any
  repair worker starts.

Apply this model gate:

1. Inspect the selected main-session model and intelligence level when the
   surface exposes them.
2. If the main session is not `gpt-5.6-sol` + Ultra, stop before Phase 1 and ask
   the user to select that profile. If the surface cannot inspect or change the
   selection, state `Main model policy unverified`; do not silently claim
   compliance or begin repair mode.
3. Before dispatch, require the named `superreview-repair` custom agent. Select
   that profile explicitly when the spawn surface supports agent types or
   profiles.
4. If the repair profile is unavailable or the spawn surface cannot select it,
   do not substitute a generic worker. Keep the confirmed findings report-only
   and report `Repair model policy blocked` with the missing capability.
5. Let an explicit user model override win, but record the deviation in the
   final report.

The distributable repair profile lives at
`agents/superreview-repair.toml`. Install it as
`~/.codex/agents/superreview-repair.toml` for a personal agent or
`.codex/agents/superreview-repair.toml` for a project-scoped agent.

## Ownership Invariant

The main agent must:

- Resolve the review target and comparison boundary: PR base/head and merge base,
  branch or commit range, index, or working tree. Inventory changed files,
  commits, and relevant repository instructions.
- Personally read the complete diff and enough surrounding code to understand
  behavior, contracts, callers, tests, and failure modes.
- Personally decide which observations are actionable findings and assign their
  priority.
- Write every repair contract, review every returned patch, run final checks,
  and decide whether the reviewed change set is acceptable.

Do not delegate the initial review, finding discovery, severity decisions, or
final acceptance. Do not ask a subagent to "review the PR", "review these
changes", "find bugs", or choose its own repair scope. A subagent may reject its
assigned finding as a false positive when it returns concrete evidence and makes
no speculative edit; the main agent decides whether to withdraw the finding.

## Mode Selection

Use these modes:

- **Review and repair:** Default when the user explicitly invokes
  `$superreview` on a review target, asks SuperReview to handle it, or explicitly
  asks to review and fix. Review first, then repair confirmed findings through
  subagents.
- **Report only:** Use when the user says to review, inspect, audit, or comment
  without changing code. Produce findings and verification limits; do not edit,
  spawn repair workers, commit, push, or publish a review.
- **Repair existing findings:** Use when the user supplies an already accepted
  finding list and explicitly asks for fixes. The main agent must validate each
  finding against the current review target before dispatching it; do not repeat
  a broad review unless final acceptance requires it.

If SuperReview was only triggered implicitly by a generic review request, keep
the run report-only unless the user also authorized changes.

Never post GitHub comments, submit approve/request-changes reviews, resolve
threads, commit, push, merge, or open another PR unless the user explicitly
requests that external action. Local repair authorization does not imply
publication authorization.

## Phase 1: Establish the Review Boundary

Before judging the review target:

1. Read repository and directory-scoped instructions.
2. Classify the target as a PR, branch comparison, commit range, staged changes,
   working-tree changes, or a combination explicitly requested by the user.
3. Freeze the exact comparison boundary. For a PR, prefer its actual merge base
   over an assumed branch name. For local changes, distinguish `HEAD`, index, and
   working tree so nothing is silently omitted or included.
4. Inventory commits, changed files, generated files, migrations, tests, and
   relevant CI state.
5. Preserve unrelated user changes. Do not overwrite a dirty worktree or absorb
   out-of-scope local edits into the review target.
6. Read the PR description or supplied requirements, plus relevant `SPEC.md` /
   `PLAN.md` files when present.
7. Record any unavailable context that limits confidence.

Review the change set's effect, not only its textual diff. Follow changed
interfaces to callers and consumers, inspect nearby invariants, and compare
tests with the behavior they claim to protect. Do not turn unrelated
pre-existing defects into findings unless the reviewed changes make them
reachable, worse, or newly relevant.

## Phase 2: Perform the Main-Agent Review

Inspect all changed behavior before dispatching any repair. Prioritize:

- Correctness, data loss, security, permissions, privacy, and concurrency.
- Contract, schema, migration, compatibility, and rollback safety.
- Boundary conditions, failure handling, idempotency, resource cleanup, and
  state transitions.
- Cross-module callers, configuration, packaging, deployment, and generated
  artifacts.
- Missing or misleading tests for material changed behavior.
- Violations of the repository's declared current or target architecture.

Avoid findings based only on taste, speculative future work, formatting already
enforced by tooling, or claims that cannot be tied to a concrete failure mode.

For each finding, record:

- Stable ID such as `SR-001`.
- Priority: `P0` release-blocking, `P1` high, `P2` normal, or `P3` low.
- Tight file and line location in the changed code when possible.
- Observable failure or violated contract.
- Evidence and the conditions required to reproduce the problem.
- Why the reviewed changes caused or exposed it.
- Smallest acceptable repair boundary.
- Focused verification that would close the finding.

Keep findings independent. Split distinct root causes. Combine symptoms only
when one atomic repair necessarily closes them together.

## Phase 3: Freeze Findings and Dispatch Atomic Repairs

Finish the initial review pass before spawning repair workers. If there are no
actionable findings, skip dispatch and proceed to final verification.

Dispatch one bounded repair contract per independent finding. Run contracts in
parallel only when their write sets are disjoint; otherwise run them
sequentially. The main agent must not edit the same files concurrently with a
repair worker.

Start every repair message with `Repair:` and include:

```md
Repair: <finding ID and one-sentence objective>
Parent Review: <review-target identity, comparison boundary, and priority>
Worker Profile: superreview-repair (gpt-5.6-sol, xhigh / Extra High)
Finding Evidence: <failure, cause, location, and reproduction conditions>
Acceptance: <observable condition that closes this finding>
Allowed Scope: <exact files/modules the worker may inspect or edit>
Forbidden Work: <unrelated cleanup, broad refactors, API changes, extra files>
Required Verification: <focused tests/checks to run>
Expected Output: <patch summary, files changed, commands/results, residual risk>
Parent Merge Plan: <how the main agent will inspect and accept or reject it>
Worker Rules: Validate the finding, make the smallest repair, do not broaden
scope, do not spawn agents, and do not claim PR-level acceptance. If the finding
is false or cannot be fixed inside scope, make no speculative edit and return
evidence to the parent.
```

When SuperGoal is also active, embed this repair contract inside its full child
Goal Contract and require the child `create_goal` / `update_goal` lifecycle.
SuperReview's `Repair:` boundary remains mandatory; SuperGoal adds lifecycle
tracking but does not delegate review ownership.

If subagent tools or the required repair profile are unavailable, stop repair
dispatch and state the blocker. Do not replace the required Extra High worker
with a generic subagent or a silent main-agent implementation.

## Phase 4: Accept or Reject Every Repair

Treat a subagent patch as untrusted input. For each returned repair, the main
agent must:

1. Re-read the original finding and inspect the complete patch.
2. Confirm the patch stays inside allowed scope and does not overwrite other
   work.
3. Reproduce or reason through the original failure and verify the root cause is
   closed rather than hidden.
4. Inspect new edge cases, compatibility effects, test quality, and accidental
   complexity.
5. Run or independently repeat the focused verification when feasible.
6. Mark the finding `accepted`, `rejected`, `withdrawn`, or `blocked`, with
   evidence.

Do not silently repair a rejected patch on top of concurrent worker changes.
Issue a narrower follow-up repair contract or integrate the fix only after the
write scope is safe.

## Phase 5: Re-review the Final Change Set

After accepted repairs are integrated, personally review the final effective
change set again. For a PR, include merge base through PR head plus every
accepted uncommitted worktree repair. For local review, include the frozen
commit/index/worktree boundary plus accepted repairs. Do not rely on
`base..HEAD` alone when worker patches are uncommitted. The final pass must
include all worker-created changes and check that fixes do not conflict with one
another.

Run the strongest proportionate verification available:

- Focused regression tests for every accepted finding.
- Relevant unit, integration, type, lint, build, migration, or smoke checks.
- Targeted searches for stale call sites, duplicate contracts, skipped cases,
  debug code, or generated artifacts.
- Manual or UI checks when changed behavior cannot be proven by automation.

If the final pass finds a new actionable defect, create a new finding and repeat
the atomic repair loop. Stop only when:

- The final diff has no unresolved actionable findings within review scope.
- Every confirmed finding is accepted, withdrawn with evidence, or explicitly
  blocked.
- Verification passes, or any unavailable/failed check is clearly reported.
- SuperDev docs are consistent with the final implementation when the reviewed
  changes alter durable architecture.

Do not claim a clean review merely because tests pass.

## SuperDev Composition

Use relevant SuperDev `SPEC.md` and `PLAN.md` files as review contracts when they
exist. Continue the review even when architecture docs are missing or stale,
but report mismatches that create concrete PR risk. Before accepting an
architecture-changing repair, require a clear target architecture and keep
implementation and docs synchronized. Do not create architecture ceremony for
a local defect fix that does not change durable boundaries.

## Final Report

Lead with the verdict, then report:

- Findings by priority, each with status and tight code references.
- Repairs delegated, accepted, rejected, withdrawn, or blocked.
- Verification commands and outcomes.
- Model-policy evidence for the main reviewer and repair workers, including any
  unverified or user-overridden selection.
- Residual risks, missing context, and unrun checks.
- Publication state: local only, or the exact explicitly authorized GitHub/git
  actions completed.

When no findings remain, say so directly but still disclose verification gaps.
Keep the report focused on evidence and decisions rather than narrating every
tool call.
