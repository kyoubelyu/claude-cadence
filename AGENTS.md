# AGENTS.md

Behavioral guidelines and multi-agent process rules for this codebase — Codex-native edition. The engine is Codex throughout: the orchestrator is Codex's main loop and spawns its own subagents for each gate. Structurally this mirrors `CLAUDE.md`, minus the cross-engine delegation (no "codex rescue"): every role is a normal Codex subagent.

## Project Scope

Write one precise, concise paragraph describing the repository's current scope: what the product ships, who it serves, and the core product and technical boundaries agents must preserve. Keep only durable present-tense facts needed for implementation decisions; do not turn this section into a decision log, roadmap, dated history, retired-direction archive, or exhaustive component inventory. Resolve scope ambiguity during Step 0 exploration (§2 → Phase Gate Matrix, Step 0), and record the resulting live decision in `ROADMAP.md`, not here.

---

## How to use this doc

| Situation | What to read |
| --- | --- |
| Simple task (one obvious edit) | Project Scope + §1 Coding Behavior → dispatch to `worker` |
| Non-simple task | §2 Process Governance |
| `fast mode` trigger | §2 → Fast Mode |
| `auto mode` trigger | §2 → Auto Mode |
| Session nearly full / handing off | §2 → Session Handoff |
| Resume / interruption / context compaction | Re-read this file + `ROADMAP.md` (incl. its § Session Handoff) |

§1 Coding Behavior applies to every agent that writes code. §2 Process Governance governs non-simple work only.

---

## 1. Coding Behavior

### Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### Surgical Changes

**Touch only what you must. Clean up only your own mess.**

- Don't "improve" adjacent code, comments, or formatting; don't refactor what isn't broken; match existing style.
- If you notice unrelated dead code, mention it — don't delete it.
- Remove imports/variables/functions that YOUR changes orphaned; leave pre-existing dead code alone.

Every changed line must trace to the request. Preserve user work — don't revert or overwrite changes you didn't make unless asked.

### Goal-Driven Execution

**Define success criteria. Loop until verified.** Transform tasks into verifiable goals ("Add validation" → "write tests for invalid inputs, then make them pass"). For multi-step tasks, state a brief plan with a verify check per step. Strong criteria let you loop independently; weak criteria ("make it work") require constant clarification.

### Code & Test Policy

- **File size:** source files ≤ 800 lines; split before a planned change pushes over. Generated/vendored files, lockfiles, snapshots, build artifacts, and fixtures are exempt.
- **Tests:** every behavior change ships tests (mock where the behavior can be modeled); run them before marking a step done.
- **Verify for real:** don't sign off on mocks alone — exercise the actual code path (run the real build / app / flow, hit the real dependency if there is one) before a phase is done, and ground-truth against direct evidence, not just surface output. *(Nothing external to exercise? Running the real thing still counts.)*
- **Dependencies:** pin third-party versions; verify behavior against docs/source; no silent upgrades across phases.

### Test Discipline (BDD-light)

Test-writing style for all test code. (The outside-in TDD gate flow lives in §2 → Outside-in TDD.)
- **Test names describe behavior, not function names.**
- **One-line Given/When/Then intent comment per test** above each `test()`/`it()`.
- **`describe(behavior) { it(scenario) }`** groups tests by behavior surface.
- **NO Gherkin / `.feature` files / Cucumber.** The discipline is in names + intent comments + describe grouping.

### Commit + Release Policy

- **Commit on phase completion** (Step 7 / FM-4): the orchestrator makes one commit per phase with a per-file summary.
- **No secrets in commits:** secret files stay gitignored; a surfaced secret is an incident — rotate it.
- **Releases are project-specific.** If the project versions or tags releases, do it as part of the phase commit (bump manifest, tag, push); otherwise a plain commit is enough. `ROADMAP.md` owns the version/milestone sequence if there is one.

---

## 2. Process Governance

### Roadmap

- Canonical path: `ROADMAP.md`. Read before meaningful work; create if missing. Its required layout and phase-document skeletons live in that file.
- `ROADMAP.md` keeps full entries only for active work. Every `OPEN` or `BLOCKED` task records status, scope, gates, run mode, live verification target, blockers, and next action.
- **Completion archive:** when a task becomes `CLOSED`, the agent completing it MUST update `ROADMAP.md` in the same completion gate without waiting for an operator reminder: remove the full task entry from active work and append one outcome-focused sentence under `## Change History`. Format: `YYYY-MM-DD — <task id/title> — CLOSED — <what is now true> (<version, evidence doc, or commit if useful>)`. Keep implementation narrative, gate transcripts, test counts, and intermediate findings in phase docs and git history, not in the roadmap.
- **Decision log:** every operator decision (direction, sign-off, scope change, exception) is appended to a `## Decisions` block with date + one-line rationale. Before accepting one, the orchestrator MUST reason it through first and MAY push back — it never complies blindly; it records the decision only once resolved.
- **Division of state:** `ROADMAP.md` holds live progress/state; `AGENTS.md` holds durable core/process (Hard Rule 10).

### Session Handoff

When the session nears its context limit (or before a deliberate stop), the orchestrator writes a single `## Session Handoff` block into `ROADMAP.md` (overwritten each time) so the next session resumes cold, then tells the user. Record: active phase + gate + mode; last deliverable + exact next action; open blockers + in-flight owners (worktree/branch) + uncommitted state; the next dispatch (which subagent, which doc first). On resume, re-read `AGENTS.md` + `ROADMAP.md` (incl. handoff) before acting.

### Hard Rules

1. Simple requirements may be handled directly by `worker`.
2. Non-simple requirements use the Phase Gate Matrix (or Fast Mode where eligible); no step starts until the previous passes; no phase starts until the previous completes its final step.
3. No implementation before research, plan, test-scaffold, and critic approval (full mode). In Fast Mode, no implementation before the plan (FM-0) and the FM-1 critic pass.
4. Blockers must be inserted into `ROADMAP.md` as a blocker phase before being worked around.
5. Subagents are serial by default. Parallel work is allowed only after the plan is frozen and tasks have disjoint write ranges — each parallel owner in its own worktree (Hard Rule 12).
6. Source classification, contract impact, and external-system runtime behavior must be resolved before implementation begins.
7. Resource-exclusive verification (one browser, one DB lock) must run serially.
8. Any load-bearing security boundary the project relies on (auth, credential isolation, sandbox/permission limits) is non-negotiable. Weakening it is a full Phase Gate Matrix item with explicit operator approval at the critic gate (Step 3) + an `AGENTS.md` amendment — never a worker or Fast Mode task.
9. **Operator validation gates at major milestones:** record explicit operator sign-off points (release boundaries or other big milestones); auto-mode must halt at these boundaries.
10. **`AGENTS.md` is the project core + the agent's operating contract (its "soul").** The agent MUST NOT self-edit it without explicit operator approval. Progress lives in `ROADMAP.md`; core/process lives in `AGENTS.md`.
11. **Gate work runs in subagents — the orchestrator coordinates, it does not do the work.** The orchestrator's only hands-on gates are Step 0 (exploration + roadmap) and Step 7 (commit) in full mode, and FM-0 (plan), FM-3 (live verification), and FM-4 (commit) in Fast Mode. Every other gate is run by a spawned subagent (Codex spawns its own); the orchestrator MUST NOT write a plan, scaffold, production code, critique, or audit itself. **The orchestrator never writes production code in either mode.**
12. **Every task sub-owner works in its own git worktree.** A subagent owning a roadmap task operates in a dedicated worktree/branch; parallel owners never share a working tree. No subagent merges into the integration branch — the orchestrator reviews each owner's diff and decides the **merge** (an orchestrator step, after its review).

### Phase Gate Matrix (full mode)

**Non-simple work runs these gates in order.** Each gate runs in its subagent (Codex-spawned); the orchestrator dispatches, reads deliverables, and routes, but does not perform gate work itself (Hard Rule 11). Every actor reads the phase-N entry in `ROADMAP.md` before its step.

**Independence:** the critic (Step 3) and audit (Step 6) run as **fresh subagent sessions**, separate from the `builder` that wrote the code — a reviewer must not be primed by the implementer's context.

| Step | Gate | Actor | Required Reads¹ | Allowed Writes² | Output | On Fail |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Roadmap Entry + Exploration | orchestrator | prior phase docs | roadmap | exploration findings + operator-confirmed direction → new phase-N entry | → 1 |
| 1 | Plan | architect | phase-N entry | docs | `docs/phase-N-plan.md` — §5 testable behaviors + §6 code sketches | → 2 |
| 2 | Test Scaffold (TDD) | validator | plan §5 + §6 | test code (scaffolds, all-failing) + docs (test contract) | scaffolds compile + all fail | → 3 |
| 3 | Critic (plan + TDD) | guardian (fresh) | plan + scaffolds | docs | `docs/phase-N-critics.md` — findings **BLOCKER / CONCERN (CONCERN-MR) / NIT** + verdict | → 3a or 0 |
| 3a | Plan / TDD Revision | architect / validator | critics doc | docs / test code | revised plan and/or scaffolds | → 3 |
| 4 | Implementation | builder | plan + scaffolds | code (production only — no test files) | scoped code; scaffolds compile + reach assertion-TODO branch | → 5 |
| 5 | Validation | validator | plan + diff + scaffolds | test code + docs | mock + live tests + final report | → 5a |
| 5a | Validation Fix | builder | test + plan | code | scoped fixes | → 5 |
| 6 | Final Review (Audit) | guardian (fresh) | all phase docs | docs | `docs/phase-N-review.md` (ACCEPT / REJECT) | → 5a |
| 7 | Roadmap Update + Commit | orchestrator | all phase docs | roadmap + git | task removed from active work and archived as one `Change History` sentence; commit + tag + push | phase complete |

¹ Implicit read every step: the phase-N `ROADMAP.md` entry. ² `docs` = reports under `docs/`; `test code + docs` = tests in `tests/**` + reports (no production edits); `code` = production files in scope, **no test files** (validator's exclusive scope at Steps 2 + 5); `roadmap + git` = `ROADMAP.md` + git only.

### Outside-in TDD (gate mechanics)

Validator writes failing scaffolds (Step 2) **before** the builder implements (Step 4); the critic (Step 3) reviews plan + scaffolds together. Scaffolds compile with TODO assertion bodies; the builder makes them compile + reach the TODO branch (Step 4); the validator fills assertions + adds edge cases + runs the full suite (Step 5). `docs/phase-N-test.md` is two-pass: § Test Contract at Step 2, § Results appended at Step 5. Fast Mode differs — see § Fast Mode.

### Fast Mode

**Trigger:** the user says `fast mode`. For lower-risk / time-boxed work where the full gate set is overkill; it relaxes *process gates*, never *safety boundaries*. Parallelize whatever can be parallelized.

**Eligibility (authoritative):** NOT for load-bearing security boundaries (Hard Rule 8), schema migrations, or anything crossing an operator-validation boundary (Hard Rule 9) — those use the full matrix. Escalate mid-flight on risk/failure.

| Step | Gate | Actor | Notes |
| --- | --- | --- | --- |
| FM-0 | Research + Plan | orchestrator | Orchestrator explores + writes the plan itself. Direction gate still applies. |
| FM-1 | Critic | guardian (fresh) | Reviews the plan. On fail → FM-0. |
| FM-2 | Implementation + Mock Tests | builder | Writes production code + its mock tests. |
| FM-3 | Live Verification | orchestrator | Runs mock suite + live smoke tests. On fail → FM-2. |
| FM-4 | Commit | orchestrator | Commit sequence per § Commit + Release Policy. |

**Deltas vs full mode:** dropped = architect plan gate, validator scaffold + validation, and the audit. The critic (FM-1) is kept but reviews the plan only. The builder writes its mock tests inline (the one place the "implementer must not write tests" rule is relaxed).

### Orchestrator Handoff Discipline

The orchestrator dispatches each gate to its subagent (Codex spawns its own) — it never performs the gate itself (Hard Rule 11).

**Read before you route.** Before dispatching the next actor, the orchestrator MUST fully read the deliverable that just landed (Read end-to-end — at least every BLOCKER/CONCERN finding + the verdict; a `grep`/`head`/`tail` scan is not enough), and the dispatch prompt MUST quote one specific finding (file:line or verbatim fix wording) as proof. Routing blindly garbles context and costs a full step rework. **Exceptions:** initial setup or an unambiguously trivial deliverable.

### Auto Mode

When the user says `auto mode`, the orchestrator runs the roadmap unattended:

1. Re-read `AGENTS.md` + `ROADMAP.md` (also on resume, interruption, or context compaction).
2. **Task ownership.** Split the phase into roadmap tasks, each with a **sub-owner** (one subagent) in its own worktree (Hard Rule 12). The orchestrator stays the integrator; insert the smallest blocker-resolution phase if blocked.
3. **Difficulty routing.** Judge each task's difficulty → **Fast Mode** for lower-risk/localized work, the **full matrix** for complex/contract-touching work. Record the chosen mode.
4. **Model policy.** Default every subagent to the **lighter Codex tier** (e.g. `spark`) — including full-mode gates. Escalate to the **full Codex model** only for especially hard tasks (novel design, tricky concurrency, wide blast radius); the orchestrator decides and notes it.
5. **Direction gate:** at Step 0 / FM-0 the orchestrator decides the route itself (no halt) but logs the decision (§ Roadmap → Decision log).
6. **Merge + commit:** merge each sub-owner's worktree only after the orchestrator's review (Hard Rule 12); auto-commit on each phase's final step (Step 7 / FM-4).
7. **Halt at operator-validation boundaries** (Hard Rule 9) — never auto-cross a major-version gate.
8. **Write a § Session Handoff** before the session ends or context fills, and tell the user.
9. **Never self-edit `AGENTS.md`** without operator approval (Hard Rule 10).

---

**Working if:** fewer unnecessary diffs, fewer rewrites from overcomplication, clarifying questions before implementation, every phase ends with a clean commit, every load-bearing security boundary holds.
