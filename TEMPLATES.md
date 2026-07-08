# TEMPLATES.md

Copy-paste skeletons for `ROADMAP.md` and the per-phase docs. Referenced from `CLAUDE.md` → § Roadmap. Keep entries terse; fill the brackets, delete what you don't use.

## `ROADMAP.md`

```markdown
# ROADMAP

## Version sequence
- v0.1 → v0.2 → …   (operator-validation gates — where auto-mode must halt: <…>)

## Phase <N> — <title>   [status: In Progress | Blocked | Done]
- **Goal:** <one line>
- **Scope:** <paths / components in play>
- **Run mode:** full | fast   •   **Difficulty → model:** sonnet | opus (<why opus, if so>)
- **FE / BE:** FE → <items>  |  BE → <items>   •   interface contract: <shape>
- **Gates:** <last gate passed> → <next gate>
- **Live verification target:** <real system + how it's exercised>
- **Blockers:** none | → <blocker phase link>
- **Next action:** <owner> — <what>

## Decisions
- <YYYY-MM-DD> — <operator decision> — rationale: <one line> — orchestrator: agreed | pushed back → <outcome>

## Session Handoff
- **Phase / gate:** <N / Step-or-FM> (<mode>)
- **Last landed:** <deliverable>   •   **Next action:** <exact next step>
- **In-flight:** <owner → worktree/branch → state>
- **Open blockers:** <…>
- **Resume by:** dispatch <actor>, read <doc> first

## Incomplete / Deferred
- <item> — <why deferred> — <link>
```

## Phase docs (`docs/phase-<N>-*.md`)

Each gate writes its own doc under fixed names so the next actor can find them.

- **`docs/phase-<N>-plan.md`** — architect (full) / orchestrator (FM-0)
  ```markdown
  # Phase <N> Plan
  ## §5 Testable behaviors
  - T-<Area>.<n>: given <…> when <…> then <…>   (pre/post: <…>)
  ## §6 Code sketches
  - <file / function> — <signature or shape>
  ## FE / BE
  - FE: <…>   •   BE: <…>   •   interface contract: <…>
  ```
- **`docs/phase-<N>-test.md`** — validator. § Test Contract (Step 2) written first; § Results (Step 5) appended after the run.
- **`docs/phase-<N>-critics.md`** — critic. Findings tagged **BLOCKER / CONCERN (CONCERN-MR) / NIT** + verdict + routing decision.
- **`docs/phase-<N>-review.md`** — audit (full mode). **ACCEPT / REJECT** + residual risks.
