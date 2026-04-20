---
name: Refactor Task
description: Handle a refactoring request with proper scoping, safety checks, and rollback discipline. Used when Fareed or Iris asks Isaac to restructure existing code - dbt models, Python modules, TypeScript components, infrastructure configs. Ensures refactors are reversible, tested, and don't silently change behavior.
---

# Refactor Task

## Purpose

Refactoring is high-leverage but high-risk work. Code that looks the same functionally can behave differently in production, and refactor regressions are particularly painful because they appear weeks after the change when nobody remembers what moved.

This skill defines the discipline I follow for any refactor — small or large. The discipline is the same; scope varies.

## When this skill triggers

Request patterns that trigger this:
- "Refactor X"
- "Clean up Y"
- "Restructure Z"
- "Move W to..."
- "Consolidate these..."
- "Split this apart..."
- "Rename..."

Requests that don't trigger this (different skills):
- "Fix this bug" — bug fix skill
- "Add a feature" — feature implementation skill
- "Rewrite this in another language" — migration skill

## Steps

### 1. Understand scope before touching anything

Before any edit:

- Read the code being refactored in full
- Identify what depends on it (imports, references, callers)
- Identify what it depends on (its dependencies)
- Understand why it's currently structured the way it is (there's often a reason)
- Confirm I understand the user's intent — is this a mechanical move, a structural change, or a behavioral fix?

If scope is ambiguous, ask before starting. Cheap now, expensive after.

### 2. Define done explicitly

Write out, for my own reference:

- What the code should look like after
- What behavior must NOT change
- What tests exist to verify behavior preservation
- What new tests might be needed if coverage is thin

Commit this to a planning doc in my working directory. Reference in commits.

### 3. Branch safely

```bash
git checkout -b refactor/<area>-<brief-description>
# Example: refactor/staging-models-prefix-consistency
```

Never refactor on main. Never refactor on an unrelated feature branch.

### 4. Baseline the existing behavior

For code with existing tests:
- Run tests on the pre-refactor code, confirm they pass
- Record test results

For code without tests:
- Write characterization tests first — tests that capture current behavior before I change anything
- These are different from "the tests we should have" — they're "what does this actually do right now"
- Then refactor with these tests as safety net

For infrastructure/config refactors:
- Deploy current config to staging, capture state
- This is the baseline that new config must match

### 5. Make the change in the smallest possible steps

Big refactors fail because they make too many changes at once. My rule:

- Commit after each atomic step that leaves tests passing
- If I'm tempted to fix "one more thing while I'm here," resist or make it a separate commit
- Each commit message describes what moved, not why it moved (why goes in PR description)

Example for renaming `staging_models` to `stg_models`:
```
commit 1: rename directory
commit 2: update imports in referring models
commit 3: update CI references to new path
commit 4: update documentation
```

Not:
```
commit 1: rename and fix various things
```

### 6. Verify at each step

After each commit:
- Tests still pass
- Code compiles / lints clean
- No functional behavior change (if that was the guarantee)

If any step breaks something, stop. Don't pile on more changes trying to fix it. Back out the last commit, rethink, try smaller.

### 7. Handle cross-agent concerns

If my refactor affects code Oracle uses (dbt models, BigQuery schemas she queries):
- Write a message to Oracle via message bus: "I'm renaming `stg_orders` to `stg_tasker__orders`, effective on merge. Your queries referring to the old name need update."
- Wait for her acknowledgment
- If she's running an active query, don't merge until she's done

If my refactor affects Iris's integrations (portal endpoints, API contracts):
- Same pattern — bus message, acknowledgment, coordinate

If my refactor affects Sentinel's monitoring (metric names, log patterns):
- Update his watchers in the same PR

Cross-agent coordination is not optional. Silent changes break other agents.

### 8. PR discipline

PR description must include:

```markdown
## What
<Brief summary of the refactor>

## Why
<Motivation — what problem this solves, what it enables>

## How
<Approach — what's moving, what's not, why this particular structure>

## Scope boundary
What's IN:
- <explicit list>
What's OUT (deliberately not touched):
- <explicit list>

## Behavior preservation
<How I verified this doesn't change runtime behavior>

## Dependencies
<Other PRs or agents affected, references to bus messages>

## Rollback plan
<How to revert if this turns out badly>
```

### 9. Merge only with approval

During probation trust level, I never merge to main without Fareed's approval. I open the PR, link to it in the portal, wait.

During supervised trust level, I can merge low-risk refactors (documentation, comments, simple renames with full test coverage) but flag it.

During autonomous trust, I can merge refactors that pass all CI gates and don't touch cross-agent interfaces.

Cross-agent interfaces always require approval regardless of trust level.

### 10. Document in memory

After merge, write to project log (`<project>_YYYY-MM-DD.md`):
- What refactor was done
- Why
- Outcome (success, partial, issues found)
- Any patterns learned that belong in shared learnings

If the refactor taught me something about the codebase structure, that becomes a structured fact in memory:
- "Staging models use stg_<source>__<entity> convention (established YYYY-MM-DD)"

## Scope patterns

### Small refactor (under 50 lines, single file)
- Full process above, but often completes in one commit
- Test verification is fast
- Usually doesn't need cross-agent coordination

### Medium refactor (single component/module, multiple files)
- Full process, with branch-per-step commits
- May need coordinated updates in caller code
- Usually bounded in time (a few hours)

### Large refactor (architecture-level, multiple components)
- REJECT THE REQUEST if scope is this large without explicit planning
- Propose breaking it into phases
- Each phase is its own PR following this skill
- Never attempt a "big bang" refactor

Big bangs fail. Small moves succeed.

## Failure modes

**Silent behavior change:** refactor looks clean, tests pass, but runtime behavior changed subtly. Mitigation: characterization tests before starting, deploy to staging before production.

**Scope creep:** "while I'm in here, let me also..." — STOP. Make a separate ticket. Current PR stays scoped.

**Lost context:** refactor changes code I don't fully understand, breaks something non-obvious. Mitigation: step 1 (understand scope) is mandatory, not skippable.

**Cross-agent break:** other agents' code breaks because I didn't coordinate. Mitigation: step 7, required.

**Untestable change:** code has no tests, I don't write characterization tests, I "trust" the refactor. This WILL break eventually. Mitigation: write characterization tests or refuse the refactor.

## Tone during refactor work

Quiet and methodical. Refactors aren't exciting; they're hygiene. I don't narrate unless something unexpected happens. I report:

- Start: "Starting refactor of <area>. Branch: <name>. ETA: <estimate>."
- Midway, only if something surprises me: "Found that <thing> depends on <X>. Adjusting approach."
- Done: "Refactor complete. PR: <link>. Ready for review."

No drama, no victory laps.

## Related skills

- `characterization-test-writing.md` — when code lacks tests
- `cross-agent-coordination.md` — the coordination pattern
- `pr-authoring.md` — PR description conventions

## Related memory

- `genesis-config/memory/shared-learnings/patterns/refactor-lessons.md` — accumulated wisdom from past refactors
- `genesis-config/memory/agent-reflections/isaac_*.md` — my own reflections on refactors that went well or badly
