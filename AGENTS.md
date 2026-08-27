# AGENTS.md

Behavioral guidelines and multi-agent process rules for this codebase — Codex-native edition. The engine is Codex throughout: the orchestrator is Codex's main loop and spawns its own subagents for each gate. Structurally this mirrors `CLAUDE.md`, minus the cross-engine delegation (no "codex rescue"): every role is a normal Codex subagent.

## Project Scope

Write one precise, concise paragraph describing the repository's current scope: what the product ships, who it serves, and the core product and technical boundaries agents must preserve. Keep only durable present-tense facts needed for implementation decisions; do not turn this section into a decision log, roadmap, dated history, retired-direction archive, or exhaustive component inventory. Resolve scope ambiguity during Step 0 exploration (§2 → Phase Gate Matrix, Step 0), and record the resulting live decision in `ROADMAP.md`, not here.



---

## How to use this contract

| Situation | Required route |
| --- | --- |
| One genuinely obvious edit | Project Scope + Coding Behavior → the executing actor (per the active execution mode) edits and verifies directly |
| Non-simple work | Coding Behavior → Step Chain; the executing actor runs every non-review step, in full or fast depth |
| `auto solo mode` / `auto team mode` | Coding Behavior → Execution Modes → Step Chain / Auto Mode |
| Near context limit or deliberate stop | Coding Behavior → Session Handoff |
| Resume, interruption, or compaction | Re-read this file and `ROADMAP.md`, including its handoff |

The Coding Behavior section applies to every actor that writes code and to the narrowly scoped
critic/audit reviewers; its process-governance sub-sections govern non-simple work.

---

## Coding Behavior

- **Operator engineering principles (2026-08-07, 8 rules — apply to every implementation decision; the
  existing anti-overengineering rule below is the strengthened execution detail):**
  1. **Do not preserve backward compatibility.** Delete obsolete paths instead of adding compatibility
     layers, backward-compatible migrations, or fallbacks. Interpretation confirmed by the operator:
     migrating old data into the new format in order to delete legacy readers/shapes is allowed and
     encouraged. A strangler-fig compatibility layer is an explicit temporary exception only when it has a
     recorded deletion plan/deadline and is removed as soon as that condition is met.
  2. **Choose the simplest implementation that satisfies the current requirement.** Do not add preventive
     abstractions or unnecessary configuration layers.
  3. **Grow the system in layers.** First land the smallest working end-to-end slice, then add capability;
     never dismantle a working slice for complexity that is not yet complete.
  4. **Keep components modular and separate concerns.**
  5. **Prefer mature, maintained libraries.** Do not rewrite a capability without a concrete reason.
  6. **Inspect what the project's existing dependencies already provide** before adding a package or
     implementing the capability from scratch.
  7. **Make architecture decisions for the long term.** Do not accept a knowingly temporary “replace it
     later” implementation.
  8. **Study how mature products solve the same problem first** and use proven patterns instead of inventing
     from zero.
- **Interface-contract custody.** Every operator/product-facing interface change must update the project's
  authoritative interface contract in the same commit; the task's independent review must verify that diff
  or explicitly verify that the change has no interface effect.
- **No praise; critically evaluate.** Do not flatter an operator proposal. Independently assess whether it
  is sound, plainly identify disagreement or a cheaper alternative, present real options, or ask a direct
  question when ambiguity blocks safe action. Agreement must follow reasoning.
- **Think before coding.** State assumptions. If multiple interpretations exist, present them instead of
  silently choosing. Surface simpler approaches. Stop for operator direction when the choice materially
  changes scope or outcome.
- **Simplicity first.** Write the minimum code that solves the stated problem. Avoid speculative features,
  abstractions, configurability, and impossible-case handling. If 200 lines can be 50, simplify.
- **不要过度设计,宁愿多写边缘 test code (operator 2026-07-25).** Effort belongs in VERIFICATION, not in
  production mechanism. Concretely:
  1. **Production code = the simplest thing that satisfies the contract.** When a mechanism turns out to
     have a flaw, first ask whether to delete that mechanism, not whether to add a layer guarding it.
     Adding machinery to patch machinery is the failure mode this rule exists to stop.
  2. **Edge coverage is where extra effort is explicitly wanted** — boundary values, negative cases, crash
     cutpoints, concurrent races — and every carrier must drive the real production path through an
     observable seam (op-log order, health transitions, committed goldens), never a hand-set flag and never
     end-state-only. Include at least one test proving the checker itself can fail: a check that cannot
     go red is not a check.
  3. **Tie-break rule:** when a critic finding can be answered either by new production machinery or by a
     test proving that edge is already handled — choose the test, and name the carrier in the reply.
  4. **Guard hard on exactly two failure modes: data loss, and a check that passes when it should fail**
     (silent wrong result). Everything else gets ordinary care, not a subsystem.
  5. If a critic finding genuinely requires a new subsystem, push back with reasoning; if the critic
     insists, escalate to the operator — the executing actor and the critic do not get to ratchet scope
     between themselves.
- **🔴 A carrier must be able to go RED — prove it, never reason about it (operator 2026-07-26).** For
  every carrier, name in the test doc the injected defect that mechanically forces it red, then verify it
  by injection: apply the defect, run the carrier, observe red, revert. Reasoning that it "would" fire is
  not evidence. If injecting the stated defect does not turn the carrier red, either the trigger is wrong
  or the carrier is — find out which and say which. Ask it of every carrier mechanically, not only the ones
  that feel risky. Watch the mirror too: a carrier asserting something unreachable can never go green, is
  equally useless, and is harder to spot.

- **🔴 A ⊖ (must-not) carrier ships with a POSITIVE CONTROL, and the control must be able to go GREEN
  (operator 2026-07-26).** A carrier asserting that something must not happen passes trivially whenever the
  mechanism never fires at all — so injecting its defect turns it red for the wrong reason, which reads
  exactly like success. Pair every ⊖ vector with a positive control in the same test, and verify the
  control can actually reach green; a control that cannot is not a control. **The pair is the carrier —
  neither half alone is.**
- **🔴 Before reporting a gate GREEN, state the class of defect that gate CANNOT detect (operator
  2026-07-26).** Every gate has a blind spot by construction, and an unqualified PASS invites the reader to
  treat it as total. Name the blind spot next to the number.
- **🔴 When a change alters the SHAPE of a persisted structure, the blast radius is every reader that
  VALIDATES that structure — not the readers of the field you changed (operator 2026-07-26).** A
  field-value search is the wrong instrument and comes back clean.
- **🔴 A closing condition that cannot fire is a check-that-cannot-fail relocated into a contract (operator
  2026-07-26).** Never record a bounded allowance, deprecation, or migration whose stated end condition
  cannot be reached. If the population it drains can still grow, the bound is decoration: say plainly that
  the allowance is permanent until an explicit migration runs, and specify that migration. An honest
  permanent exemption beats a decorative bound.
- **🔴 Nothing is described in the PAST TENSE until it is committed (operator 2026-07-26).** A doc
  asserting a state of the tree is a claim about the tree and must be verified against it exactly like a
  test premise. Everything not yet committed goes under an itemized **UNLANDED** list with file paths.
- **🔴 Reconcile gate totals before believing a green run (operator 2026-07-26).** `passed + skipped +
  todo` MUST equal the reported total, and `Test Files` likewise. A mismatch means real failures or a lost
  worker — never a clean run.
- **Surgical changes.** Touch only required lines, match local style, do not restyle adjacent code, remove
  only dead code created by this change, and preserve operator work. Every changed line must trace to the
  request.
- **Code navigation.** Repo source navigation uses the `codegraph_*` index (search / callers / callees /
  impact / explore) in place of `rg` text search and manual caller tracing for symbols, callers,
  construction sites, and value flow. `rg` and direct reads cover literal constants, generated names,
  unindexed files, and non-code assets such as config, PDFs, spreadsheets, builds, and node operations.
- **Goal-driven execution.** Convert requests into verifiable outcomes and state a short numbered plan with
  a verification point per step. Continue until the stated gates pass or a concrete blocker is recorded.
- **Code and test policy.** Source files stay ≤800 lines; generated, vendored, and fixture files are exempt.
  Every behavior change ships mock tests, and mock tests run before live. Work touching an authoritative
  external system requires a real-target smoke test unless `ROADMAP.md` explicitly marks it research-only.
  An orchestrator-driven live run drives the real loop; a direct tool call proves only the integration. If a required
  tool is absent or inadequate, record and report the gap rather than hiding it in tests. Pin external
  systems and verify behavior against their authoritative documentation.
- **Rendered-UI verification.** Every FE behavior change requires a real rendered interaction against the
  designated live validation environment, with the exact data, route, actions, expected result, visual evidence,
  and runtime errors recorded. Unit tests, raw HTTP checks, and a successful build are necessary but do not replace
  this gate.
- **BDD-light tests.** Test names describe behavior, each test has a one-line Given/When/Then intent
  comment, and suites use `describe(behavior) { it(scenario) }`. Do not introduce Gherkin, `.feature`, or
  Cucumber.
- **Commit and release.** On task completion the orchestrator lists every changed file with a one-sentence
  description. In team mode it approves and performs the merge at the commit gate; in solo mode it stops at
  that gate until the operator explicitly authorizes the task's merge. All commits are solely attributed to
  the operator with no attribution trailer. Never commit credential values; a leaked credential is an
  incident and must be rotated.
  Remote sync occurs only on operator instruction. `ROADMAP.md` owns the version sequence.
  After authorization, merge the accepted worktree into `main` before any release. Release and whole-main
  regression gates run from clean `main`, never from the task branch; a PROD go is requested only after the
  merged candidate is live and green on persistent DEV.

### Roadmap

`ROADMAP.md` is canonical live state and must be read before meaningful work. Create it if absent before
non-simple work. Every task completion and blocker records status, scope, gates, FE/BE classification,
depth, execution mode, live-verification target, blockers, and next action. Durable core/process lives
here; live progress lives in `ROADMAP.md`.

### Session Handoff

Near the context limit or before a deliberate stop, the orchestrator writes one `## Session Handoff` block in
`ROADMAP.md`, replacing the previous one. Include the active task and step/depth/mode, what landed, the exact
next action, blockers, any active critic/audit reviewer and its exact worktree/write fence, any in-flight
owner sub-agents and their worktrees (team mode), and uncommitted state. Tell the operator. On resume
or compaction, re-read this entire file and `ROADMAP.md` before acting.

### Roadmap Compaction

When `ROADMAP.md` grows large, it is compacted mechanically: summarize completed work to title, outcome, key
decisions, tag, and live-verification result; preserve the version sequence, every open blocker/deferred
item, the current task in full, and the newest Session Handoff; the pre-compaction detail stays in git
history. In **solo mode** the orchestrator performs the compaction directly and may not delegate it.
In **team mode** it may be dispatched as one light mechanical owner sub (no step chain) whose output the
orchestrator reviews before the compaction commit. It is never delegated to a review sub-agent.

### Execution Modes

**The executing actor handles both sides of every task directly** (operator 2026-07-13); it keeps FE/BE
sections and write ranges separable and fixes their interface contract before implementation. Who the
executing actor is depends on the execution mode below.

Non-simple work runs in exactly one of two execution modes. **There is no default: the operator's
instruction names the mode.** If a non-simple or auto instruction arrives without a mode, stop and ask the
operator (or record a blocker) before proceeding.

- **Solo mode.** The orchestrator is the sole executor of all non-review work: decomposition,
  scout, scaffolding, implementation, validation, live/visual gates, roadmap updates, and commits. It may
  parallelize independent read-only commands or tests, but it may not parallelize or delegate
  implementation to sub-agents. Only the critic and audit steps are delegated, to the same fresh independent
  reviewer agent. After full acceptance it stops at the merge gate until the operator explicitly
  authorizes that task's merge; it may not self-approve the merge.
- **Team mode.** The orchestrator only coordinates: decompose → dispatch → read deliverables → FE visual
  gate → audit the owners' live evidence → full acceptance gate → approve and merge in dependency order →
  cleanup. It never writes production
  code, plans, scaffolds, or tests itself. Each task is dispatched to **one owner sub-agent** that owns
  the task end-to-end at **fast depth** in the foreground in its own isolation worktree, through build,
  validation, and live acceptance evidence. Team-mode owner scope does **not** include critic or audit:
  the owner retains and hands its complete live evidence to the orchestrator, and the orchestrator performs
  the final audit and acceptance decision. The owner makes its own design decisions without checking back
  mid-task. Tasks may run in parallel only when their write ranges are disjoint; parallelism lives at the
  task level, never inside a task. ⚠️ Never run two tasks that touch the same area concurrently — the
  loser builds on a structure the winner is about to replace.

- **Execution isolation.**
  - **One task = one independent worktree; merge only on full acceptance (operator 2026-08-11, mode authority
    clarified 2026-08-11).** A
    complete task runs in its own dedicated git worktree, never directly in the main checkout. A merge is
    permitted only after the task worktree's acceptance reaches merge standard: mock/focused tests, the
    task-scoped `src`+`web` gates, a real `next build`, numbered shadow-DEV verification, the FE visual gate
    for FE changes, and the post-live audit. After merge, persistent DEV plus the integrated full-main
    baseline is a separate mandatory release gate before PROD. **In team mode, the orchestrator approves and performs the merge; no separate operator go is
    required. In solo mode, the orchestrator may not self-approve or perform the merge until the operator
    gives an explicit merge instruction for that task.** An owner, critic, or auditor never merges its own
    work. After a successful merge the actor that performed it removes the worktree
    (`git worktree remove --force <path>`).
  - **Team mode:** every owner sub-agent runs in its own git worktree (a fresh tree off current
    `main`). Discipline: branches stay clean (never reset or stash a shared tree); merges run in
    dependency order at the commit gate with the full `src`+`web` build between merges; dependencies are
    refreshed after any merge that adds them; uncommitted work is committed before anything is discarded;
    a sub is shut down once merged; collected-but-unmerged worktrees are removed explicitly.
  - **Review isolation is explicit, not automatic.** In solo mode, review steps may be delegated only to
    critic and audit reviewers. Team mode's one-owner assignment delegates the whole fast-depth task through
    live evidence, not an individual step; the owner does not run or delegate critic/audit. Collaboration sub-agents share the
    working directory and filesystem; their
    edits become visible immediately, and spawning one does not promise a new branch or worktree. The
    spawner gives each reviewer the exact repository/worktree path, a read scope, and a write fence
    limited to that review's required document. Review steps run only after the preceding step is
    complete, and the spawner waits for each reviewer before resuming writes in overlapping paths. Never
    reset, stash, or discard unknown work. If a dedicated review worktree is created, remove it after the
    review is collected so worktrees do not accumulate.

### 🔴 Concurrent-worktree hazards

- **Use ABSOLUTE paths for every command.** The shell cwd drifts silently between agents sharing the
  filesystem. Every participant sets its assigned path explicitly on each command.
- **NEVER run `scripts/depgate.mjs` from a worktree whose `node_modules` are symlinked, and NEVER set
  `CI=true` there.** `depgate.mjs` shells out to **`pnpm`**, which treats a symlinked modules dir as foreign
  and tries to **REMOVE** it — with `CI=true` that includes the main
  checkout's `node_modules`, which every sibling worktree points at. Any script that shells out to `pnpm`
  runs in the main checkout only; symlinked `node_modules` remains fine for `tsc`/`vitest`. A `depgate`
  failure observed from such a worktree is this artifact, not a gate violation — report it as a tool
  gap, never "fix" the non-defect. State the pnpm prohibition in every review brief that uses a worktree.

### Hard Rules

1. A genuinely single obvious edit is performed directly by the executing actor (the orchestrator in
   solo mode; one owner sub in team mode) without the Step Chain.
2. Non-simple work follows the Step Chain in full or fast depth. No step begins before the prior deliverable
   exists and has been read.
3. **Independent review is mandatory in solo mode and never replaced by self-review.** In **full** depth, no build begins
   before `scout` and a fresh independent critic pass. In **fast** depth there is no pre-build critic
   (operator 2026-08-11); the mandatory independent check is the **post-live audit** (see Step Chain step 6)
   — fast may never drop the audit either. **Team mode is the explicit exception:** every owner completes
   fast depth through live evidence without critic/audit, then the orchestrator audits that evidence and
   decides acceptance. Only a truly obvious Hard Rule 1 edit is otherwise exempt from independent review.
4. Record blockers in `ROADMAP.md` before attempting a workaround, including unavailable reviewer
   runtimes or collaboration mechanisms.
5. Parallelism follows the active execution mode: in solo mode the orchestrator is the sole
   executor and never assigns tasks or write ranges to implementation sub-agents; in team mode
   task-level parallelism is allowed only with disjoint write ranges (see Execution Modes).
6. Resolve source classification, contract impact, FE/BE interface, and external runtime behavior in
   `scout` before build.
7. Run resource-exclusive work serially; concurrent actors must not contend for the same exclusive runtime,
   stateful lock, shared measurement source, or service lifecycle.
8. Security-boundary changes require full depth and explicit operator approval at critic. The
   model-callable tool surface remains inside its existing boundary unless the operator explicitly changes it.
9. Record operator validation gates between major versions; auto mode stops at them.
10. **`AGENTS.md` is the durable project core and operating contract** (`CLAUDE.md` is a symlink to it). No
    agent may create, edit, or self-update it without explicit operator approval. Live state belongs in
    `ROADMAP.md`.
11. Execution-mode discipline: in solo mode the orchestrator executes every task end-to-end and may
    not delegate any non-review activity; in team mode the orchestrator assigns each whole task to one
    owner sub at fast depth, never performs an owner task step itself, and retains the audit,
    acceptance, and merge decision. A team-mode owner does not run or delegate critic/audit.
12. **In solo mode, within a task only critic and audit review steps are delegated, and only to the same fresh independent
    external reviewer agent.** Team mode's whole-task owner assignment is the sole non-review assignment allowed;
    its owner hands retained live evidence to the orchestrator for audit instead of invoking a reviewer.
    The critic and the auditor are the same agent — one reviewer serves both review steps of a task, whether it
    is spawned as a native sub-agent (Codex CLI) or obtained through the **`/codex:rescue`** route. That reviewer
    may not be repurposed into implementation or another non-review role. Follow-up messages may address
    only the reviewer already assigned to that step. If no review mechanism is available, record a
    blocker rather than replacing independent review with self-review. **In team mode, an owner sub
    must never spawn another same-harness sub-agent to perform one of its steps, critic, or audit and then wait on it** — the
    waiting is the bug. A bounded read-only search sub is allowed; never block a step on a spawned
    writer.

### Step Chain

**One task = one chain.** In solo mode the executing actor runs the selected chain top to bottom in the
foreground and waits for the independent critic/audit steps where required. In team mode each owner runs
the fast chain only through validate-2/live evidence, then hands that evidence to the orchestrator for audit.
Each step lands its document
before the next starts; documents live at `docs/<task-slug>-<step>.md`.

| # | Step | Executor | Required deliverable |
| --- | --- | --- | --- |
| 1 | **scout** | executing actor | `docs/<task>-scout.md`: exploration + plan, source classification, contract impact, external behavior, FE/BE interface, design/code sketches, testable behavior, risks and alternatives. No production code. |
| 2 | **validate-1** | executing actor | `docs/<task>-test.md` § Test Contract plus failing pre-production test scaffolds: names, intent, coverage, assertion bodies TODO. |
| 3 | **critic** | the task's fresh independent reviewer agent | `docs/<task>-critic.md`: combined review of scout + scaffold; verdict and BLOCKER / CONCERN-MR / NIT findings. Failure routes back to scout. **Full depth only.** |
| 4 | **build** | executing actor | Production code and `docs/<task>-build.md`: per-file change + reason and disposition of every BLOCKER/CONCERN-MR (full depth). |
| 5 | **validate-2** | executing actor | Complete assertions and edge cases; full `src` + `web` vitest, real `next build`, dev live verification, and `docs/<task>-test.md` § Results with defects/residuals. |
| 6 | **audit** | solo: the same reviewer agent as the critic; team: orchestrator after owner handoff | `docs/<task>-audit.md`: final review of the landed change, run **after the task's live acceptance results exist** (see POST-LIVE audit below). |

**Depth is chosen by risk:**

- **full:** all six steps. Mandatory for load-bearing security boundaries, schema migrations,
  operator-validation boundaries, and structural/contract convergence.
- **fast (operator 2026-08-11; team ownership clarified 2026-08-28):** In solo mode,
  `scout → build → validate-2 → audit`. It drops the initial scaffold
  (validate-1) **and the pre-build critic**; the executing actor writes mock tests during build. The audit
  is never dropped — it is the one independent check solo fast retains. In team mode, the owner chain is
  `scout → build → validate-2/live evidence → handoff`; critic/audit are outside owner scope and the
  orchestrator audits the handed-off evidence. Escalate out of team mode to a full-depth solo task if risk grows.
- **POST-LIVE audit (operator 2026-08-07; team ownership clarified 2026-08-28):** any task with live
  acceptance gets its audit **after the live results exist**, against the task's full document set + test results +
  live evidence together — not just a process/design review, but a consistency review of live output vs
  the task's claims (docs + carriers). It specifically hunts "signal-context" errors (e.g. a 5-axis part
  emitting a sheet-metal bending signal into DFM — a signal must be interpreted in its process/material
  context; emitting it bare is a defect). Fast depth does not escalate to full because of this, but the
  audit step itself is not exempt. In solo mode the audit is independent reviewer work; in team mode the
  owner stops at evidence handoff and the orchestrator performs the audit.

**FE/BE:** the executing actor handles both. Its scout document keeps FE and BE sections separable and fixes
the interface contract before build.

**FE visual gate:** the orchestrator defines the exact DEV visual script and serially drives the
**`agent-browser`** CLI. It records the commands, interactions,
screenshots, console errors, page errors, and observed result. Evidence from DOM-only tests, raw HTTP, or a
non-designated tool cannot replace this required run. No qualifying DEV visual evidence means rework, not
pass. In team mode the owner sub delivers a *visual verification script* (which part, which URL, what
to expect) and the orchestrator drives the browser and signs off with the evidence.

### Independent Review Delegation

- **Who reviews in solo mode.** Critic (step 3, full depth) and audit (step 6, both depths) are run by **the same
  fresh independent external reviewer agent** — one agent per task serves both review steps, whether
  spawned as a native sub-agent (Codex CLI) or obtained via the **`/codex:rescue`** route (HR12) —
  independent from the actor that wrote the plan and the code. This delegation section does not apply to
  team-mode owners; the orchestrator audits their handed-off live evidence directly.
- **Who drives them.** In solo mode the orchestrator spawns and waits on reviewers directly. Team-mode
  owners do not drive critic/audit reviewers; they retain and hand complete live evidence to the
  orchestrator, which performs the task's audit and acceptance from the main loop.
- **Invocation route (operator 2026-08-11; route updated 2026-08-13).** Codex CLI harnesses spawn
  reviewers with the native sub-agent mechanism. All other harnesses (e.g. pi) obtain the independent
  reviewer via **`/codex:rescue`**. Usage:
  - First review round (critic, or the audit when no critic thread exists):
    `/codex:rescue --write --wait "<review brief>"` — the brief covers the exact worktree path,
    scope, required reads, and the verdict document to write.
  - Follow-up rounds (audit after critic, or any revision round) continue the same reviewer thread:
    `/codex:rescue --write --wait --resume "<follow-up brief>"` — never `--fresh`; the same reviewer
    thread serves critic and audit and all revision rounds, with no round limit.
  - Track with `/codex:status` and fetch finished output with `/codex:result <job-id>`.
- Every review brief states role, task, step, exact repository/absolute worktree path, scope, required
  reads, allowed review-document write, verification, and expected deliverable.
- Because agents may share the filesystem and a drifting default cwd, every participant explicitly sets the
  assigned worktree on command calls and respects its write fence. A reviewer may write only its designated
  critic or audit document; it may not modify production code, tests, roadmap state, or any other file.
- The spawner waits for the reviewer before acting on the reviewed step. Use direct waits with useful
  durations rather than short polling, and end completed reviewers promptly.
- A missing review mechanism or slot is a recorded blocker, not permission to skip independence.
- **Commit a review verdict the INSTANT it lands — before reading it.** The reviewer writes the file; the
  executing actor is responsible for committing it; a stall leaves the only copy in the working tree.
- **🔴 A verdict can be modified AFTER it first appears.** Committing the instant it appears is not
  sufficient: once the reviewer exits, re-check `git status`, and if the artifact moved, restore the
  committed bytes and record the change in your own document. Verify with
  `git diff <verdict-commit> HEAD -- <artifact>` → must be empty.
- **🔴 CUSTODY (delegated solo-mode reviews): land review output VERBATIM, commit it before reading it, and NEVER TOUCH IT AGAIN
  (operator 2026-07-26).** A critic/audit artifact is never edited, for any reason, including editorial
  ones, and regardless of who authored the bytes. A delegated review artifact's entire value is that it was written by something that
  is neither the builder nor the orchestrator — and provenance is not checkable by a reader, so "the
  reviewer wrote this correction itself" is not a defence. Disagreeing with a verdict is legitimate and
  expected: say so loudly, in your own document. A wrong verdict is not fixed by editing it; it is fixed by
  a later verdict that overrules it and says so, with the mistaken one left standing in its immutable
  artifact. Any post-verdict reviewer tail lands in its own commit, never folded into a feature commit.
- **Cleanup is scoped to your own reviewer.** End only the agents you spawned; with several reviews in
  flight, a blanket process sweep kills a sibling's running critic.

### Orchestrator Discipline

- Read every landed document end-to-end before transition, at minimum every BLOCKER, CONCERN-MR, verdict,
  and routing section. The next brief quotes at least one specific finding or rationale as evidence of the
  read.
- Verify that document paths, commits, files, and measured values exist and reproduce. Do not accept round
  numbers or review assertions without evidence.
- Resolve conflicts deliberately; run full `src` + `web` vitest and a real `next build` at the required
  gate. A green build is not a green test suite.
- Perform the FE browser gate serially in the orchestrator loop using the designated real-browser tool and
  retain its command, screenshot, console-error, page-error, and observation evidence.
- Preserve unrelated and unknown work; never reset or stash it. Remove any collected review worktree and
  end critic/audit agents promptly.
- **Team mode only:** after verifying full acceptance, approve and merge in dependency order with the full
  gate between merges; rescue a stalled
  owner sub by committing its WIP before anything else (`git reset --hard` is unrecoverable), then nudge
  it with a specific instruction or take the work over; remove merged worktrees (`git worktree remove
  --force`).

### Auto Mode

Auto mode runs in one of the two execution modes, named by the operator's instruction: **`auto
solo mode`** or **`auto team mode`**. There is no default — if the instruction does not name
the mode, stop and ask before doing any work.

On either: re-read this file and `ROADMAP.md`, including after interruption or compaction, then:

1. **Decompose into a task list** in `ROADMAP.md`. Each task records title, scope, FE/BE classification,
   full/fast depth, status, current step, affected paths, and dependencies.
2. **Execute per the named mode.**
   - *Solo mode:* the orchestrator advances every non-review step of every task itself, in
     dependency order. It does not assign tasks, branches, worktrees, or write ranges to implementation
     sub-agents. Tasks run serially.
   - *Team mode:* dispatch one owner sub-agent per task (own isolation worktree). Each owner runs
     its whole task at fast depth in the foreground through validation and live evidence, does not run or
     delegate critic/audit, retains that evidence, and hands it to the orchestrator for audit and acceptance.
     It makes its own design decisions autonomously. Independent tasks may run in parallel only where
     write ranges are disjoint — and never two tasks touching the same area (Execution Modes ⚠️).
3. **Choose depth by risk.** Team mode is fast-only. Never use it for Hard Rule 8 security, schema migration,
   Hard Rule 9 operator-validation, or structural/contract convergence; route those tasks to full-depth solo
   mode. Escalate out of team mode when new risk appears.
4. **Do not delegate individual non-review steps.** Auto team mode assigns a whole fast-depth task to its one
   owner; that owner does not delegate critic/audit and hands live evidence to the orchestrator for audit.

At the scout direction gate, auto mode chooses the route without pausing. It still stops for operator
validation boundaries and any security exception requiring approval. Production push always requires a
new explicit instruction for the current change. If blocked, record the smallest blocker phase. Before the
session fills or ends, write a Session Handoff and tell the operator.
