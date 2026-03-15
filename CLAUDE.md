# Multi-Agent Software Development System

This project uses a team of specialized AI subagents to collaboratively build software — spawned on-demand based on task complexity, not all-at-once.

## Agent Catalog

| Agent | Role | When to use |
|-------|------|-------------|
| `architect-designer` | Creates software designs from requirements | COMPLEX/EPIC tasks only |
| `architect-reviewer` | Reviews and critiques designs | COMPLEX/EPIC tasks only |
| `developer-coder` | Writes production code from designs | All tasks |
| `developer-reviewer` | Reviews code for quality and bugs | SIMPLE and above |
| `tester` | Tests code by actually running it | MODERATE and above |
| `documenter` | Creates project documentation | EPIC tasks only |

## FULL_TASKS.md — Persistent Task Tracker (Mini Jira)

### Session Start Protocol (MANDATORY)

**Every session MUST begin by checking for `FULL_TASKS.md` in the project root.**

1. **If `FULL_TASKS.md` exists**: Read it, display a compact board summary, and ask the user what to work on.
2. **If `FULL_TASKS.md` does NOT exist**: After understanding the user's request, create it.

### Format

```markdown
# Project Task Board
> Last updated: {date} | Session: {n}

## Legend
- `TODO` — Not started
- `IN_PROGRESS` — Currently being worked on
- `IN_REVIEW` — Code written, under review
- `TESTING` — Under testing
- `DONE` — Completed and verified
- `BLOCKED` — Waiting on something

## Tasks

### TASK-001: {Title}
- **Status**: {status}
- **Priority**: P0 | P1 | P2 | P3
- **Complexity**: TRIVIAL | SIMPLE | MODERATE | COMPLEX | EPIC
- **Agents Needed**: {determined by complexity}
- **Assigned To**: {agent or unassigned}
- **Blocked By**: {TASK-XXX or none}
- **Description**: {what needs to be done}
- **Acceptance Criteria**:
  - [ ] {criterion}
- **Notes**: {context}
- **History**:
  - {date}: {event}
```

### Lifecycle Rules
- Break user work into discrete TASK-XXX items
- Update status before and after each phase
- Add History entry on every change
- Mark acceptance criteria as checked on completion
- FULL_TASKS.md persists across sessions

## Dynamic Agent Selection

**Only spawn agents the task actually needs.**

| Complexity | Agents | Use When |
|-----------|--------|----------|
| **TRIVIAL** | `developer-coder` | Typo, config, single-line fix |
| **SIMPLE** | `developer-coder` + `developer-reviewer` | Small bug fix, 1-2 file change |
| **MODERATE** | `developer-coder` + `developer-reviewer` + `tester` | New endpoint, enhancement, 2-5 files |
| **COMPLEX** | `architect-designer` + `architect-reviewer` + `developer-coder` + `developer-reviewer` + `tester` | New feature/module, 5+ files |
| **EPIC** | All 6 agents (+ parallel if applicable) | New project, major system change |

### Complexity Analysis (before spawning)
1. How many files created/modified?
2. Design decisions needed?
3. Existing code understanding required?
4. Integration points to test?
5. Documentation needed?

## Workforce Scaling — Hire and Fire

**Scale agent count based on workload. More independent tasks = more parallel agents.**

### Scaling Rules
- **N independent tasks → up to N coders** (capped at 6)
- **Scale reviewers**: 1 reviewer per 2-3 coders
- **Scale testers**: 1 tester per independent stream
- **Architects stay lean**: 1 designer + 1 reviewer (design before scaling)
- **Fire when done**: Shut down agents once their tasks are DONE

### Naming: `coder-auth`, `coder-payments`, `reviewer-backend`, `tester-api` (by domain)

### Scaling Table
| TODO Tasks | Coders | Reviewers | Testers |
|-----------|--------|-----------|---------|
| 1 | 1 | 1 | 1 |
| 2-3 | 2-3 | 1-2 | 1-2 |
| 4-6 | 4-6 | 2-3 | 2-3 |
| 7+ | Cap 6 | Cap 3 | Cap 3 |

## Task Affinity — Same Developer, Related Tasks

**Assign related tasks to the same agent. Resume > fresh spawn for domain continuity.**

### Rules
1. **Same module → same agent**: All auth tasks → `coder-auth`
2. **Bug fix for code agent wrote → resume that agent**
3. **Reviewer who reviewed → same reviewer for re-review**
4. **Tester who tested → same tester for regression**

### Agent Registry (tracked in FULL_TASKS.md)
```markdown
## Agent Registry
| Agent ID | Name | Domain | Tasks |
|----------|------|--------|-------|
| {id} | coder-auth | Auth, sessions | TASK-001, TASK-005 |
| {id} | coder-payments | Billing | TASK-002, TASK-006 |
```

### Task Grouping Flow
1. Group TODO tasks by domain (files they touch)
2. Assign each group to one agent (affinity)
3. Order tasks within group by dependency
4. Spawn agents per group — groups run in PARALLEL, tasks within group run SEQUENTIAL

### Announce Before Starting (MANDATORY)

Single task:
```
📋 Task: TASK-XXX — {title}
Complexity: {level}
Agents: {list}
Workflow: {phases}
```

Multi-task batch:
```
📋 Batch: {N} tasks across {M} domains
Affinity Groups:
  - Auth (coder-auth): TASK-001, TASK-005
  - Payments (coder-payments): TASK-002, TASK-006
Workflow: Design → Parallel Code → Parallel Review → Parallel Test
```

## Workflows by Complexity

### TRIVIAL
1. `developer-coder` → change → FULL_TASKS.md → DONE

### SIMPLE
1. `developer-coder` → code
2. `developer-reviewer` → CODE_REVIEW.md
3. If NEEDS_REVISION: resume coder (max 3)
4. FULL_TASKS.md → DONE

### MODERATE
1. `developer-coder` → code
2. `developer-reviewer` → CODE_REVIEW.md
3. If NEEDS_REVISION: resume coder (max 3)
4. `tester` → TEST_REPORT.md
5. If FAILED: resume coder + re-test (max 2)
6. FULL_TASKS.md → DONE

### COMPLEX
1. `architect-designer` → DESIGN.md
2. `architect-reviewer` → DESIGN_REVIEW.md
3. If NEEDS_REVISION: resume designer (max 3)
4. `developer-coder` → implement
5. `developer-reviewer` → CODE_REVIEW.md
6. If NEEDS_REVISION: resume coder (max 3)
7. `tester` → TEST_REPORT.md
8. If FAILED: resume coder + re-test (max 2)
9. FULL_TASKS.md → DONE

### EPIC
Same as COMPLEX, plus:
- `documenter` at end
- Check DESIGN.md for parallel streams
- If 2+ independent modules with no shared files → parallel coding/review
- Use TeamCreate for parallel, TeamDelete to cleanup

## Execution Mode: Sequential vs Parallel

**Default to sequential. Only use parallel for EPIC tasks when DESIGN.md identifies independent streams.**

### Parallel Rules
- NEVER let two agents write to the same file
- Each stream has its own file list
- Shared files → assigned to ONE stream
- Same file needed by both → sequential for that file

## Session Management (Agent Resume)

| Scenario | Action |
|----------|--------|
| Revision loop (same agent, same task) | **Resume** — feedback delta only |
| Related task, same domain (affinity) | **Resume** — agent knows the domain, just describe new task |
| Unrelated new task | **Fresh** — different context |
| Different agent, same workflow | **Fresh** — pass artifact paths |

## Shared Artifacts
```
FULL_TASKS.md      ← team lead maintains
DESIGN.md          ← architect-designer writes
DESIGN_REVIEW.md   ← architect-reviewer writes
CODE_REVIEW.md     ← developer-reviewer writes
TEST_REPORT.md     ← tester writes
FEATURE_DOCS.md    ← documenter writes
```

## Rules
- Delegate to agents via Agent tool — don't do their work
- Read output files after each agent to check results
- **Update FULL_TASKS.md** on every status change
- Max iterations: 3 for design/code, 2 for testing
- **Resume** for revision loops, **fresh** for new tasks
- Spawn only agents the task complexity requires
- **Scale agents with workload** — N independent tasks = N parallel coders
- **Use task affinity** — related tasks → same agent, resume for domain continuity
- **Fire idle agents** — shut down when domain tasks are DONE
- **Track Agent Registry** in FULL_TASKS.md for resume mapping
- Tester MUST run code — ask user to start server
- Reviewer MUST trace values across files
