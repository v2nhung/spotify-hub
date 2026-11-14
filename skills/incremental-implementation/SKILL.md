---
name: incremental-implementation
description: Delivers changes incrementally. Use when implementing any feature or change that touches more than one file. Use when you're about to write a large amount of code at once, or when a task feels too big to land in one step.
---

# Incremental Implementation — Hermes Project

## Overview

Build in thin vertical slices — implement one piece, test it, verify it, then expand. Hermes is a Maven multi-module Java 8 / Spring Boot 2.x monorepo with an Angular 12 frontend. Each increment must leave every module compilable and existing tests green. Never accumulate large uncommitted changes across modules.

## When to Use

- Implementing any change that touches more than one file
- Building a new feature from a task breakdown
- Refactoring an existing `ServiceImpl` or Angular component
- Any time you're about to write more than ~100 lines without testing

## The Increment Cycle

```
┌──────────────────────────────────────────────┐
│                                              │
│   Implement ──→ Compile ──→ Test ──→ Verify  │
│       ▲                              │       │
│       └────── Commit ◄───────────────┘       │
│               │                             │
│               ▼                             │
│           Next slice                        │
│                                              │
└──────────────────────────────────────────────┘
```

For each slice:

1. **Implement** the smallest complete piece of functionality
2. **Compile** — `mvn -T1C -q -DskipTests clean compile` (all modules must compile)
3. **Test** — `mvn test -pl <module>` (run the affected module's tests)
4. **Verify** — confirm behavior matches acceptance criteria
5. **Commit** — descriptive message following `git-workflow-and-versioning` guidance
6. **Next slice** — carry forward

## Hermes Slicing Strategy

### Vertical Slices (Preferred)

For a new feature in `execution-service`:

```
Slice 1: Entity + Repository (with siteId filter)
    → Compile passes, no tests broken

Slice 2: Service interface + ServiceImpl (happy path)
    → Unit tests green: mvn test -pl execution-service

Slice 3: Service failure paths + unit tests
    → All service tests green

Slice 4: Controller + DTO (@Valid, @PreAuthorize)
    → Compile passes, Postman smoke test passes

Slice 5: Angular TypeScript model + service (HttpClient)
    → ng build succeeds

Slice 6: Angular component (PrimeNG table or form)
    → End-to-end flow works in local environment
```

### Contract-First Slicing (Backend + Frontend in Parallel)

When the Angular team needs to work while the backend is still in progress:

```
Slice 0: Define REST contract (DTO shape, endpoint URL, error format)
Slice 1a: Backend implements against the contract + unit tests
Slice 1b: Angular mock service matching the contract + component tests
Slice 2: Integration — point Angular at real backend, verify end-to-end
```

### Risk-First Slicing

Tackle the highest-risk piece first:

```
Slice 1: Prove the new Feign client call works (highest risk: external dependency)
Slice 2: Add business logic on top of the proven Feign call
Slice 3: Add error handling and fallbacks
```

## Implementation Rules

### Rule 0: Simplicity First

Before writing any code, ask: "What is the simplest thing that could work?"

In Hermes this usually means:
- Do not create a new `AbstractXxxService` if one service suffices
- Do not build a generic configuration mechanism for a single use case

```
SIMPLICITY CHECK:
✗ New abstract base class with generic type parameters for one subclass
✓ Concrete ServiceImpl with a private helper method

✗ New Spring @Configuration + strategy beans for one variation
✓ A private method with a clear comment

✗ New shared module class for code used in only one module
✓ Package-private class in the consuming module
```

### Rule 0.5: Scope Discipline

Touch only what the task requires. Do NOT:
- Clean up adjacent code while implementing something else
- Refactor imports in files you are only reading
- Add Angular features not in the current task
- Change Liquibase scripts that are already merged

If you notice something worth improving, note it explicitly:

```
NOTICED BUT NOT TOUCHING:
- OperationExecutionServiceImpl has a raw RuntimeException at line 2345 (separate task)
- Angular WorkstationComponent has an unused `@Input` (separate task)
→ Want me to create tasks for these?
```

### Rule 1: One Logical Thing per Commit

Do not mix:
- A new entity AND a refactor of an existing service
- A bug fix AND a new feature
- Backend changes AND Angular changes (commit them separately)

### Rule 2: Keep Every Module Compilable

After every increment, run:
```bash
mvn -T1C -q -DskipTests clean compile
```

### Rule 3: siteId Is Never Optional

Every new query slice that touches tenant data must include `siteId` from the first commit. Do not add it "later". Verify with:
```java
// Repository JPQL must always bind siteId from context
Long siteId = SiteIdThreadContext.getSiteId();
repository.findByXxxAndSiteId(xxx, siteId);
```

### Rule 4: @Transactional From the Start

Apply `@Transactional(readOnly = true)` or `@Transactional` immediately when writing `ServiceImpl` methods. Don't add them in a follow-up commit.

### Rule 5: Rollback-Friendly

- New entities / new columns are additive — easy to revert
- Never drop a column or rename a column and a service method in the same commit
- Liquibase `changeSet` must have a corresponding `rollback` block for any DDL change
- Feign client interface changes: update the provider first, then the consumer

## Working with Agents

When directing an agent to implement incrementally in Hermes:

```
"Let's implement Task 2 from the plan: Service interface + ServiceImpl for XxxService.

In scope:
- XxxService interface in service/
- XxxServiceImpl in service/impl/
- Unit tests in src/test/

Out of scope for now:
- Controller (Task 4)
- Angular component (Task 6)

After implementing, run: mvn test -pl execution-service"
```

Be explicit about module scope and what is NOT in scope.

## Increment Checklist (Java Backend)

After each increment:

- [ ] `mvn -T1C -q -DskipTests clean compile` — all modules compile
- [ ] `mvn test -pl <module>` — affected module's tests green
- [ ] No raw `RuntimeException` — only `BadRequestException` / `BusinessException`
- [ ] siteId filtered on every new tenant-scoped query
- [ ] `@Transactional` / `@Transactional(readOnly = true)` on all service methods
- [ ] `@PreAuthorize` on every new controller method
- [ ] `@Valid` on every `@RequestBody`
- [ ] Committed with a descriptive message

## Increment Checklist (Angular Frontend)

After each increment:

- [ ] `ng build` succeeds
- [ ] `ng lint` passes
- [ ] Component renders without console errors
- [ ] PrimeNG components use project-standard configuration
- [ ] HTTP calls go through the shared API service / interceptor chain
- [ ] Committed separately from backend changes

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll add siteId filtering after the feature works" | A feature without siteId is a data-leak bug from its first commit. Add it now. |
| "I'll add @Transactional later" | Missing transaction annotations cause data integrity bugs under concurrent load. Add them in the same commit as the method. |
| "These changes are too small to test" | `mvn test -pl <module>` takes seconds. Run it every increment. |
| "I'll commit backend and Angular together" | Mixed commits make rollback painful and code review harder. Always separate them. |

## Red Flags

- More than ~100 lines written without a compile check
- Backend and Angular changes in the same commit
- New query without siteId filter
- New `ServiceImpl` method missing `@Transactional`
- New controller method missing `@PreAuthorize`
- Running `mvn compile` twice on unchanged code — run it after each edit, not as reassurance

## Verification

After completing all increments for a task:

- [ ] Each increment was individually compiled, tested, and committed
- [ ] `mvn test -pl <module>` fully green
- [ ] `mvn -DskipTests clean package` succeeds
- [ ] Feature works end-to-end in local environment
- [ ] No uncommitted changes remain


## Overview

Build in thin vertical slices — implement one piece, test it, verify it, then expand. Avoid implementing an entire feature in one pass. Each increment should leave the system in a working, testable state. This is the execution discipline that makes large features manageable.

## When to Use

- Implementing any multi-file change
- Building a new feature from a task breakdown
- Refactoring existing code
- Any time you're tempted to write more than ~100 lines before testing

**When NOT to use:** Single-file, single-function changes where the scope is already minimal.

## The Increment Cycle

```
┌──────────────────────────────────────┐
│                                      │
│   Implement ──→ Test ──→ Verify ──┐  │
│       ▲                           │  │
│       └───── Commit ◄─────────────┘  │
│              │                       │
│              ▼                       │
│          Next slice                  │
│                                      │
└──────────────────────────────────────┘
```

For each slice:

1. **Implement** the smallest complete piece of functionality
2. **Test** — run the test suite (or write a test if none exists)
3. **Verify** — confirm the slice works as expected (tests pass, build succeeds, manual check)
4. **Commit** -- save your progress with a descriptive message (see `git-workflow-and-versioning` for atomic commit guidance)
5. **Move to the next slice** — carry forward, don't restart

## Slicing Strategies

### Vertical Slices (Preferred)

Build one complete path through the stack:

```
Slice 1: Create a task (DB + API + basic UI)
    → Tests pass, user can create a task via the UI

Slice 2: List tasks (query + API + UI)
    → Tests pass, user can see their tasks

Slice 3: Edit a task (update + API + UI)
    → Tests pass, user can modify tasks

Slice 4: Delete a task (delete + API + UI + confirmation)
    → Tests pass, full CRUD complete
```

Each slice delivers working end-to-end functionality.

### Contract-First Slicing

When backend and frontend need to develop in parallel:

```
Slice 0: Define the API contract (types, interfaces, OpenAPI spec)
Slice 1a: Implement backend against the contract + API tests
Slice 1b: Implement frontend against mock data matching the contract
Slice 2: Integrate and test end-to-end
```

### Risk-First Slicing

Tackle the riskiest or most uncertain piece first:

```
Slice 1: Prove the WebSocket connection works (highest risk)
Slice 2: Build real-time task updates on the proven connection
Slice 3: Add offline support and reconnection
```

If Slice 1 fails, you discover it before investing in Slices 2 and 3.

## Implementation Rules

### Rule 0: Simplicity First

Before writing any code, ask: "What is the simplest thing that could work?"

After writing code, review it against these checks:
- Can this be done in fewer lines?
- Are these abstractions earning their complexity?
- Would a staff engineer look at this and say "why didn't you just..."?
- Am I building for hypothetical future requirements, or the current task?

```
SIMPLICITY CHECK:
✗ Generic EventBus with middleware pipeline for one notification
✓ Simple function call

✗ Abstract factory pattern for two similar components
✓ Two straightforward components with shared utilities

✗ Config-driven form builder for three forms
✓ Three form components
```

Three similar lines of code is better than a premature abstraction. Implement the naive, obviously-correct version first. Optimize only after correctness is proven with tests.

### Rule 0.5: Scope Discipline

Touch only what the task requires.

Do NOT:
- "Clean up" code adjacent to your change
- Refactor imports in files you're not modifying
- Remove comments you don't fully understand
- Add features not in the spec because they "seem useful"
- Modernize syntax in files you're only reading

If you notice something worth improving outside your task scope, note it — don't fix it:

```
NOTICED BUT NOT TOUCHING:
- src/utils/format.ts has an unused import (unrelated to this task)
- The auth middleware could use better error messages (separate task)
→ Want me to create tasks for these?
```

### Rule 1: One Thing at a Time

Each increment changes one logical thing. Don't mix concerns:

**Bad:** One commit that adds a new component, refactors an existing one, and updates the build config.

**Good:** Three separate commits — one for each change.

### Rule 2: Keep It Compilable

After each increment, the project must build and existing tests must pass. Don't leave the codebase in a broken state between slices.

### Rule 3: Feature Flags for Incomplete Features

If a feature isn't ready for users but you need to merge increments:

```typescript
// Feature flag for work-in-progress
const ENABLE_TASK_SHARING = process.env.FEATURE_TASK_SHARING === 'true';

if (ENABLE_TASK_SHARING) {
  // New sharing UI
}
```

This lets you merge small increments to the main branch without exposing incomplete work.

### Rule 4: Safe Defaults

New code should default to safe, conservative behavior:

```typescript
// Safe: disabled by default, opt-in
export function createTask(data: TaskInput, options?: { notify?: boolean }) {
  const shouldNotify = options?.notify ?? false;
  // ...
}
```

### Rule 5: Rollback-Friendly

Each increment should be independently revertable:

- Additive changes (new files, new functions) are easy to revert
- Modifications to existing code should be minimal and focused
- Database migrations should have corresponding rollback migrations
- Avoid deleting something in one commit and replacing it in the same commit — separate them

## Working with Agents

When directing an agent to implement incrementally:

```
"Let's implement Task 3 from the plan.

Start with just the database schema change and the API endpoint.
Don't touch the UI yet — we'll do that in the next increment.

After implementing, run `npm test` and `npm run build` to verify
nothing is broken."
```

Be explicit about what's in scope and what's NOT in scope for each increment.

## Increment Checklist

After each increment, verify:

- [ ] The change does one thing and does it completely
- [ ] All existing tests still pass (`npm test`)
- [ ] The build succeeds (`npm run build`)
- [ ] Type checking passes (`npx tsc --noEmit`)
- [ ] Linting passes (`npm run lint`)
- [ ] The new functionality works as expected
- [ ] The change is committed with a descriptive message

**Note:** Run each verification command after a change that could affect it. After a successful run, don't repeat the same command unless the code has changed since — re-running on unchanged code adds no information.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll test it all at the end" | Bugs compound. A bug in Slice 1 makes Slices 2-5 wrong. Test each slice. |
| "It's faster to do it all at once" | It *feels* faster until something breaks and you can't find which of 500 changed lines caused it. |
| "These changes are too small to commit separately" | Small commits are free. Large commits hide bugs and make rollbacks painful. |
| "I'll add the feature flag later" | If the feature isn't complete, it shouldn't be user-visible. Add the flag now. |
| "This refactor is small enough to include" | Refactors mixed with features make both harder to review and debug. Separate them. |
| "Let me run the build command again just to be sure" | After a successful run, repeating the same command adds nothing unless the code has changed since. Run it again after subsequent edits, not as reassurance. |

## Red Flags

- More than 100 lines of code written without running tests
- Multiple unrelated changes in a single increment
- "Let me just quickly add this too" scope expansion
- Skipping the test/verify step to move faster
- Build or tests broken between increments
- Large uncommitted changes accumulating
- Building abstractions before the third use case demands it
- Touching files outside the task scope "while I'm here"
- Creating new utility files for one-time operations
- Running the same build/test command twice in a row without any intervening code change

## Verification

After completing all increments for a task:

- [ ] Each increment was individually tested and committed
- [ ] The full test suite passes
- [ ] The build is clean
- [ ] The feature works end-to-end as specified
- [ ] No uncommitted changes remain

## See Also

Per-increment verification is the local check. Before declaring a task done, apply the project-wide Definition of Done as the final gate, the standing bar every increment clears regardless of the task. See `references/definition-of-done.md`.


- [Print Unit Label](context/printing-unit-label.md)