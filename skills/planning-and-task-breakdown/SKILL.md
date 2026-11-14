---
name: planning-and-task-breakdown
description: Breaks work into ordered tasks. Use when you have a spec or clear requirements and need to break work into implementable tasks. Use when a task feels too large to start, when you need to estimate scope, or when parallel work is possible.
---

# Planning and Task Breakdown — Spotify Microservices Project

## Overview

Decompose work into small, verifiable tasks with explicit acceptance criteria. This project is a Maven multi-module Java 17 / Spring Boot 3.1.5 + PostgreSQL + Angular microservices monorepo (`api-gateway`, `spotify-media`, `spotify-notification`, etc.). Good task breakdown prevents cross-service regressions, keeps increments compilable, and surfaces hidden data-isolation, security (Keycloak), or transaction concerns before they become bugs.

## When to Use

- You have a spec and need to break it into implementable units
- A task feels too large or vague to start
- Work spans multiple Maven microservices or touches both backend and Angular frontend
- The implementation order isn't obvious (e.g., new entity requires Flyway migration + repository + service + controller + Angular page)

**When NOT to use:** Single-file changes with obvious scope, or when the spec already contains well-defined tasks.

## The Planning Process

### Step 1: Enter Plan Mode (Read-Only)

Before writing any code:

- Read the relevant module(s) under `api-gateway/`, `spotify-media/`, `spotify-notification/`, etc.
- Identify existing patterns: entity hierarchy, service interface → `ServiceImpl`, OpenFeign clients, ActiveMQ/event publishers, Keycloak security context
- Map dependencies between modules (shared DTOs/utilities in common modules)
- Note risks: new Flyway DB migrations, OpenFeign/API contract changes, Angular API client updates

**Do NOT write code during planning.** The output is a plan document, not implementation.

### Step 2: Identify the Dependency Graph

Layer order for any new feature:

```
DB schema / Flyway migration
        │
        ├── Entity (JPA / Hibernate)
        │       │
        │       ├── Repository interface (Spring Data JPA)
        │       │       │
        │       │       └── Service interface  →  ServiceImpl
        │       │               │
        │       │               ├── Controller (@PreAuthorize / @Valid)
        │       │               │       │
        │       │               │       └── OpenFeign client / Message Queue (other services)
        │       │               │
        │       │               └── DTO Mapper (MapStruct / manual mapper)
        │       │
        └── Angular feature module
                ├── Service (HttpClient / shared API service)
                ├── Component (.ts / .html / .scss)
                └── Model (TypeScript interface)
```

Build bottom-up: schema → entity → repository → service → controller → frontend.

### Step 3: Slice Vertically

Build one complete feature path at a time, not one layer at a time.

**Bad (horizontal slicing):**
```
Task 1: Write all entities for the new module
Task 2: Write all repositories
Task 3: Write all services
Task 4: Write all controllers
Task 5: Write Angular components
```

**Good (vertical slicing):**
```
Task 1: Create Flyway migration + entity + repository (with user/owner scope check)
Task 2: Service interface + ServiceImpl (happy path + error paths)
Task 3: Controller + @PreAuthorize + @Valid DTO + unit tests
Task 4: Angular service + model (TypeScript)
Task 5: Angular list component
Task 6: Angular detail / form component
```

Each vertical slice produces a compilable, testable increment.

### Step 4: Write Tasks

Each task follows this structure:

```markdown
## Task [N]: [Short descriptive title]

**Module:** `spotify-<module-name>` / `spotify-ui`

**Description:** One paragraph explaining what this task accomplishes.

**Acceptance criteria:**
- [ ] [Specific, testable condition]
- [ ] User/ownership authorization is enforced on every user-scoped query/action
- [ ] `@Transactional(readOnly = true)` on read methods; `@Transactional` on writes

**Verification:**
- [ ] `mvn test -pl <module>` passes
- [ ] `mvn -DskipTests clean compile` succeeds
- [ ] Manual check: [description of what to verify]

**Dependencies:** [Task numbers this depends on, or "None"]

**Files likely touched:**
- `<module>/src/main/java/com/v2nhung/spotify/<module>/entity/XxxEntity.java`
- `<module>/src/main/java/com/v2nhung/spotify/<module>/repository/XxxRepository.java`
- `<module>/src/main/java/com/v2nhung/spotify/<module>/service/XxxService.java`
- `<module>/src/main/java/com/v2nhung/spotify/<module>/service/impl/XxxServiceImpl.java`
- `<module>/src/main/java/com/v2nhung/spotify/<module>/controller/XxxController.java`
- `<module>/src/main/resources/db/migration/V<version>__<description>.sql`

**Estimated scope:** [XS: 1 file | S: 1-3 files | M: 3-6 files | L: 6-10 files]
```

### Step 5: Order and Checkpoint

Arrange tasks so that:

1. Flyway DB migration / entity first (other tasks depend on it)
2. Repository before service (service depends on repository)
3. Service before controller (controller delegates to service)
4. OpenFeign client / event contract changes before consumer services
5. Angular changes last (depends on stable REST contract)
6. High-risk tasks early (fail fast)

Add explicit checkpoints:

```markdown
## Checkpoint: After Tasks 1-3 (Backend Core)
- [ ] `mvn -DskipTests clean compile` — all modules compile
- [ ] `mvn test -pl <module>` — service unit tests green
- [ ] No raw `RuntimeException`; domain/business exceptions handled via `@ControllerAdvice`
- [ ] User/owner access checks present on all protected queries

## Checkpoint: After Tasks 4-6 (Frontend)
- [ ] `ng build` succeeds
- [ ] Angular lint passes
- [ ] Core flow works end-to-end in local environment
- [ ] Review with human before merging
```

## Task Sizing Guidelines

| Size | Files | Scope | Example |
| :--- | :--- | :--- | :--- |
| **XS** | 1 | Single method or config change | Add a validation rule or query filter |
| **S** | 1-3 | One entity + repository or one component | New repository method + service wrapper |
| **M** | 3-6 | One feature slice | New playlist action: entity + repo + service + controller |
| **L** | 6-10 | Multi-module feature | New notification trigger touching `spotify-media` + `spotify-notification` + Angular |
| **XL** | 10+ | **Too large — break it down further** | — |

If a task is XL, decompose before starting. An agent performs best on S and M tasks.

**When to break a task down further:**
- It would take more than one focused session (roughly 2+ hours of agent work)
- It touches two or more independent Maven microservices
- It requires both backend changes and Angular changes
- Acceptance criteria needs more than 3 bullet points
- Task title contains "and" (a sign it is two tasks)

## Plan Document Template

```markdown
# Implementation Plan: [Feature/Project Name]

## Overview
[One paragraph summary of what we're building]

## Affected Modules
- `spotify-<module>` — [why]
- `api-gateway` — [why]

## Architecture Decisions
- [Key decision 1 and rationale]
- [Key decision 2 and rationale]

## Design Notes (SOLID + GOF)
- **SOLID Findings:** [SRP/OCP/LSP/ISP/DIP impact]
- **GOF Candidate(s):** [Pattern & location, or "None — direct implementation preferred"]
- **Simpler Alternative Considered:** [Alternative and trade-off]

## Task List

### Phase 1: Data Layer
- [ ] Task 1: Flyway migration + Entity
- [ ] Task 2: Repository (with ownership/access filter)

### Checkpoint: Data Layer
- [ ] Compile passes, existing tests green

### Phase 2: Business Logic
- [ ] Task 3: Service interface + ServiceImpl
- [ ] Task 4: Unit tests (happy path + failure paths)

### Checkpoint: Business Logic
- [ ] `mvn test -pl <module>` green

### Phase 3: API
- [ ] Task 5: Controller + DTO + `@Valid` + `@PreAuthorize`

### Checkpoint: API
- [ ] Postman / curl smoke test passes

### Phase 4: Frontend
- [ ] Task 6: Angular service + TypeScript model
- [ ] Task 7: Angular component

### Checkpoint: Complete
- [ ] End-to-end flow works in local env
- [ ] All acceptance criteria met

## Risks and Mitigations
| Risk | Impact | Mitigation |
| :--- | :--- | :--- |
| Missing user/owner filter on new query | High — unauthorized data access | Review every JPA/JPQL query before merge |
| OpenFeign / event contract change | Medium — breaks consumer services | Define DTO/event contract first, coordinate deploy order |

## Open Questions
- [Question needing human input]
```

## Parallelization Opportunities

When multiple agents or sessions are available:

- **Safe to parallelize:** Independent feature slices in different microservices, Angular components after REST contract is stable, unit tests for already-implemented services, documentation
- **Must be sequential:** Flyway migrations, shared entity changes, OpenFeign client interface or event schema changes
- **Needs coordination:** Any change to shared modules or API Gateway routes (affects multiple services)

## Common Rationalizations

| Rationalization | Reality |
| :--- | :--- |
| "I'll figure it out as I go" | That's how you end up with a tangled mess and rework. 10 minutes of planning saves hours. |
| "The tasks are obvious" | Write them down anyway. Explicit tasks surface hidden dependencies and forgotten edge cases. |
| "Planning is overhead" | Planning is the task. Implementation without a plan is just typing. |
| "I can hold it all in my head" | Context windows are finite. Written plans survive session boundaries and compaction. |

## Project-Specific Red Flags

- Starting implementation without checking if the microservice already has a similar pattern
- Missing user/owner access verification on user-scoped queries or mutations
- Creating a `ServiceImpl` method that is `> ~300 lines` — break it down further
- No `@Transactional` annotation on write methods
- No `@Valid` on `@RequestBody` parameters
- Modifying database schema directly without a versioned Flyway migration script
- Task list has no verification step that runs `mvn test`

## Verification

Before starting implementation, confirm:

- [ ] Every task has acceptance criteria including ownership / `@Transactional` checks
- [ ] Every task has a verification step
- [ ] Task dependency order follows: Flyway schema → entity → repo → service → controller → frontend
- [ ] No task touches more than ~6 files
- [ ] Checkpoints exist between backend and frontend phases
- [ ] The human has reviewed and approved the plan

## See Also

Acceptance criteria are per-task and answer "did we build the right thing?". They sit on top of the project-wide Definition of Done, the standing bar every task clears before it counts as done.