# CLAUDE.md

Behavioral guidelines and multi-agent process rules for this codebase.

**Tradeoff:** Bias toward caution over speed. For trivial tasks, use judgment.

## Project Scope

_Reserved for per-repository scope. Populate with:_

- **Product goal** — what this codebase ships and to whom.
- **Approved capabilities** — the libraries, frameworks, runtime surfaces, and external interfaces the agent may use.
- **Owned components** — paths under this repo that the agent edits (`src/`, `tests/`, configs, README, etc.) plus the published product contract (bin entrypoint, exports, on-disk schemas).
- **External authoritative systems** — frameworks/SDKs/services treated as external truth (pin versions, verify behavior against docs, do not silently upgrade).
- **Explicit non-goals** — features and integrations that are out of scope; record retired/abandoned directions with date + rationale.
- **Live environment** — what the agent operates against during verification (real user account, real browser, real API), and which phases are read-only vs. full live windows.

If any of these are unclear, resolve them in the Research gate (§2 → Phase Gate Matrix, Step 1) before implementation.

---

## How to use this doc

| Situation | What to read |
| --- | --- |
| Simple task (one obvious edit) | Project Scope + §1 Coding Behavior → dispatch to `worker` |
| Non-simple task | §2 Process Governance |
| `auto mode` trigger | §2 → Auto Mode |
| Resume / interruption / context compaction | Re-read this file + `ROADMAP.md` |

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
- Any load-bearing security claim (e.g. a no-bash / no-shell-out boundary at the agent's tool layer, capability allowlist, credential isolation) is non-negotiable: changes that weaken it are Phase Gate Matrix items, never worker tasks.
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
- **Live verification:** every implementation phase that touches an external authoritative system includes platform-level smoke tests against the real target. Mock-only is insufficient unless `ROADMAP.md` defines the phase as research/prototype-only.
- **Interactive / end-to-end testing:** when an agent must be driven end-to-end (full LLM loop + real tools) for a live verification, use the project-sanctioned harness (e.g. a real CLI launched inside a tmux pane and steered with `tmux send-keys` / `tmux capture-pane`). Ground-truth every result against the audit log and direct evidence from the external system — never trust UI text alone.
- **Real-agent vs. tool-smoke distinction:** a *live test of the agent* MUST drive the full agent loop (system prompt + LLM + the agent's own tool selection). A test that invokes tools directly bypassing the LLM is a **tool/integration smoke** — it verifies the tool layer, not that the agent works, and MUST NOT be counted as agent live-verification.
- **Tool-gap rule:** if the real agent cannot complete a live-test task because a tool is missing or inadequate, the `validator` RECORDS and REPORTS it as a finding — never papers the gap over with a test-side workaround. The **orchestrator** evaluates the gap and decides whether a new/extended tool is warranted; tool inventory changes are a Phase Gate decision, never a validator one.
- **External authoritative systems cross-check:** treat external SDKs/protocols/services as authoritative — pin versions, verify behavior against docs and source, do not assume features exist without dependency-level confirmation. Do not silently upgrade across phases.

### Test Discipline (outside-in TDD + BDD-light)

**Outside-in TDD**: validator writes failing test scaffolds **BEFORE** builder implements (Phase Gate Matrix Step 4a precedes Step 4b). Scaffolds compile + all assertion bodies are TODO; tests intentionally fail. Builder's job at Step 4b is to make scaffolds compile + reach the assertion-TODO branch (no need to make TODO assertions pass yet — those are filled at Step 5). Validator's Step 5 fills assertion bodies + adds edge cases + runs the full suite.

Why: the test scaffold IS the contract. Writing scaffolds before code prevents validator unconsciously aligning tests to whatever builder happened to implement. Architect's §5 plan must list **testable behaviors** (concrete pre/post conditions), not abstract gates. The pre-written scaffold catches builder/spec mismatch at Step 4b not at Step 5a rounds later.

**BDD-light** (independent of TDD; adopted together):
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

**`docs/phase-N-test.md` is two-pass.** Validator writes it twice in a single phase:
1. **Step 4a** (scaffold-time): test contract — list of test names + intent comments + which gates they cover. Tells builder + guardian what behaviors are about to be verified.
2. **Step 5 final** (post-run, after all 5a iterations): test report — actual results, defects found, gate verdicts, residual risks. Appended as additional sections; the Step-4a test contract stays in the doc as historical record.

The accumulated `phase-N-test.md` is self-contained: contract → results trail.

### Commit + Release Policy

- **Auto-commit on phase completion:** when Step 7 of the Phase Gate Matrix marks a phase complete, the orchestrator creates a git commit summarizing that phase's work.
- When preparing a commit, list every changed file with a one-sentence description of its change.
- Commit attribution trailer is project-defined (record it in this section if used).
- Commits never include credential values. Credential / secret files stay outside git (gitignored). If an agent surfaces a credential in any docs, log, or commit message, treat it as an incident and rotate the credential.
- **Step 7 sequence:**
  0. **Bump `package.json` `version` (or equivalent version manifest) to match `<next-version>` (without the `v` prefix).** If a startup auto-updater reads the manifest, drift causes false "behind" detection and infinite update loops on global installs. The bump must be part of the phase commit.
  1. `git add` + `git commit` with phase summary.
  2. `git tag -a <next-version> -m "<phase title>"` — annotated tag; body becomes the GitHub Release notes.
  3. `git push origin main` then `git push origin <next-version>` — tag push fires the release workflow which auto-creates a GitHub Release.

Concrete version assignment is owned by `ROADMAP.md` § Version sequence — orchestrator reads ROADMAP at Step 7 to pick the next tag.

### LLM Configuration

- **Default model:** pinned in code/config; switching providers does not require a new release. Record the verified default alias key here when populating per-project.
- **Precedence (suggested):** factory `model:` parameter > CLI `--model <provider:modelId>` flag > env var > on-disk auth config `default` field > hardcoded fallback.
- **Provider auth:** env var (e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) OR managed auth config file. Providers that piggyback on another's protocol (e.g. DeepSeek via OpenAI-compatible `baseURL`) are recorded here.
- **Inline `.env` reader at CLI startup** is acceptable (no runtime `dotenv` dependency needed); provide an opt-out env var.

> Per-project env-var enumeration belongs in `ROADMAP.md` § Env-var reference — a per-phase-accreting catalogue belongs with phase tracking, not in CLAUDE.md (which holds stable code/project requirements + process).

---

## 2. Process Governance

### Roadmap

- Canonical path: `ROADMAP.md`. Read before meaningful work; create if missing before non-simple implementation.
- Every phase completion and every blocker must update the roadmap with: status, scope, gates, live verification target, blockers, next action.

### Hard Rules

1. Simple requirements (e.g. a single obvious edit) may be handled directly by `worker`.
2. Non-simple requirements must use the Phase Gate Matrix; no step starts until the previous passes; no phase starts until the previous completes Step 7.
3. No implementation before research, plan, and critic approval.
4. Blockers must be inserted into `ROADMAP.md` as a blocker phase before being worked around.
5. Subagents are serial by default. Parallel work is allowed only after the main plan is frozen and the tasks have independent concerns or disjoint write ranges.
6. Source classification, contract impact, and external-system runtime behavior must be resolved before implementation begins.
7. Resource-exclusive verification (e.g. only one browser instance, only one DB lock) must run serially.
8. Any load-bearing security boundary defined in §1 → Product Contract is non-negotiable **at the agent / tool layer**. Operator-initiated CLI-layer exceptions (narrow maintenance commands) require Phase Gate Matrix Steps 1–7 with explicit operator approval at Step 3, and a CLAUDE.md amendment recording the approved use case. The agent's LLM-callable tool surface remains within the boundary without exception.
9. **Operator validation gates between major versions:** record explicit operator sign-off points (e.g. v0.3 → v0.4 → v1.0). Auto-mode must halt at these boundaries.

### Phase Gate Matrix

**Non-simple work runs these gates in order.** The orchestrator tells every subagent: phase number, title, scope, current gate, required reads, allowed writes, expected output. **Every agent must read the phase-N entry in `ROADMAP.md` before doing its step** — the entry defines goal, scope, gates, live verification target, blockers, next action.

**Outside-in TDD:** Step 4 is split into 4a (validator scaffold) and 4b (builder implementation). See § Test Discipline above for the full rationale.

| Step | Gate | Agent | Required Reads¹ | Allowed Writes² | Output | On Fail |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Roadmap Entry | orchestrator | prior phase docs (if any) | roadmap | new phase-N entry in `ROADMAP.md` | → 1 |
| 1 | Research | `scout` | — | docs | `docs/phase-N-research.md` | → 2 |
| 2 | Plan | `architect` | research doc | docs | `docs/phase-N-plan.md` (§5 must list **testable behaviors** with concrete pre/post conditions, not abstract gates) | → 3 |
| 3 | Critic | `guardian` | research + plan | docs | `docs/phase-N-critics.md` | → 3a or 3b |
| 3a | Research Revision | `scout` | critics doc | docs | revised research | → 2 or 3 |
| 3b | Plan Revision | `architect` | critics doc | docs | revised plan | → 3 |
| **4a** | **Test Scaffold** | **`validator`** | plan §5 + §6 sketches | test code (scaffolds — compileable, all-failing, named per §5; assertion bodies are TODO) + docs (initial `docs/phase-N-test.md` § Test Contract: name + GWT intent comment + which gates each covers) | scaffolds compile + all fail; test contract doc | → 4b |
| **4b** | **Implementation** | **`builder`** | plan + 4a scaffolds | code (production only — NO test files) | scoped code; scaffolds compile + reach assertion-TODO branch (no need to satisfy TODO assertions yet — those are filled at Step 5) | → 5 |
| 5 | Validation | `validator` | plan + builder diff + own scaffolds | test code (fill assertion bodies + add edge cases) + docs (append § Results to `docs/phase-N-test.md`) | mock + live tests + final test report (defects, gate verdicts, residuals) | → 5a |
| 5a | Validation Fix | `builder` | test + plan | code (production only) | scoped fixes | → 5 |
| 6 | Final Review | `guardian` | all phase docs | docs | `docs/phase-N-review.md` | → 5a |
| 7 | Roadmap Update + Commit | orchestrator | all phase docs | roadmap + git | phase-N entry marked complete; **`git commit` + `git tag` + `git push origin main` + `git push origin <tag>` (workflow auto-publishes GitHub Release)** | phase complete |

¹ **Implicit read for every step:** the phase-N entry in `ROADMAP.md`. The "Required Reads" column lists additional inputs on top of that.

² **Write scopes** — `docs`: reports under `docs/` only. `test code + docs`: may write mock + live test code in `tests/**` + reports under `docs/`; no production code edits. `code`: production files within approved phase scope only — **builder MUST NOT write test files (mock or live); that is validator's exclusive scope at Steps 4a + 5**. If builder believes a test is needed during Step 4b, hand it back to validator via SendMessage to the orchestrator. `roadmap + git`: only `ROADMAP.md` + git operations (commit/tag/push).

### Agent Roster

**Six agents — five specialized + one general-purpose.** Specialists run inside the Phase Gate Matrix. `worker` is the only general-purpose agent, sits outside the matrix, and handles simple work per Hard Rule 1. All agents may Read/Grep/Glob, run safe Bash (read-only), and WebFetch/WebSearch. All agents must `SendMessage` to the orchestrator (clarify, escalate, hand back); the orchestrator must `SendMessage` to any teammate.

**Effort levels**: if the harness exposes only a single global `effortLevel` knob, all agents inherit it. The per-agent effort column below is **design intent** — encode it in `.claude/agents/<name>.md` frontmatter once per-agent effort is supported.

| Agent | Steps | Write Permissions | Model | Effort |
| --- | --- | --- | --- | --- |
| `scout` | 1, 3a | docs | `claude-sonnet-4-6` | xhigh |
| `architect` | 2, 3b | docs | `claude-opus-4-7` | xhigh |
| `guardian` | 3, 6 | docs | `claude-sonnet-4-6` | xhigh |
| `builder` | 4b, 5a | code (within phase scope) | `claude-opus-4-7` | xhigh (harness limit; design intent: medium) |
| `validator` | 4a, 5 | test code + docs | `claude-sonnet-4-6` | xhigh |
| `worker` | (outside matrix) | code (simple tasks only) | `claude-sonnet-4-6` | xhigh |
| orchestrator | 0, 7 | roadmap + git | — | — |

### Orchestrator Handoff Discipline

**Between every Phase Gate Matrix step transition, the orchestrator MUST fully read the deliverable that just landed before dispatching the next agent.** "Read" means: **use the `Read` tool** on the file end-to-end (or at minimum: every BLOCKER + CONCERN-MR finding's full body, plus the verdict + routing-decision section). **Bash `grep` / `head` / `tail` / line-count is NOT enough** — those only confirm file presence and snippet shape, not content understanding.

**Enforcement:** the dispatch prompt to the next agent MUST quote at least one specific finding (file:line cite, exact recommended fix wording, or verbatim verdict rationale) as proof the deliverable was read. Hand-waving like "Guardian found 3 issues, route to fix" is a violation.

**Why:** the dispatch prompt IS the next agent's contract. Routing blindly produces garbled context (wrong file:line citations, missed CONCERN-MRs, scope drift). Reading is cheap; mis-routing cascades cost a full step rework.

Apply at each transition:

| Transition | Orchestrator must read before dispatch |
|---|---|
| Step 1 (scout) → Step 2 (architect) | `docs/phase-N-research.md` end-to-end; surface any BLOCKER-RISK / OQ items into architect's dispatch as required-action items. |
| Step 2 (architect) → Step 3 (guardian) | `docs/phase-N-plan.md` end-to-end; verify §6.4 locked code sketches exist; note any concerns to flag for guardian's specific items-to-verify list. |
| Step 3 (guardian) → Step 3a (scout) / 3b (architect) / 4a (validator) | `docs/phase-N-critics.md` end-to-end; categorize all BLOCKER / CONCERN / NIT findings; thread CONCERN-MR items into the next dispatch's instructions. |
| Step 4a (validator) → Step 4b (builder) | Read initial `docs/phase-N-test.md` § Test Contract end-to-end; verify scaffolds list matches plan §5 testable-behaviors; thread test-name list into builder's dispatch so builder knows which behaviors must compile-pass. |
| Step 4b (builder) → Step 5 (validator) | Run project-defined check/lint/build; verify scaffolds compile + reach assertion-TODO branch; thread any builder-flagged drift or pre-existing issues into validator's dispatch. |
| Step 5 (validator) → Step 5a (builder) / Step 6 (guardian) | `docs/phase-N-test.md` end-to-end; record test pass counts + per-gate G-PN.x verdicts in dispatch context. |
| Step 6 (guardian) → Step 7 (orchestrator self) / 5a (builder) | `docs/phase-N-review.md` end-to-end; if ACCEPT, proceed to Step 7; if REJECT, route back with full BLOCKER list. |

**The only exceptions** are during initial setup (before Step 0 of phase 1) or when the deliverable is unambiguously trivial (e.g. a 5-line `.gitignore` patch from worker — `cat`-level scan suffices). Use judgment but bias toward reading.

### GitHub Issue Intake

New GitHub issues do NOT automatically become phases. Every issue must pass a three-stage triage before it enters `ROADMAP.md`:

1. **Reproduction — `scout`.** Reproduce the reported behavior against the relevant authoritative system. Produce a short intake note under `docs/issue-N-intake.md` containing: exact command(s) run, observed JSON/behavior, expected behavior per issue, and whether reproduction succeeded.
2. **Evaluation — `guardian`.** Read scout's intake note plus the relevant source. Decide: is this truly a defect in this codebase? An upstream issue in an external authoritative system? A regression in the target environment (possibly mitigable here)? User/agent misuse already documented? Out of scope (non-goal)? Record the classification in `docs/issue-N-intake.md` with a file:line justification.
3. **Decision — orchestrator (team-lead).** Read scout + guardian output. Choose exactly one:
   - **Accept → new phase.** Insert a phase into `ROADMAP.md` (In Progress or queued), then run the Phase Gate Matrix from Step 0.
   - **Defer.** Record under `ROADMAP.md` → "Incomplete / Deferred" with the intake note linked.
   - **Reject.** Close the issue on GitHub with a one-line rationale pointing to the intake note.

**Hard rule:** no phase may be opened for a GitHub issue before all three stages complete. `worker` may only handle an issue directly if it is a simple requirement per Hard Rule 1 AND the orchestrator has classified it as such after triage.

### Auto Mode

When the user says `auto mode`:

1. Re-read this `CLAUDE.md` and `ROADMAP.md` (also on resume, interruption, or context compaction).
2. Continue the current phase, or insert the smallest blocker-resolution phase if blocked.
3. Follow the Phase Gate Matrix exactly; do not advance until all gates pass.
4. **Auto-commit on Step 7** per Commit Policy above.
5. **Halt at operator-validation boundaries** (per Hard Rule 9) for explicit operator validation; do not auto-advance across major version gates.

---

**Working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, clarifying questions come before implementation rather than after mistakes, every phase ends with a clean commit, every load-bearing security boundary holds across every PR.
