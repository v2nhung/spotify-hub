---
name: using-agent-skills
description: Discovers and invokes agent skills. Use when starting a session or when you need to discover which skill applies to the current task. This is the meta-skill that governs how all other skills are discovered and invoked.
---

# Using Agent Skills

## Overview

Agent Skills is a collection of engineering workflow skills organized by development phase. Each skill encodes a specific process that senior engineers follow. This meta-skill helps you discover and apply the right skill for your current task within the Spotify microservices ecosystem.

## Available Skills in This Workspace

Only invoke skills that exist inside `spotify-hub/skills/`:
- `interview-me`: Surface what the user actually wants before any plan, spec, or code exists.
- `planning-and-task-breakdown`: Decompose requirements into small, verifiable tasks (with SOLID check + GOF recommendations).
- `incremental-implementation`: Build thin vertical slices, verifying each slice before expanding.
- `code-review-and-quality`: Review plans and code across quality, security, and maintainability gates.

## Project Architecture Context (Read First)

Before planning tasks (`planning-and-task-breakdown`), implementing code (`incremental-implementation`), or reviewing code (`code-review-and-quality`), check for architecture and convention documents inside `spotify-hub/` (e.g., `AGENTS.md`, `architechture/`, `convention/`) if they exist:

- **Architecture Blueprint**: microservices topology (`api-gateway`, `spotify-media`, `spotify-notification`, etc.), layering rules, security/Keycloak auth flows, communication patterns, and extension rules.
- **Folders Structure Blueprint**: module layout, package placement, naming conventions, file-location rules, and folder templates.

Planning design rule (mandatory):
- During planning, run a SOLID check on proposed tasks and boundaries (SRP, OCP, LSP, ISP, DIP).
- Recommend GOF patterns only when they reduce coupling, clarify extension points, or remove branching complexity.
- If no pattern is justified, explicitly choose the simpler direct design.
- For each non-trivial plan, include: SOLID impact notes, GOF candidate(s), expected benefits, and trade-offs.

Decision rule before every plan/implementation/review:
- If architecture documents are aligned, follow them.
- If they conflict, prefer `AGENTS.md` for mandatory guardrails, then reconcile/update the blueprints.
- If the area is not covered, fall back to `AGENTS.md` and direct code exploration across the target module.

Maintenance rule:
- If a change alters architecture or structure (new service, shared module, package pattern, cross-cutting concern, folder convention), update or regenerate the relevant blueprint(s) before continuing.

## Skill Discovery

When a task arrives, identify the development phase and apply the corresponding skill:

```
Task arrives
    │
    ├── Don't know what you want yet / ambiguous scope? ──→ interview-me
    ├── Have a feature/requirement, need tasks? ──────────→ planning-and-task-breakdown (include SOLID check + GOF recommendations)
    ├── Have tasks, need plan review? ────────────────────→ code-review-and-quality (plan review)
    ├── Implementing code? ───────────────────────────────→ incremental-implementation
    └── Reviewing code before merge? ─────────────────────→ code-review-and-quality
```

## Core Operating Behaviors

These behaviors apply at all times, across all skills. They are non-negotiable.

### Comment Style When Implementing Features

Do not generate verbose Javadoc boilerplate on top of standard endpoints or straightforward methods:

- **Don't:**
```java
/**
 * Sync media playback metadata to update user listening history.
 * Called when track playback finishes and needs to notify notification-service.
 * This is an internal sync endpoint and should not be called by external clients.
 *
 * @param request The playback sync request containing track ID and user ID
 * @return ResponseEntity with result indicating success/failure
 */
@PostMapping("/sync-playback")
public ResponseEntity<ResultJson> syncPlaybackHistory(
        @RequestBody TrackPlaybackRequest request) {
    return setResponseEntity(playbackService.syncPlaybackHistory(request));
}
```

- **Should Do:**
```java
@PostMapping("/sync-playback")
public ResponseEntity<ResultJson> syncPlaybackHistory(
        @RequestBody TrackPlaybackRequest request) {
    return setResponseEntity(playbackService.syncPlaybackHistory(request));
}
```
*(Only add concise inline comments for non-obvious business rules or edge cases).*

### 1. Surface Assumptions

Before implementing anything non-trivial, explicitly state your assumptions:

```
ASSUMPTIONS I'M MAKING:
1. [assumption about requirements]
2. [assumption about architecture]
3. [assumption about scope]
→ Correct me now or I'll proceed with these.
```

Don't silently fill in ambiguous requirements. The most common failure mode is making wrong assumptions and running with them unchecked. Surface uncertainty early — it's cheaper than rework.

### 2. Manage Confusion Actively

When you encounter inconsistencies, conflicting requirements, or unclear specifications:

1. **STOP.** Do not proceed with a guess.
2. Name the specific confusion.
3. Present the tradeoff or ask the clarifying question.
4. Wait for resolution before continuing.

**Bad:** Silently picking one interpretation and hoping it's right.
**Good:** "I see X in the requirement but Y in the existing service. Which takes precedence?"

### 3. Push Back When Warranted

You are not a yes-machine. When an approach has clear problems:

- Point out the issue directly.
- Explain the concrete downside (quantify when possible — "this adds ~200ms latency" not "this might be slower").
- Propose an alternative.
- Accept the human's decision if they override with full information.

Sycophancy is a failure mode. "Of course!" followed by implementing a bad idea helps no one. Honest technical disagreement is more valuable than false agreement.

### 4. Enforce Simplicity

Your natural tendency is to overcomplicate. Actively resist it.

Before finishing any implementation, ask:
- Can this be done in fewer lines?
- Are these abstractions earning their complexity?
- Would a staff engineer look at this and say "why didn't you just..."?

If you build 1000 lines and 100 would suffice, you have failed. Prefer the boring, obvious solution. Cleverness is expensive.

### 5. Maintain Scope Discipline

Touch only what you're asked to touch.

Do NOT:
- Remove comments you don't understand
- "Clean up" code orthogonal to the task
- Refactor adjacent services/modules as a side effect
- Delete code that seems unused without explicit approval
- Add features not in the requirement because they "seem useful"

Your job is surgical precision, not unsolicited renovation.

### 6. Verify, Don't Assume

Every skill includes a verification step. A task is not complete until verification passes. "Seems right" is never sufficient — there must be evidence (passing tests, build output, runtime data).

- Per-skill verification is the local check.
- The project-wide bar that applies to every change, regardless of which skill is active, is the Definition of Done: tests pass, no regressions, behavior verified at runtime, docs updated.

## Failure Modes to Avoid

These are the subtle errors that look like productivity but create problems:

1. Making wrong assumptions without checking
2. Not managing your own confusion — plowing ahead when lost
3. Not surfacing inconsistencies you notice
4. Not presenting tradeoffs on non-obvious decisions
5. Being sycophantic ("Of course!") to approaches with clear problems
6. Overcomplicating code and APIs
7. Modifying code or comments orthogonal to the task
8. Removing things you don't fully understand
9. Building without a clear plan/spec because "it's obvious"
10. Skipping verification because "it looks right"

## Skill Rules

1. **Check for an applicable skill before starting work.** Skills encode processes that prevent common mistakes.
2. **Skills are workflows, not suggestions.** Follow the steps in order. Don't skip verification steps.
3. **Multiple skills can apply in sequence.** A feature implementation typically follows: `interview-me` → `planning-and-task-breakdown` → `code-review-and-quality` (plan review) → `incremental-implementation` (one slice at a time) → `code-review-and-quality` (final code review).
4. **When in doubt, clarify first.** If the task is non-trivial and requirements are unclear, begin with `interview-me` before `planning-and-task-breakdown`.

## Planning Design Heuristics (SOLID + GOF)

When producing or reviewing a plan, apply this gate before implementation starts:

1. **SOLID boundary check:**
    - **SRP:** each task/component should have one reason to change.
    - **OCP:** plan extension through new classes/strategies over editing stable branches.
    - **LSP/ISP:** keep interfaces behaviorally compatible and minimal.
    - **DIP:** depend on abstractions at service boundaries and integration seams.

2. **GOF recommendation pass:**
    - Recommend pattern use only where complexity already exists or is highly likely.
    - Prefer these defaults in this codebase when justified:
        - **Strategy** for interchangeable business rules.
        - **Factory Method / Abstract Factory** for creation logic with variants.
        - **Template Method** for fixed workflow with variable steps.
        - **Observer (Spring events)** for post-action side effects.
        - **Adapter** for external API contract mismatch.
        - **State** for explicit status transition behavior.

3. **Simplicity guard:**
    - Do not force patterns into simple CRUD or one-path logic.
    - Include a short "why not" note for rejected patterns when the decision is non-obvious.

4. **Planning output requirement:**
    - Add a "Design Notes" block in planning artifacts with:
        - SOLID findings
        - Recommended GOF pattern(s) and location
        - Simpler alternative considered
        - Migration/refactor trigger if pattern is deferred

## Plan File Management

When creating plans (via `planning-and-task-breakdown`), save the resulting plan document following this convention:

### File Naming Convention
- **Format:** `SPOTIFY-${camelCaseName}.md`
- **Example:** `SPOTIFY-mediaPlaylistStreaming.md`
- No frontmatter (pure markdown content)

### Directory Structure
Save plan files to the designated plans directory:
- **Location:** `spotify-hub/requirement/plans/`
- **Alternative:** If project root level is preferred, save to `${projectRoot}/plans/`

### When to Create Plan Files
Create a plan file when:
1. User requests `planning-and-task-breakdown`
2. User explicitly asks to save a plan
3. A feature/ticket requires multi-step implementation across services
4. The task complexity warrants documented breakdown

### Plan File Template
Each plan should include:
- **Title:** Clear feature/ticket name
- **Objective:** What is being built/changed
- **Context:** Relevant background, affected services (`api-gateway`, `spotify-media`, `spotify-notification`, etc.), existing implementation references
- **Acceptance Criteria:** From ticket/requirement
- **Design Notes:** SOLID findings, GOF pattern candidates, simpler alternatives
- **Implementation Steps:** Numbered, actionable tasks
- **Special Considerations:** Edge cases, risks, cross-service dependencies
- **Testing Checklist:** Unit, integration, manual tests
- **Definition of Done:** Completion criteria

### Example Command Flow
```
User: /skill:using-agent-skills đây là requirement cho tính năng mới, hãy lên plan giúp tôi
Agent: [Invokes planning-and-task-breakdown, creates plan with SOLID/GOF analysis and steps]
User: Save the resulting plan into a file
Agent: [Saves to spotify-hub/requirement/plans/SPOTIFY-featureName.md]
```

## Lifecycle Sequence

For a complete feature using the available skills in `spotify-hub/skills/`:

```
1. interview-me                → Clarify vague requirements & edge cases
2. planning-and-task-breakdown → Break into verifiable slices + SOLID/GOF design check
3. code-review-and-quality     → Review plan with user before writing code
4. incremental-implementation  → Build slice by slice (run tests & verify each slice)
5. code-review-and-quality     → Final code review before merge
```

Not every task needs every skill. A small change or straightforward feature can start directly at `planning-and-task-breakdown` or `incremental-implementation`.

## Quick Reference

| Phase | Skill | One-Line Summary |
| :--- | :--- | :--- |
| **Define** | `interview-me` | Surface what the user actually wants before any plan, spec, or code exists |
| **Plan** | `planning-and-task-breakdown` | Decompose requirements into small, verifiable tasks with SOLID/GOF check |
| **Build** | `incremental-implementation` | Thin vertical slices, verify and test each slice before expanding |
| **Review** | `code-review-and-quality` | Review plans and code across quality, security, and simplicity gates |