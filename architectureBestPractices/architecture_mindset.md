# Architecture Mindset: Reviewing Architecture & Code

> A mental framework for thinking like an architect — whether you're designing systems, reviewing PRs, or evaluating existing codebases.

---

## 1. Think in Systems, Not Just Components

Before diving into individual modules or functions, zoom out.

- **What problem does this system solve?** Understand the business intent before judging the technical choices.
- **What are the system boundaries?** Identify what's inside vs. outside the scope of ownership.
- **What does failure look like?** Trace failure modes — what breaks if this component goes down?
- **What are the data flows?** Follow data from entry point to exit. Where is it transformed, stored, or exposed?

---

## 2. Evaluate Non-Functional Requirements (NFRs)

Most bugs in architecture are invisible in the happy path. Ask:

| Concern | Questions to Ask |
|---|---|
| **Scalability** | Can this handle 10x the load? Where is the bottleneck? |
| **Reliability** | What are the SLAs? Is there retry logic, circuit breaking, fallback? |
| **Maintainability** | Can a new engineer understand this in 30 minutes? |
| **Security** | Are secrets managed properly? Are inputs validated? Is least-privilege enforced? |
| **Observability** | Is there logging, tracing, and alerting? Can you debug a production issue? |
| **Testability** | Can units be tested in isolation? Are integration tests feasible? |
| **Performance** | Are there N+1 queries, blocking I/O, or unbounded loops? |

---

## 3. Separation of Concerns (SoC)

A healthy architecture has clear boundaries of responsibility.

- **Does each module/service have a single, well-defined purpose?**
- **Are cross-cutting concerns (auth, logging, error handling) centralized or duplicated?**
- **Is business logic leaking into the UI, DB layer, or infrastructure?**
- **Are concerns mixed that should be separated?** (e.g., data access logic inside a controller)

---

## 4. Coupling & Cohesion

> Aim for **high cohesion** within components and **loose coupling** between them.

- Are modules tightly coupled through shared state or direct references they shouldn't need?
- Can you swap out a dependency (DB, message queue, HTTP client) without touching business logic?
- Are interfaces/contracts well-defined, or does consumer code need to know internal implementation details?
- Watch for: circular dependencies, God objects, and implicit global state.

---

## 5. Data Model & Storage Decisions

The data model outlives the code. Get it right.

- Is the data model aligned with the domain model?
- Are the right storage engines chosen? (Relational vs. document vs. graph vs. cache)
- Are indexes defined on frequently queried fields?
- Is there a strategy for schema evolution and migrations?
- Are transactions used correctly? Is eventual consistency acceptable where used?
- Is sensitive data encrypted at rest and in transit?

---

## 6. API & Interface Design

Public interfaces are contracts — hard to change once exposed.

- Is the API intuitive and consistent in naming, versioning, and error responses?
- Are APIs idempotent where they should be (especially for mutations)?
- Is there over-fetching or under-fetching in API responses?
- Is the API versioned? How are breaking changes handled?
- Are error codes and messages actionable for the caller?

---

## 7. Resilience & Error Handling

- Are all external calls (HTTP, DB, queue) wrapped with timeout and retry logic?
- Are errors caught at the right layer — not swallowed silently?
- Is there graceful degradation when a dependency is unavailable?
- Are exceptions treated as exceptional? Or used for control flow?
- Is there a distinction between transient and permanent failures?

---

## 8. Security Mindset

- **Authentication & Authorization**: Is identity verified? Is access scoped per role/resource?
- **Input Validation**: Is all user input validated and sanitized before use?
- **Secrets Management**: Are credentials in environment variables / secret stores — never hardcoded?
- **Injection Prevention**: SQL injection, XSS, SSRF, command injection — are these addressed?
- **Dependency Risk**: Are third-party libraries audited? Are versions pinned?
- **Audit Logging**: Are sensitive operations logged for forensic traceability?

---

## 9. Code Quality Signals

When reviewing code through an architectural lens:

- **Naming**: Do names reveal intent? Avoid abbreviations and vague names like `data`, `manager`, `handler`.
- **Function/Method Size**: Functions that do too much are a design smell. Prefer small, composable units.
- **Abstraction Levels**: Does a single function mix high-level orchestration with low-level implementation?
- **DRY vs. WET**: Is duplication accidental or intentional? Shared logic should be extracted; unrelated logic shouldn't be forcefully unified.
- **Immutability**: Is state mutated when it doesn't need to be?
- **Dead Code**: Are there unused variables, unreachable branches, or legacy code left behind?

---

## 10. Evolutionary & Operational Fit

Architecture must survive in the real world.

- **Is the complexity justified?** Don't build distributed systems for single-team problems.
- **Can the team operate this?** A Kubernetes microservices setup is wrong if the team is 3 people.
- **Is it cost-aware?** Architectural choices have cloud cost implications.
- **Is there a clear deployment strategy?** CI/CD, feature flags, rollback plans.
- **Is there a migration path?** How does the system evolve? Can old and new versions coexist?

---

## 11. Architecture Review Checklist

Use this as a quick reference during reviews:

```
[ ] Functional requirements clearly understood
[ ] Non-functional requirements (perf, scale, reliability) addressed
[ ] Single responsibility per module/service
[ ] Loose coupling between components
[ ] Data model aligned with domain; storage choices justified
[ ] API contracts well-defined and versioned
[ ] Error handling and resilience patterns applied
[ ] Security: auth, input validation, secrets, least-privilege
[ ] Observability: logging, metrics, tracing in place
[ ] Testability: unit, integration, e2e coverage strategy
[ ] Operational fit: team can build, deploy, and maintain this
[ ] No premature optimization or over-engineering
[ ] Technical debt documented and tracked
```

---

## 12. The Right Questions to Ask (Not Just What, But Why)

| Instead of... | Ask... |
|---|---|
| "Is this correct?" | "Is this the simplest correct solution?" |
| "Does it work?" | "Will it work under load, failure, and edge cases?" |
| "Is it readable?" | "Can a new team member maintain this safely?" |
| "Is it fast?" | "Is it fast enough for the actual requirement?" |
| "Is it scalable?" | "Does it need to scale, and in which dimension?" |

---

## Key Principles to Live By

> **Simple > Clever**
> Clever code is hard to debug, maintain, and review. Prefer clarity.

> **Explicit > Implicit**
> Magic and convention hide intent. Make dependencies, flows, and assumptions visible.

> **Design for failure**
> Assume every external dependency will fail. Design accordingly.

> **Make it easy to change**
> The best architecture is one that lets you move fast safely — today and next year.

> **Defer decisions that don't need to be made now**
> Don't over-architect for hypothetical futures. Build for today's real constraints, with tomorrow in mind.

---

*Use this document as a thinking tool — not a checkbox exercise. The goal is to ask the right questions, surface hidden assumptions, and build systems that are correct, maintainable, and resilient.*
