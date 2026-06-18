# CLAUDE.md

Behavioral guidelines and multi-agent process rules for this codebase.

## Project Scope

_Reserved for per-repository scope. Populate with:_

- **Product goal** — what this codebase ships and to whom.
- **Approved capabilities** — the libraries, frameworks, runtime surfaces, and external interfaces the agent may use.
- **Owned components** — paths under this repo that the agent edits (`src/`, `tests/`, configs, README, etc.) plus the published product contract (bin entrypoint, exports, on-disk schemas). Record where the **frontend / backend boundary** falls (paths, packages) so FE/BE routing is unambiguous.
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

### Product Contract (operator-facing surface)

**The published package surface — CLI entrypoint, library exports, system prompt structure, tool inventory, on-disk schemas — is the operator-facing product contract.**

- Public structural invariants (prompt composition order, file layout, command shape) are part of the contract. Content inside the structure may be tuned per phase; structure changes require a major version bump.
- Tool name + parameter schema is part of the contract. Renaming or schema-tightening requires a major version bump.
- Any load-bearing security claim (e.g. a no-bash / no-shell-out boundary at the agent's tool layer, capability allowlist, credential isolation) is non-negotiable: changes that weaken it are Phase Gate Matrix items, never worker tasks (and never Fast Mode — see § Fast Mode → Eligibility).
- On-disk paths and config file schemas are stable across patch versions; schema changes go through the Phase Gate Matrix with an explicit migration plan.
- Maintainers updating these surfaces go through §2 Phase Gate Matrix.

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

- **File size:** source files ≤ 800 lines. Split before a planned change would push over. Generated files, lockfiles, vendored dependencies, snapshots, build artifacts, and data fixtures are exempt when not manually maintained.
- **Tests:** every behavior change ships mock test code where the behavior can be modeled. Automated mock tests run before live verification.
- **Live verification:** every implementation phase that touches an external authoritative system includes platform-level smoke tests against the real target. Mock-only is insufficient unless `ROADMAP.md` defines the phase as research/prototype-only. Required in both modes (who runs it differs — see § Fast Mode).
- **Interactive / end-to-end testing:** when an agent must be driven end-to-end (full LLM loop + real tools) for a live verification, use the project-sanctioned harness (e.g. a real CLI launched inside a tmux pane and steered with `tmux send-keys` / `tmux capture-pane`). Ground-truth every result against the audit log and direct evidence from the external system — never trust UI text alone.
- **Real-agent vs. tool-smoke distinction:** a *live test of the agent* MUST drive the full agent loop (system prompt + LLM + the agent's own tool selection). A test that invokes tools directly bypassing the LLM is a **tool/integration smoke** — it verifies the tool layer, not that the agent works, and MUST NOT be counted as agent live-verification.
- **Tool-gap rule:** if the real agent cannot complete a live-test task because a tool is missing or inadequate, whoever runs validation RECORDS and REPORTS it as a finding — never papers the gap over with a test-side workaround. The **orchestrator** evaluates the gap and decides whether a new/extended tool is warranted; tool inventory changes are a Phase Gate decision.
- **External authoritative systems cross-check:** treat external SDKs/protocols/services as authoritative — pin versions, verify behavior against docs and source, do not assume features exist without dependency-level confirmation. Do not silently upgrade across phases.

### Test Discipline (BDD-light)

Test-writing style for all test code. (The outside-in TDD gate flow — who writes scaffolds when, and the two-pass test doc — lives in §2 → Outside-in TDD.)
- **Test names describe behavior, not function names.** Example: `T-Compact.2: when messages.length <= LAST_K, returns messages unchanged with summarizedCount=0`. Not: `test compactMessages()`.
- **One-line Given/When/Then intent comment per test** above each `test()` / `it()` body. Example:
  ```typescript
  // Given: piped stdin EOF before while-loop runs
  // When:  rl.question() called on closed readline interface
  // Then:  resolves null cleanly via try/catch (no ERR_USE_AFTER_CLOSE throw)
  test("...", () => { ... });
  ```
- **`describe(behavior) { it(scenario) }`** blocks group related tests by behavior surface. Prefer describe/it for new tests.
- **NO Gherkin / NO `.feature` files / NO Cucumber.** The discipline is in test names + intent comments + describe grouping; separate spec files add overhead without value.

### Commit + Release Policy

- **Auto-commit on phase completion:** when the final step of the active mode marks a phase complete (Step 7 in full mode, FM-4 in Fast Mode), the orchestrator creates a git commit summarizing that phase's work.
- When preparing a commit, list every changed file with a one-sentence description of its change.
- Commit attribution trailer is project-defined (record it in this section if used).
- Commits never include credential values. Credential / secret files stay outside git (gitignored). If an agent surfaces a credential in any docs, log, or commit message, treat it as an incident and rotate the credential.
- **Commit sequence:**
  0. **Bump `package.json` `version` (or equivalent version manifest) to match `<next-version>` (without the `v` prefix).** If a startup auto-updater reads the manifest, drift causes false "behind" detection and infinite update loops on global installs. The bump must be part of the phase commit.
  1. `git add` + `git commit` with phase summary.
  2. `git tag -a <next-version> -m "<phase title>"` — annotated tag; body becomes the GitHub Release notes.
  3. `git push origin main` then `git push origin <next-version>` — tag push fires the release workflow which auto-creates a GitHub Release.

Concrete version assignment is owned by `ROADMAP.md` § Version sequence — orchestrator reads ROADMAP at commit time to pick the next tag.

---

## 2. Process Governance

### Roadmap

- Canonical path: `ROADMAP.md`. Read before meaningful work; create if missing before non-simple implementation.
- Every phase completion and every blocker must update the roadmap with: status, scope, gates, **FE/BE work-item classification**, run mode (full vs Fast Mode), live verification target, blockers, next action.
- **Division of state:** `ROADMAP.md` holds live progress/state; `CLAUDE.md` holds durable core/process (Hard Rule 10). Don't put process rules in the roadmap, or transient phase state in `CLAUDE.md`.

### Session Handoff

When the current session approaches its context limit (or before a deliberate stop), the orchestrator writes a **handoff** into `ROADMAP.md` so the next session can resume cold, then tells the user a handoff was written and the work should continue in a fresh session.

Maintain a single `## Session Handoff` block in `ROADMAP.md` (overwritten each time), recording:
- active phase + current gate/step, and run mode (full / Fast Mode);
- what just landed (last deliverable) and the exact next action;
- open blockers, in-flight FE/BE work + which actor owns each, and any uncommitted state;
- the next dispatch: which actor to invoke and which doc to read first.

On resume / context compaction, re-read `CLAUDE.md` + `ROADMAP.md` (including the handoff) before acting.

### Roadmap Compaction

When `ROADMAP.md` grows large (accumulated completed-phase detail), the orchestrator dispatches a `worker` subagent on a **Sonnet-class model** to compact it: summarize completed phases into a condensed **Completed phases** digest (phase title, outcome, key decisions, version/tag, live-verification result) and drop their blow-by-blow detail.

- **Preserve, never lose:** the version sequence, every still-open blocker / deferred item, the current phase entry in full, and the latest § Session Handoff. Compaction is mechanical summarization — it MUST NOT alter in-progress state or drop unresolved items.
- **Recoverable:** the full pre-compaction roadmap stays in git history; the compaction is its own commit.
- **Scope:** a `worker` maintenance task (outside the Phase Gate Matrix, Hard Rule 1); for this task only, `worker` may edit `ROADMAP.md`, and the orchestrator reviews the result (confirming no open item was dropped) before the compaction commit.

### Hard Rules

1. Simple requirements (e.g. a single obvious edit) may be handled directly by `worker`.
2. Non-simple requirements must use the Phase Gate Matrix (or Fast Mode where eligible); no step starts until the previous passes; no phase starts until the previous completes its final step.
3. No implementation before research, plan, test-scaffold, and critic approval (full mode). In Fast Mode, no implementation before the orchestrator's plan and the FM-1 critic pass.
4. Blockers must be inserted into `ROADMAP.md` as a blocker phase before being worked around — including unavailability of the Codex rescue runtime.
5. Subagents are serial by default. Parallel work is allowed only after the plan is frozen and the tasks have independent concerns or disjoint write ranges — the FE/BE split is the canonical parallel case (see § Frontend / Backend Split for the rule).
6. Source classification, contract impact, FE/BE routing, and external-system runtime behavior must be resolved before implementation begins.
7. Resource-exclusive verification (e.g. only one browser instance, only one DB lock) must run serially.
8. Any load-bearing security boundary defined in §1 → Product Contract is non-negotiable **at the agent / tool layer**. Operator-initiated CLI-layer exceptions (narrow maintenance commands) require the full Phase Gate Matrix with explicit operator approval at the critic gate (Step 3), and a CLAUDE.md amendment recording the approved use case. The agent's LLM-callable tool surface remains within the boundary without exception.
9. **Operator validation gates between major versions:** record explicit operator sign-off points (e.g. v0.3 → v0.4 → v1.0). Auto-mode must halt at these boundaries.
10. **`CLAUDE.md` is the project core + the agent's operating contract (its identity / "soul").** The agent MUST NOT create, edit, or self-update `CLAUDE.md` without explicit operator approval — operator-approved amendments (e.g. the Hard Rule 8 exception path) are the only way it changes. Live progress/state lives in `ROADMAP.md`; durable core/process lives in `CLAUDE.md`.

### Phase Gate Matrix (full mode)

**Non-simple work runs these gates in order.** The orchestrator tells every actor: phase number, title, scope, current gate, FE/BE assignment, required reads, allowed writes, expected output. **Every actor must read the phase-N entry in `ROADMAP.md` before doing its step** — the entry defines goal, scope, gates, FE/BE classification, live verification target, blockers, next action.

**Outside-in TDD:** the validator writes the test scaffold (Step 2) **before** the implementers (Step 4), and the critic (Step 3) reviews plan + scaffold together — detailed in § Outside-in TDD below.

**Operator direction gate (Step 0):** the orchestrator explores directly, then confirms the chosen route/direction with the human operator **before** writing it into `ROADMAP.md`. In normal (non-auto) mode the orchestrator **halts** for operator sign-off here. In auto mode the orchestrator stands in for the operator and decides the direction itself (see § Auto Mode).

| Step | Gate | Actor | Required Reads¹ | Allowed Writes² | Output | On Fail |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Roadmap Entry + Exploration | orchestrator | prior phase docs (if any) | roadmap | exploration findings + operator-confirmed direction + FE/BE classification, written as the new phase-N entry in `ROADMAP.md` | → 1 |
| 1 | Plan | architect | phase-N entry | docs | `docs/phase-N-plan.md` — §5 **testable behaviors** (concrete pre/post conditions) + §6 **code sketches**; FE/BE-separable sections + the FE↔BE interface contract | → 2 |
| 2 | Test Scaffold (TDD) | validator | plan §5 (testable behaviors) + §6 (code sketches) | test code (scaffolds — compileable, all-failing, named per §5; assertion bodies are TODO) + docs (initial `docs/phase-N-test.md` § Test Contract) | scaffolds compile + all fail; test contract doc | → 3 |
| 3 | Critic (plan + TDD) | guardian | plan + scaffolds + test contract | docs | `docs/phase-N-critics.md` — verdict on plan + scaffolds; findings tagged **BLOCKER** / **CONCERN** (CONCERN-MR = must-resolve before proceeding) / **NIT** | → 3a or 0 |
| 3a | Plan / TDD Revision | architect (plan) / validator (TDD) | critics doc | docs / test code | revised plan and/or revised scaffolds (research/exploration gap → hand back to orchestrator → Step 0) | → 3 |
| **4** | **Implementation (FE ∥ BE)** | **`builder` (frontend) + Codex (backend)** | plan + Step 2 scaffolds | code (production only — NO test files; routed per § Frontend / Backend Split) | scoped code; scaffolds compile + reach assertion-TODO branch | → 5 |
| 5 | Validation | validator | plan + FE & BE diffs + own scaffolds | test code (fill assertion bodies + add edge cases) + docs (append § Results to `docs/phase-N-test.md`) | mock + live tests + final test report (defects, gate verdicts, residuals) | → 5a |
| 5a | Validation Fix (FE ∥ BE) | `builder` (frontend) / Codex (backend) | test + plan | code (production only) | scoped fixes | → 5 |
| 6 | Final Review (Audit) | guardian | all phase docs | docs | `docs/phase-N-review.md` | → 5a |
| 7 | Roadmap Update + Commit | orchestrator | all phase docs | roadmap + git | phase-N entry marked complete; **`git commit` + `git tag` + `git push origin main` + `git push origin <tag>` (workflow auto-publishes GitHub Release)** | phase complete |

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
| FM-0 | Research + Plan | orchestrator | Orchestrator explores **and** writes the plan itself (no separate scout/architect), including FE/BE classification + the FE↔BE interface contract. Output: lightweight plan in the phase-N entry (+ optional `docs/phase-N-plan.md`). Operator direction gate still applies (non-auto halts; auto decides). |
| FM-1 | Critic | guardian | Reviews the orchestrator's plan + FE/BE split. On fail → back to FM-0. |
| FM-2 | Implementation + Mock Tests (FE ∥ BE) | `builder` (frontend) + Codex (backend) | Each writes production code **and** its layer's mock tests, in parallel on disjoint write ranges (§ Frontend / Backend Split). |
| FM-3 | Live Verification | orchestrator | Orchestrator runs the mock suite + live smoke tests against the real target. On fail → back to FM-2 for the responsible layer. |
| FM-4 | Commit | orchestrator | Commit sequence per § Commit + Release Policy. |

**Deltas vs full mode:**
- **Dropped:** the separate validator test-scaffold step (full-mode Step 2) and the final guardian audit (Step 6). Critic (FM-1) reviews the plan only — no scaffold exists yet.
- **Test-authoring exception:** each implementer writes its own layer's **mock tests** alongside the code — the only place the "implementers must not write tests" rule is relaxed, scoped to mock tests.
- **Participants:** orchestrator, `guardian` (FM-1), `builder` (FE), Codex (BE); `architect` and `validator` step aside.
- **Unchanged:** all §1 safety boundaries, the FE/BE split + parallelism rules, and live-verification requirements for external authoritative systems.

### Frontend / Backend Split

Implementation and fix work is routed by layer; this section is authoritative for routing + parallelism.

- **Frontend → Claude `builder`. Backend → Codex (`codex:codex-rescue` subagent).**
- **Who classifies.** The **orchestrator** classifies each work item as FE or BE and records the classification in the phase-N `ROADMAP.md` entry (and references it from the plan). The architect's plan must keep FE and BE concerns in **separable sections with disjoint write ranges** so the two implementers don't collide.
- **Parallelism (the rule).** Once the plan is frozen (full mode: after the Step 3 critic pass; Fast Mode: after FM-1), disjoint FE and BE work runs **in parallel** (Hard Rule 5); further independent work items fan out too. A work item that necessarily touches shared FE+BE files is **serialized**, not parallelized (Hard Rules 5 + 7).
- **Cross-boundary contract.** Where FE depends on a BE interface (API shape, payload schema), that contract is fixed in the plan before parallel implementation starts, so neither side blocks the other.
- **Boundaries unchanged.** All §1 write-scope, security, test, and product-contract rules bind both implementers identically.

### Codex Rescue Delegation (backend)

- **Invocation.** The orchestrator invokes the `codex:codex-rescue` subagent via the **Agent tool** (`subagent_type: "codex:codex-rescue"`), forwarding the backend task; Codex performs the work through its own runtime and returns the deliverable. Do **not** call `Skill(codex:rescue)` from the agent loop — that re-enters the operator-facing slash command and hangs the session (`/codex:rescue` is the human entry point; there is no `codex:codex-rescue` skill). The orchestrator then applies § Orchestrator Handoff Discipline to that deliverable exactly as for a Claude subagent's output.
- **Prerequisite.** Codex CLI must be ready. Run `/codex:setup` once before the first delegated step; if the rescue runtime is unavailable mid-phase, insert a blocker phase (Hard Rule 4) — do **not** silently reassign backend work to a Claude agent without recording it.
- **Briefing contract.** The rescue prompt IS Codex's contract: phase number/title/scope, current gate, the BE work items + their interface contract, required reads, allowed write scope (backend production code only in full mode; backend code + mock tests in Fast Mode), and expected output.

### Orchestrator Handoff Discipline

The orchestrator dispatches each gate to the actor named in the matrix — the `architect`, `validator`, and `guardian` subagents, the `builder` subagent for frontend code, and the `codex:codex-rescue` subagent for backend code; `worker` handles simple work outside the matrix (Hard Rule 1). Subagents may Read/Grep/Glob, run read-only Bash, and WebFetch/WebSearch, and `SendMessage` the orchestrator to clarify, escalate, or hand back; the orchestrator dispatches Claude subagents via `SendMessage` and invokes Codex for backend work via the **Agent tool** (`subagent_type: "codex:codex-rescue"`), never via `Skill()`. Per-subagent model/effort, where the harness supports it, is set in each `.claude/agents/<name>.md` frontmatter — not pinned here.

**Between every step transition, the orchestrator MUST fully read the deliverable that just landed before dispatching the next actor.** "Read" means: **use the `Read` tool** on the file end-to-end (or at minimum: every BLOCKER + CONCERN-MR finding's full body, plus the verdict + routing-decision section). **Bash `grep` / `head` / `tail` / line-count is NOT enough.** This applies equally to deliverables returned by Codex.

**Enforcement:** the dispatch prompt to the next actor MUST quote at least one specific finding (file:line cite, exact recommended fix wording, or verbatim verdict rationale) as proof the deliverable was read. Hand-waving like "found 3 issues, route to fix" is a violation. The dispatch prompt IS the next actor's contract — routing blindly produces garbled context (wrong file:line citations, missed CONCERN-MRs, scope drift) and a mis-route costs a full step rework.

The rule above covers most transitions (read the just-landed deliverable end-to-end, quote a finding). These transitions carry **extra, non-obvious threading requirements**:

| Transition | Extra requirement |
|---|---|
| Step 2 (validator) → Step 3 (guardian critic) | Thread the scaffold list + plan into the critic's brief — the critic reviews **both** the plan and the TDD scaffolds. |
| Step 3 (guardian) → Step 3a / Step 0 / Step 4 | Categorize all BLOCKER / CONCERN / NIT findings; thread CONCERN-MR items + the routing decision (revise plan/TDD, re-explore, or proceed) into the next dispatch. |
| Step 4 dispatch (FE ∥ BE) | Brief `builder` with FE items + scaffolds and Codex with BE items + interface contract; confirm disjoint write ranges; quote a specific plan/critic finding to each. |
| Step 5 (validator) → Step 5a / Step 6 | Record per-gate verdicts; route FE defects to `builder`, BE defects to Codex. |

In **Fast Mode** the same discipline applies at FM-0→FM-1 (read the plan), FM-1→FM-2 (read the critic verdict; brief FE/BE in parallel), and FM-2→FM-3 (read both diffs before live verification).

**The only exceptions** are during initial setup (before Step 0 of phase 1) or when the deliverable is unambiguously trivial (e.g. a 5-line `.gitignore` patch from worker — `cat`-level scan suffices). Use judgment but bias toward reading.

### Auto Mode

When the user says `auto mode`:

1. Re-read this `CLAUDE.md` and `ROADMAP.md` (also on resume, interruption, or context compaction).
2. Continue the current phase, or insert the smallest blocker-resolution phase if blocked (including Codex rescue runtime unavailability).
3. Follow the active mode's gates exactly; do not advance until all gates pass.
4. **Step 0 / FM-0 direction gate:** the orchestrator stands in for the operator and decides the chosen route itself, then writes it into `ROADMAP.md` — it does **not** halt for sign-off here (normal mode halts; auto mode does not).
5. **Auto-commit** on the mode's final step (Step 7 / FM-4) per Commit Policy above.
6. **Halt at operator-validation boundaries** (per Hard Rule 9) for explicit operator validation; do not auto-advance across major version gates.
7. **Write a § Session Handoff** to `ROADMAP.md` before the session ends or context fills, and tell the user — so the next session resumes cleanly.
8. **Never self-edit `CLAUDE.md`** without explicit operator approval (Hard Rule 10), even when running unattended.

---

**Working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, clarifying questions come before implementation rather than after mistakes, every phase ends with a clean commit, every load-bearing security boundary holds across every PR.
