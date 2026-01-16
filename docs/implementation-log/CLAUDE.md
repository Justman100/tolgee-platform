# Implementation Log Conventions

This directory contains detailed implementation logs for changes to this repo. These logs serve as both documentation and a safety net for rollbacks.

---

## Directory Structure

Each significant feature or change gets its own folder:

```
implementation-log/
├── CLAUDE.md                              # This file
├── 2026-01-15-add-tracing/               # Feature folder (date + short description)
│   ├── PLAN.md                           # High-level plan
│   ├── PHASE_1.md                        # Detailed guide for Phase 1
│   ├── PHASE_1_LOG.md                    # Execution log for Phase 1
│   ├── PHASE_2.md                        # Detailed guide for Phase 2
│   ├── PHASE_2_LOG.md                    # Execution log for Phase 2
│   └── ...
└── 2026-02-01-refactor-auth/             # Another feature
    └── ...
```

**Folder naming:** `YYYY-MM-DD-short-description`

---

## PLAN.md Structure

The high-level plan provides context and breaks work into phases.

```markdown
# Feature Name

## Overview
Brief description of what we're implementing and why.

## Goals
- Goal 1
- Goal 2

## Non-Goals
- What we're explicitly not doing

## Architecture
Diagrams and explanations of the target architecture.

## Phases

### Phase 1: [Name]
- Description of deliverable
- Key decisions
- Files affected

### Phase 2: [Name]
- Description of deliverable
- Key decisions
- Files affected

### Phase N: Documentation
Design and implement documentation following the Code Documentation Principle.
- Create DOCS_PLAN.md with documentation hierarchy
- Implement documentation in code
- Verify newcomer can follow the trail

## Rollback Strategy
High-level rollback approach.

## Open Questions
Decisions that need to be made during implementation.
```

**Key principles:**
- Each phase should be a coherent deliverable (roughly one PR, but not strict)
- Phases should be ordered by dependency and risk
- Earlier phases should be lower risk / easier to rollback
- **Always include a Documentation phase** for features with non-trivial code

---

## PHASE_N.md Structure

Detailed step-by-step implementation guide with testable checkpoints.

````markdown
# Phase N: [Name]

## Overview

| Step | Description | Test Method |
|------|-------------|-------------|
| 0 | First step | How to verify |
| 1 | Second step | How to verify |

## Handling Unexpected Situations

**This is critical.** During execution, if you encounter something unexpected that requires a non-read operation not pre-approved in the plan/phase document, you MUST:

1. **STOP** - Do not proceed with the unexpected action
2. **ALERT** - Inform the user what happened and what action you believe is needed
3. **WAIT** - Let the user decide whether to proceed

**Examples of unexpected situations requiring pause:**
- A required dependency doesn't exist or has a different version than specified
- You discover the plan needs a step that wasn't anticipated
- A different approach is needed to solve a problem
- A database migration is needed that wasn't in the plan
- An unexpected error occurs that requires manual intervention
- Significant changes to parts of the codebase that are not covered by this document
- If at any point you've realized that you've significantly deviated from the plan (regardless of whether the deviation is correct and necessary or not)

**What to do:**
```
"I've encountered an unexpected situation: [describe what happened].

To proceed, I would need to [describe the action, e.g., 'add a new dependency',
'create a database migration', 'update the library version from X to Y'].

This wasn't in the original plan. Should I proceed with this action?"
```

**Why this matters:**
- Unexpected actions may have unintended consequences
- The user may have context you don't (e.g., "we're pinned to that version for a reason")
- It maintains the integrity of the plan as the source of truth
- It prevents cascading issues from well-intentioned but incorrect fixes

**Exception:** Read-only operations (checking versions, listing files, reading code) to gather information are fine without approval.

---

## Step 0: [Name]

### Goal
What this step accomplishes.

### Prerequisites
What must be done/true before this step.

### Instructions
Detailed commands, file contents, etc.

### Testing
Commands to verify the step worked.

### Checklist
- [ ] Item 1
- [ ] Item 2
- [ ] **Log entry appended to PHASE_N_LOG.md**

### Notes
```
Space for runtime notes during implementation.
```

---

## Step 1: [Name]
...

---

## Rollback
Commands to undo everything in this phase.
````

**Key principles:**
- Each step should be independently testable
- Include exact file contents and commands
- Checklists are updated during implementation (marked [x] when done)
- **Every step must include logging** - the checklist must have "Log entry appended to PHASE_N_LOG.md"
- Notes section captures decisions and observations during implementation

---

## PHASE_N_LOG.md Structure

**This is the critical safety document.** An append-only log of every action taken.

````markdown
# Phase N Implementation Log

Reference documents:
- [PLAN.md](./PLAN.md)
- [PHASE_N.md](./PHASE_N.md)

---

## Step 0: [Name]

### Context
Why we're doing this step, tied to the bigger picture.

### Step 0.1: [Sub-step]

**Command:**
```bash
exact command run
```

**Timestamp:** YYYY-MM-DDTHH:MM:SSZ

**Result:** SUCCESS/FAILURE
```
actual output (sanitized of secrets)
```

**Rollback:** `command to undo this specific action`

---

### Step 0.2: [Sub-step]
...

---

## Step 0 Summary

| Resource | Status | Rollback |
|----------|--------|----------|
| Thing 1 | CREATED | rollback command |
| Thing 2 | MODIFIED | rollback command |

---

## Step N Correction: [Description]

### Context
What went wrong and why we're correcting it.

### Correction Applied
What we changed.

---
````

### Critical Requirements for Logs

1. **Append-only**: Never modify or delete previous entries. If something was wrong, add a correction section.

2. **Rollback commands for everything**: Every action that changes state (creates files, modifies code, runs commands with side effects) must have a corresponding rollback command.

3. **Include external changes**: Database migrations, configuration changes, dependency updates - anything with external impact must be logged with rollback instructions.

4. **Sufficient detail for disaster recovery**: The log must contain enough information to fully rollback even if:
   - The conversation is compacted or cleared
   - The session is lost entirely
   - A different person/agent needs to perform the rollback

5. **No secrets in logs**: Sanitize outputs. Use placeholders like `<encrypted>` or `<redacted>`.

6. **Timestamps**: Include timestamps for actions that have external effects (e.g., database migrations).

7. **Context**: Each step should explain *why* it's being done, not just *what*.

---

## Workflow

1. **Before starting**: Read PLAN.md and PHASE_N.md to understand the work
2. **During implementation**:
   - Follow PHASE_N.md step by step
   - Log every action to PHASE_N_LOG.md immediately
   - Update checklists in PHASE_N.md as steps complete
3. **If something goes wrong**:
   - Do NOT modify previous log entries
   - Add a correction section explaining what happened and what was fixed
4. **After completing a phase**: Verify all checklists are complete, all logs are written

---

## Logging Discipline

**The log is not optional.** It is part of the deliverable. A phase is not complete without its log.

### Common Failure Modes

1. **Task tunnel vision**: Getting focused on "making it work" and forgetting to document
2. **Problem-solving mode**: When errors occur, jumping to fix them without logging
3. **Implicit completion**: Marking a task "done" when the code works, but the log is missing

### Prevention Strategies

1. **Include logging in the todo list explicitly**:
   ```
   - [ ] Create PHASE_N_LOG.md
   - [ ] Implement step 0 (logging as I go)
   - [ ] Implement step 1 (logging as I go)
   - [ ] Verify all log entries complete
   ```

2. **Log BEFORE executing**: The discipline should be:
   - Write what you're about to do in the log
   - Execute the action
   - Record the result
   - NOT: do things and hope to document later

3. **Treat the log as a deliverable**: The definition of "done" includes:
   - Code/config changes complete
   - Tests passing
   - **Log file complete with all actions and rollback commands**

4. **Create the log file first**: Before any implementation, create PHASE_N_LOG.md with the header and reference links. This makes it visible and harder to forget.

### Recovery

If you realize you forgot to log:

1. **Stop immediately** - don't continue without logging
2. **Create the log retroactively** - reconstruct what was done as accurately as possible
3. **Mark it as retroactive** - add a note that the log was created after the fact
4. **Continue with proper logging** - log all subsequent actions in real-time

---

## Example: Correcting a Mistake

Wrong approach (modifying history):
```markdown
## Step 2.3: Add OpenTelemetry dependency
**Version:** 1.35.0  # <-- editing previous entry
```

Correct approach (append correction):
```markdown
## Step 2.3: Add OpenTelemetry dependency
**Version:** 1.32.0

---

## Step 2 Correction: Update Dependency Version

### Context
The version 1.32.0 was taken from outdated plan documentation.
Current stable version is 1.35.0.

### Correction Applied
Updated opentelemetry-bom from 1.32.0 to 1.35.0 in settings.gradle.
Ran ./gradlew dependencies --refresh-dependencies successfully.
```

---

## Why This Matters

Code changes can have cascading effects. A detailed, append-only log ensures:

1. **Transparency**: Anyone can see exactly what was done
2. **Reversibility**: Any change can be undone
3. **Continuity**: Work can continue across sessions without loss of context
4. **Debugging**: When something breaks, the log shows what changed
5. **Learning**: Mistakes are documented, not hidden

---

## Documentation as a Separate Phase

**Reference:** See [Code Documentation Principle](../../CLAUDE.md#code-documentation-principle) for the documentation standards.

When implementing features that introduce new concepts, integrations, or non-trivial code, documentation should be planned and executed as a **separate phase** in the implementation plan.

### Why a Separate Phase?

1. **Prevents documentation being an afterthought**: Explicit phase ensures it's not skipped
2. **Allows proper planning**: Documentation hierarchy can be designed before writing
3. **Better quality**: Focused attention on docs rather than squeezing it in with code
4. **Reviewable**: Documentation can be reviewed independently

### Documentation Phase Structure

Create a `DOCS_PLAN.md` file in the implementation-log folder that maps out:

```markdown
# Documentation Plan

## Overview
What concepts need to be documented and why.

## Documentation Hierarchy

### Entry Point
Where should a newcomer start? What file/comment provides the overview?

### Components to Document
| Component | Location | What to Document |
|-----------|----------|------------------|
| Main config class | `path/to/File.kt` | High-level overview, links to details |
| Constants | `path/to/File.kt` | What each constant means, links to external docs |
| ... | ... | ... |

### External Concepts
| Concept | 1-2 Sentence Summary | Link |
|---------|---------------------|------|
| OpenTelemetry | Observability framework for traces/metrics/logs | https://opentelemetry.io/docs/ |
| ... | ... | ... |

## Verification
How to verify the documentation is sufficient:
- [ ] Newcomer can understand the feature by following the hierarchy
- [ ] All magic strings/constants are explained
- [ ] External concepts are linked with summaries
```

### Process

1. **After code implementation**: Create `DOCS_PLAN.md` in the implementation-log folder
2. **Map out the hierarchy**: Design where readers should start and how they navigate
3. **Get approval**: Review the documentation plan before implementing
4. **Implement**: Add documentation to the code following the plan
5. **Verify**: Check that the hierarchy works for a newcomer
