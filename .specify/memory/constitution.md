# Go-Plate Constitution

<!--
SYNC IMPACT REPORT
==================
Version Change: Initial → 1.0.0
Modified Principles: N/A (Initial creation)
Added Sections:
  - Core Principles (7 principles)
  - Complexity Constraints
  - Governance
Removed Sections: N/A
Templates Requiring Updates:
  ✅ plan-template.md - Constitution Check section ready to reference these principles
  ✅ spec-template.md - Requirements align with API-First, TDD, and Error Handling principles
  ✅ tasks-template.md - Task structure supports TDD workflow and repository pattern organization
Follow-up TODOs: None - all placeholders filled
-->

## Core Principles

### I. API-First Design

**MUST** create OpenAPI/Swagger documentation before implementation.

**Rationale**: API contracts serve as executable specifications that align frontend, backend, and client expectations before a single line of implementation code is written. This prevents integration surprises and enables parallel development.

**Non-Negotiable Rules**:
- OpenAPI 3.x specification MUST exist before endpoint implementation begins
- All endpoints MUST be documented with request/response schemas, status codes, and error formats
- Contract tests MUST validate implementation against OpenAPI spec
- Breaking changes MUST increment API version

### II. Test-Driven Development

**MUST** write tests before implementation, especially for payment flows.

**Rationale**: Payment systems demand correctness. TDD ensures every code path is tested, prevents regressions, and serves as living documentation. For financial operations, test coverage is a security and compliance requirement, not a nice-to-have.

**Non-Negotiable Rules**:
- Tests written → Tests fail → Then implement (Red-Green-Refactor)
- Payment flows MUST achieve 100% branch coverage
- Integration tests MUST cover money-in and money-out scenarios
- No code merged without passing tests

### III. Error Handling

**MUST** use consistent error response format and proper HTTP status codes.

**Rationale**: Clients depend on predictable error structures for retry logic, user messaging, and debugging. Inconsistent errors cause client-side brittleness and poor user experience.

**Non-Negotiable Rules**:
- All errors MUST follow standardized JSON format: `{"error": {"code": "ERR_CODE", "message": "Human readable", "details": {...}}}`
- HTTP status codes MUST be semantically correct (4xx client errors, 5xx server errors)
- Payment errors MUST include idempotency guidance and safe retry indicators
- Errors MUST be logged with correlation IDs for traceability

### IV. Database Transactions

**MUST** make all multi-step operations atomic.

**Rationale**: Payment operations involve multiple state changes (account balance, transaction log, audit trail). Partial failures leave the system in inconsistent states that are costly and dangerous to reconcile.

**Non-Negotiable Rules**:
- Payment workflows MUST use database transactions
- Transaction boundaries MUST be explicit and documented
- Idempotency keys MUST be enforced at database level
- Rollback logic MUST be tested with failure injection

### V. Code Organization

**MUST** maintain clear separation of concerns: handler → service → repository.

**Rationale**: Layered architecture enables independent testing, facilitates code reuse, and keeps HTTP concerns separate from business logic. This separation is critical for unit testing business rules without spinning up HTTP servers.

**Non-Negotiable Rules**:
- **Handlers** (controllers) MUST only handle HTTP request/response marshaling and validation
- **Services** MUST contain business logic and orchestrate repository calls
- **Repositories** MUST abstract data access; no SQL in service layer
- Dependencies MUST flow inward: handler → service → repository

### VI. Documentation

**MUST** provide GoDoc comments for all public functions.

**Rationale**: Go's tooling ecosystem depends on GoDoc for discoverability and usage guidance. Public APIs without documentation impose cognitive overhead on users and maintainers.

**Non-Negotiable Rules**:
- All exported functions, types, and packages MUST have GoDoc comments
- Comments MUST start with the function/type name: `// ProcessPayment handles...`
- Complex business rules MUST include examples in comments
- Package-level documentation MUST describe package purpose and usage patterns

### VII. Dependency Injection

**MUST** use interfaces for testability.

**Rationale**: Interfaces enable mock injection for unit tests, decouple implementations from contracts, and support concurrent development of modules against stable interfaces.

**Non-Negotiable Rules**:
- Services MUST accept interfaces, not concrete types
- External dependencies (HTTP clients, databases, payment gateways) MUST be injected via interfaces
- Constructors MUST validate injected dependencies (non-nil checks)
- Tests MUST use mocks/stubs, not real external services

## Complexity Constraints

**Guiding Philosophy**: Start simple. Add complexity only when profiling or requirements demand it.

- **Business Logic Separation**: Keep business logic separate from HTTP handlers (see Principle V)
- **Repository Pattern**: Use repository pattern for data access abstraction (see Principle V)
- **Premature Optimization**: Avoid premature optimization; start simple and profile before optimizing
- **Standard Library First**: Prefer standard library over third-party packages when feasible (exceptions: well-vetted libraries for crypto, HTTP routing, database drivers)

**Complexity Justification Required For**:
- Introduction of message queues or asynchronous processing (justify with throughput measurements)
- Caching layers (justify with latency profiles)
- Microservice splits (justify with team size or scaling bottlenecks)
- Custom frameworks or DSLs (justify why standard library insufficient)

## Governance

**Supremacy**: This constitution supersedes all other practices, guidelines, and preferences. When in conflict, constitution wins.

**Amendment Procedure**:
1. Proposed changes MUST be documented with rationale and impact analysis
2. Amendments require approval from project maintainers
3. Constitution version MUST increment (MAJOR for removals/redefinitions, MINOR for additions, PATCH for clarifications)
4. Migration plan required for breaking changes to developer workflows

**Compliance Review**:
- All PRs MUST verify compliance with constitution principles
- Code reviews MUST call out violations explicitly
- Complexity introduced MUST be justified against Complexity Constraints
- Waivers require documented rationale and approval

**Versioning Policy**:
- MAJOR: Backward incompatible governance changes or principle removals
- MINOR: New principles or material expansions of existing principles
- PATCH: Clarifications, typo fixes, wording refinements

**Version**: 1.0.0 | **Ratified**: 2025-11-10 | **Last Amended**: 2025-11-10
