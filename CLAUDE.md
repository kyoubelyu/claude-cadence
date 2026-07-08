# CLAUDE.md

Behavioral guidelines and multi-agent process rules for this codebase.

## Project Scope

_Reserved for per-repository scope. Populate with:_

- **Product goal** — what this codebase ships and to whom.
- **Approved capabilities** — the libraries, frameworks, runtime surfaces, and external interfaces the agent may use.
- **Owned components** — paths the agent edits (`src/`, `tests/`, configs, docs). Record where the **frontend / backend boundary** falls (paths, packages) so FE/BE routing is unambiguous.
- **External authoritative systems** — frameworks/SDKs/services treated as external truth (pin versions, verify behavior against docs, do not silently upgrade).
- **Explicit non-goals** — features and integrations that are out of scope; record retired/abandoned directions with date + rationale.
- **Live environment** — what the agent operates against during verification (real user account, real browser, real API), and which phases are read-only vs. full live windows.

If any of these are unclear, resolve them during Step 0 exploration (§2 → Phase Gate Matrix, Step 0) before implementation.

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

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.
- Remove imports/variables/functions that YOUR changes orphaned; leave pre-existing dead code alone.

Every changed line must trace directly to the user's request. Preserve user work — do not revert or overwrite changes you did not make unless asked.

### Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

### Code & Test Policy

- **File size:** source files ≤ 800 lines; split before a planned change would push over. Generated/vendored files, lockfiles, snapshots, build artifacts, and fixtures are exempt.
- **Tests:** every behavior change ships tests (mock where the behavior can be modeled); run them before marking a step done.
- **Verify for real:** don't sign off on mocks alone — exercise the actual code path (run the real build / app / flow, and hit the real dependency if there is one) before a phase is done, and ground-truth the result against direct evidence, not just surface output. *(If there's nothing external to exercise, running the real thing still counts.)*
- **Dependencies:** pin third-party versions; verify behavior against docs/source; no silent upgrades across phases.

### Test Discipline (BDD-light)

Test-writing style for all test code. (The outside-in TDD gate flow — who writes scaffolds when, and the two-pass test doc — lives in §2 → Outside-in TDD.)
- **Test names describe behavior, not function names.** Example: `T-Cache.2: when the key is absent, returns the default and writes nothing`. Not: `test getValue()`.
- **One-line Given/When/Then intent comment per test** above each test body. Example:
  ```
  // Given: an empty cache
  // When:  get(key) is called
  // Then:  returns the default; no write occurs
  test("...", () => { ... });
  ```
- **`describe(behavior) { it(scenario) }`** blocks group related tests by behavior surface. Prefer describe/it for new tests.
- **NO Gherkin / NO `.feature` files / NO Cucumber.** The discipline is in test names + intent comments + describe grouping; separate spec files add overhead without value.

### Commit + Release Policy

- **Commit on phase completion:** when the active mode's final step marks a phase complete (Step 7 / FM-4), the orchestrator makes one git commit for that phase, listing each changed file with a one-sentence description.
- **No secrets in commits:** credential/secret files stay gitignored; a surfaced secret is an incident — rotate it. Attribution trailer is project-defined.
- **Releases are project-specific.** If the project versions or tags releases, do it as part of the phase commit (bump the version manifest, tag, push); otherwise a plain commit is enough. `ROADMAP.md` owns the version/milestone sequence if there is one.

---

## 2. Process Governance

### Roadmap

- Canonical path: `ROADMAP.md`. Read before meaningful work; create if missing before non-simple implementation. Layout templates live in `TEMPLATES.md` (roadmap entry + phase doc).
- Every phase completion and every blocker must update the roadmap with: status, scope, gates, **FE/BE work-item classification**, run mode (full vs Fast Mode), live verification target, blockers, next action.
- **Decision log:** every operator decision (direction, sign-off, scope change, exception) is appended to a `## Decisions` block in `ROADMAP.md` with date + one-line rationale. Before accepting one, the orchestrator MUST reason it through first and MAY push back with a counter-argument — it never complies blindly; it records the decision only once resolved, then proceeds.
- **Division of state:** `ROADMAP.md` holds live progress/state; `CLAUDE.md` holds durable core/process (Hard Rule 10). Don't put process rules in the roadmap, or transient phase state in `CLAUDE.md`.

### Session Handoff

When the current session approaches its context limit (or before a deliberate stop), the orchestrator writes a **handoff** into `ROADMAP.md` so the next session can resume cold, then tells the user a handoff was written and the work should continue in a fresh session.

Maintain a single `## Session Handoff` block in `ROADMAP.md` (overwritten each time), recording:
- active phase + current gate/step, and run mode (full / Fast Mode);
- what just landed (last deliverable) and the exact next action;
- open blockers, in-flight FE/BE work + which actor owns each, and any uncommitted state;
- the next dispatch: which actor to invoke and which doc to read first.

On resume / context compaction, re-read `CLAUDE.md` + `ROADMAP.md` (including the handoff) before acting.

### Hard Rules

1. Simple requirements (e.g. a single obvious edit) may be handled directly by `worker`.
2. Non-simple requirements must use the Phase Gate Matrix (or Fast Mode where eligible); no step starts until the previous passes; no phase starts until the previous completes its final step.
3. No implementation before research, plan, test-scaffold, and critic approval (full mode). In Fast Mode, no implementation before the orchestrator's plan (FM-0) and the FM-1 critic pass.
4. Blockers must be inserted into `ROADMAP.md` as a blocker phase before being worked around — including unavailability of the Codex rescue runtime.
5. Subagents are serial by default. Parallel work is allowed only after the plan is frozen and the tasks have independent concerns or disjoint write ranges — each parallel owner in its own worktree (Hard Rule 12); the FE/BE split is the canonical parallel case (see § Frontend / Backend Split).
6. Source classification, contract impact, FE/BE routing, and external-system runtime behavior must be resolved before implementation begins.
7. Resource-exclusive verification (e.g. only one browser instance, only one DB lock) must run serially.
8. Any load-bearing security boundary the project relies on (auth, credential isolation, sandbox/permission limits) is non-negotiable. Weakening it is a full Phase Gate Matrix item with explicit operator approval at the critic gate (Step 3) + a CLAUDE.md amendment recording the approved use case — never a worker or Fast Mode task.
9. **Operator validation gates at major milestones:** record explicit operator sign-off points (release boundaries or other big milestones). Auto-mode must halt at these boundaries.
10. **`CLAUDE.md` is the project core + the agent's operating contract (its identity / "soul").** The agent MUST NOT create, edit, or self-update `CLAUDE.md` without explicit operator approval — operator-approved amendments (e.g. the Hard Rule 8 exception path) are the only way it changes. Live progress/state lives in `ROADMAP.md`; durable core/process lives in `CLAUDE.md`.
11. **Gate work runs in subagents — the orchestrator coordinates, it does not do the work.** In **full mode** the orchestrator's *only* hands-on gates are Step 0 (exploration + roadmap) and Step 7 (commit); **every other gate MUST be dispatched to its designated actor via the Agent tool** — the `architect` / `validator` / `builder` Claude subagents, or Codex (via the companion CLI, § Codex Delegation) for backend + critic + audit — and the orchestrator MUST NOT write a plan, test scaffold, production code, critique, or audit itself. Collapsing a full-mode gate into the main loop defeats the independent-review design and is a process violation. In **Fast Mode** the orchestrator is hands-on only for FM-0 (plan), FM-3 (live verification), and FM-4 (commit); the FM-1 critic and FM-2 implementation are dispatched (critic + backend → Codex, frontend → `builder`). **The orchestrator never writes production code in either mode.** (Simple `worker` tasks under Hard Rule 1 also run in the `worker` subagent, not the main loop.)
12. **Every task sub-owner works in its own git worktree.** A subagent that owns a roadmap task (or an FE/BE work item) operates in a dedicated worktree/branch; parallel owners never share a working tree. No subagent merges into the integration branch — the orchestrator reviews each owner's diff and decides the **merge** (merge is an orchestrator step, after its review).

### Phase Gate Matrix (full mode)

**Non-simple work runs these gates in order.** Each gate runs in its actor's **own subagent invocation** (Agent tool) — the orchestrator dispatches, reads deliverables, and routes, but does **not** perform gate work itself (Hard Rule 11). The orchestrator tells every actor: phase number, title, scope, current gate, FE/BE assignment, required reads, allowed writes, expected output. **Every actor must read the phase-N entry in `ROADMAP.md` before doing its step** — the entry defines goal, scope, gates, FE/BE classification, live verification target, blockers, next action.

**Outside-in TDD:** the validator writes the test scaffold (Step 2) **before** the implementers (Step 4), and the critic (Step 3) reviews plan + scaffold together — detailed in § Outside-in TDD below.

**Operator direction gate (Step 0):** the orchestrator explores directly, then confirms the chosen route/direction with the human operator **before** writing it into `ROADMAP.md`. In normal (non-auto) mode the orchestrator **halts** for operator sign-off here. In auto mode the orchestrator stands in for the operator and decides the direction itself (see § Auto Mode).

| Step | Gate | Actor | Required Reads¹ | Allowed Writes² | Output | On Fail |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Roadmap Entry + Exploration | orchestrator | prior phase docs (if any) | roadmap | exploration findings + operator-confirmed direction + FE/BE classification, written as the new phase-N entry in `ROADMAP.md` | → 1 |
| 1 | Plan | architect | phase-N entry | docs | `docs/phase-N-plan.md` — §5 **testable behaviors** (concrete pre/post conditions) + §6 **code sketches**; FE/BE-separable sections + the FE↔BE interface contract | → 2 |
| 2 | Test Scaffold (TDD) | validator | plan §5 (testable behaviors) + §6 (code sketches) | test code (scaffolds — compileable, all-failing, named per §5; assertion bodies are TODO) + docs (initial `docs/phase-N-test.md` § Test Contract) | scaffolds compile + all fail; test contract doc | → 3 |
| 3 | Critic (plan + TDD) | Codex (fresh session) | plan + scaffolds + test contract | docs | `docs/phase-N-critics.md` — verdict on plan + scaffolds; findings tagged **BLOCKER** / **CONCERN** (CONCERN-MR = must-resolve before proceeding) / **NIT** | → 3a or 0 |
| 3a | Plan / TDD Revision | architect (plan) / validator (TDD) | critics doc | docs / test code | revised plan and/or revised scaffolds (research/exploration gap → hand back to orchestrator → Step 0) | → 3 |
| **4** | **Implementation (FE ∥ BE)** | **`builder` (frontend) + Codex (backend)** | plan + Step 2 scaffolds | code (production only — NO test files; routed per § Frontend / Backend Split) | scoped code; scaffolds compile + reach assertion-TODO branch | → 5 |
| 5 | Validation | validator | plan + FE & BE diffs + own scaffolds | test code (fill assertion bodies + add edge cases) + docs (append § Results to `docs/phase-N-test.md`) | mock + live tests + final test report (defects, gate verdicts, residuals) | → 5a |
| 5a | Validation Fix (FE ∥ BE) | `builder` (frontend) / Codex (backend) | test + plan | code (production only) | scoped fixes | → 5 |
| 6 | Final Review (Audit) | Codex (fresh session) | all phase docs | docs | `docs/phase-N-review.md` | → 5a |
| 7 | Roadmap Update + Commit | orchestrator | all phase docs | roadmap + git | phase-N entry marked complete; **commit per § Commit + Release Policy** | phase complete |

¹ **Implicit read for every step:** the phase-N entry in `ROADMAP.md`. The "Required Reads" column lists additional inputs on top of that.

² **Write scopes** — `docs`: reports under `docs/` only. `test code + docs`: may write mock + live test code in `tests/**` + reports under `docs/`; no production code edits. `code`: production files within approved phase scope only — **the implementers MUST NOT write test files (mock or live) in full mode; that is validator's exclusive scope at Steps 2 + 5**. FE/BE routing + disjoint write ranges are governed by § Frontend / Backend Split. If a test gap surfaces during Step 4, the orchestrator hands it back to validator rather than letting an implementer author tests. `roadmap + git`: only `ROADMAP.md` + git operations (commit/tag/push). At Step 0 the orchestrator owns exploration directly (Read/Grep/Glob/WebSearch).

### Outside-in TDD (gate mechanics)

**Outside-in TDD**: validator writes failing test scaffolds (Step 2) **BEFORE** the implementers (Step 4) — and the critic (Step 3) reviews the plan **and** the scaffolds together. Scaffolds compile + all assertion bodies are TODO; tests intentionally fail. The implementers' job at Step 4 is to make scaffolds compile + reach the assertion-TODO branch (TODO assertions are filled at Step 5). Validator's Step 5 fills assertion bodies + adds edge cases + runs the full suite.

Why scaffold-before-critic: writing scaffolds first lets the critic catch plan/test mismatch in one pass, and prevents the validator unconsciously aligning tests to whatever the implementers produced.

**`docs/phase-N-test.md` is two-pass (full mode).** Validator writes it twice in a single phase:
1. **Step 2** (scaffold-time): test contract — test names + intent comments + which gates they cover. Tells the critic + implementers what behaviors are about to be verified.
2. **Step 5 final** (post-run, after all 5a iterations): test report — actual results, defects, gate verdicts, residual risks. Appended; the Step-2 contract stays as historical record.

The accumulated `phase-N-test.md` is self-contained: contract → results trail. Fast Mode differs (no scaffold step; implementers write their own mock tests) — see § Fast Mode → Deltas vs full mode.

### Fast Mode

**Trigger:** the user says `fast mode` (a process mode — distinct from Claude Code's built-in `/fast` output-speed toggle). For lower-risk or time-boxed work where the full gate set is overkill; it relaxes *process gates*, never *safety boundaries*. **Parallelize whatever can be parallelized.**

**Eligibility (authoritative):** Fast Mode is **NOT** allowed for changes that touch a load-bearing security boundary (Hard Rule 8), schema migrations, or anything crossing an operator-validation boundary (Hard Rule 9). Those always use the full Phase Gate Matrix. If risk or a failure emerges mid-flight, escalate: open a normal phase and resume under full gates.

| Step | Gate | Actor | Notes |
| --- | --- | --- | --- |
| FM-0 | Research + Plan | orchestrator | Orchestrator explores **and** writes the plan itself (no scout/architect), including FE/BE classification + the FE↔BE interface contract. Output: lightweight plan in the phase-N entry (+ optional `docs/phase-N-plan.md`). Direction gate still applies (see matrix preamble). |
| FM-1 | Critic | Codex (fresh session) | Reviews the orchestrator's plan + FE/BE split. On fail → back to FM-0. |
| FM-2 | Implementation + Mock Tests (FE ∥ BE) | `builder` (frontend) + Codex (backend) | Each writes production code **and** its layer's mock tests, in parallel on disjoint write ranges (§ Frontend / Backend Split). |
| FM-3 | Live Verification | orchestrator | Orchestrator runs the mock suite + live smoke tests against the real target. On fail → back to FM-2 for the responsible layer. |
| FM-4 | Commit | orchestrator | Commit sequence per § Commit + Release Policy. |

**Deltas vs full mode:**
- **Dropped:** the architect plan gate (orchestrator plans), the validator test-scaffold + validation gates, and the final **audit**. The critic (FM-1) is kept but reviews the plan only — no TDD scaffold exists.
- **Test-authoring exception:** each implementer writes its own layer's **mock tests** alongside the code — the only place the "implementers must not write tests" rule is relaxed, scoped to mock tests.
- **Participants:** orchestrator (plan / live verification / commit), `builder` (FE), Codex (FM-1 critic + BE implementation, separate sessions); `architect` and `validator` step aside.

### Frontend / Backend Split

Implementation and fix work is routed by layer; this section is authoritative for routing + parallelism.

- **Frontend → Claude `builder`. Backend → Codex.** (Both modes; Codex runs via the companion CLI — § Codex Delegation.)
- **Who classifies.** The **orchestrator** classifies each work item as FE or BE and records the classification in the phase-N `ROADMAP.md` entry (and references it from the plan). The architect's plan must keep FE and BE concerns in **separable sections with disjoint write ranges** so the two implementers don't collide.
- **Parallelism (the rule).** Once the plan is frozen (full mode: after the Step 3 critic pass; Fast Mode: after the FM-1 critic pass), disjoint FE and BE work runs **in parallel** (Hard Rule 5); further independent work items fan out too. A work item that necessarily touches shared FE+BE files is **serialized**, not parallelized (Hard Rules 5 + 7).
- **Cross-boundary contract.** Where FE depends on a BE interface (API shape, payload schema), that contract is fixed in the plan before parallel implementation starts, so neither side blocks the other.
- **Boundaries unchanged.** All §1 rules bind both implementers identically.

### Codex Delegation

Codex runs the **backend implementation/fix** (Steps 4, 5a / FM-2), the **critic** (Step 3 / FM-1), and — full mode only — the **audit** (Step 6). The orchestrator delegates by running the Codex companion CLI in a **Bash** call — no subagent wrapper:

```bash
CODEX="…/codex-companion.mjs"      # path shown by /codex:setup
node "$CODEX" task --write --fresh "<brief: phase/gate/scope · work items (+ interface contract) · required reads · write scope · expected output>"
# same thread again: --resume-last   ·   review-only (no writes): drop --write
```

- **Independence (critic/audit):** run the critic (Step 3 / FM-1) and audit (Step 6) as **fresh** threads (`--fresh`), separate from the backend-implementation thread — Codex must not review code in the context that wrote it.
- **Write scope:** implementation → backend code (+ mock tests in Fast Mode); critic/audit → reports under `docs/` only.
- **Prerequisite:** Codex must be ready (`/codex:setup`); if unavailable mid-phase, insert a blocker phase (Hard Rule 4) — don't silently reassign the work.
- **Clean up orphans:** a backgrounded or hung Codex job blocks the next task. After each delegation, `node "$CODEX" status --all` and `cancel` any orphan before starting a new one.

### Orchestrator Handoff Discipline

The orchestrator dispatches each gate to its actor — Claude subagents via the **Agent tool**, Codex via the companion CLI (§ Codex Delegation) — and never performs the gate itself (Hard Rule 11).

**Read before you route.** Before dispatching the next actor, the orchestrator MUST fully read the deliverable that just landed — use the `Read` tool end-to-end (at minimum every BLOCKER/CONCERN finding + the verdict; a `grep`/`head`/`tail` scan is not enough) — and the dispatch prompt MUST quote one specific finding (file:line or verbatim fix wording) as proof. Routing blindly garbles context and costs a full step rework.

**Exceptions:** initial setup (before phase 1's Step 0) or an unambiguously trivial deliverable (e.g. a 5-line `worker` patch).

### Auto Mode

When the user says `auto mode`, the orchestrator runs the roadmap unattended:

1. Re-read `CLAUDE.md` + `ROADMAP.md` (also on resume, interruption, or context compaction).
2. **Task ownership.** Split the current phase into roadmap tasks and give each a **sub-owner** — one subagent, working in its own worktree (Hard Rule 12). The orchestrator stays the integrator; insert the smallest blocker-resolution phase if blocked.
3. **Difficulty routing.** The orchestrator judges each task's difficulty and routes it: **Fast Mode** for lower-risk/localized work, the **full Phase Gate Matrix** for complex or contract-touching work. Record the chosen mode on the task.
4. **Model policy.** Default every subagent to a **Sonnet-class** model — including full-mode gates. Escalate a task to an **Opus-class** model only when it is especially hard (novel design, tricky concurrency, wide blast radius); the orchestrator decides and notes it on the task.
5. **Direction gate:** at Step 0 / FM-0 the orchestrator stands in for the operator and picks the route itself (no halt) — but still logs the decision (§ Roadmap → Decision log).
6. **Merge + commit:** merge each sub-owner's worktree only after the orchestrator's review (Hard Rule 12); auto-commit on each phase's final step (Step 7 / FM-4).
7. **Halt at operator-validation boundaries** (Hard Rule 9) — never auto-cross a major-version gate.
8. **Write a § Session Handoff** before the session ends or context fills, and tell the user.
9. **Never self-edit `CLAUDE.md`** without operator approval (Hard Rule 10).

---

**Working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, clarifying questions come before implementation rather than after mistakes, every phase ends with a clean commit, every load-bearing security boundary holds across every PR.
