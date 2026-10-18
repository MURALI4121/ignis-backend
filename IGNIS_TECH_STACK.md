# IGNIS Technology Stack and Engineering Standards

**Status:** Approved project baseline  
**Applies to:** IGNIS web application, planned Android and iOS applications, and shared backend API  
**Audience:** Developers, reviewers, and AI coding assistants

## Purpose and authority

This document records the approved technology stack and engineering standards for IGNIS. Follow it for implementation decisions unless the project owner approves an explicit architectural decision that changes it. Do not silently replace a listed technology or introduce a deferred one.

This is a technology and engineering guide, not a complete product specification. Feature behavior, API contracts, and detailed domain decisions belong in the relevant product and architecture documents. If another document appears to select a conflicting technology, resolve the conflict before implementation; do not assume the older choice overrides this approved stack.

Exact dependency versions are intentionally not specified here. Select compatible stable versions during project initialization, follow framework compatibility guidance, and manage versions reproducibly.

## Contents

- [1. Product and architecture context](#1-product-and-architecture-context)
- [2. Approved technology stack](#2-approved-technology-stack)
- [3. Frontend engineering requirements](#3-frontend-engineering-requirements)
- [4. Backend engineering requirements](#4-backend-engineering-requirements)
- [5. Architecture and design principles](#5-architecture-and-design-principles)
- [6. Initial backend module structure](#6-initial-backend-module-structure)
- [7. Financial data and security standards](#7-financial-data-and-security-standards)
- [8. Deferred technologies](#8-deferred-technologies)
- [9. Development and delivery standards](#9-development-and-delivery-standards)
- [10. Implementation roadmap](#10-implementation-roadmap)
- [11. Decision and version management](#11-decision-and-version-management)

## 1. Product and architecture context

IGNIS is a personal life-management platform. The initial release focuses on personal finance and expense tracking. Later releases may add habits, fitness, nutrition, productivity, goals, and other personal management features.

### Platform direction

- Build the web application first with React Native and Expo, using their web support where practical.
- Add Android and iOS applications in later phases.
- Reuse frontend components, domain logic, validation, and API contracts where platform behavior allows.
- Serve web and mobile clients from one Spring Boot backend API.
- Use a modular monolith initially: one deployable backend application with clear internal feature boundaries.
- Keep the first implementation understandable and testable. Add infrastructure only when a real product or operational requirement justifies it.

### Architecture outline

The frontend uses Expo Router to present platform-appropriate routes and screens, then communicates with the backend REST API through a centralized API layer. Spring Boot contains the identity integration, finance features, persistence, and operational endpoints in one deployable application.

Keycloak owns user authentication and identity lifecycle. Spring Security validates the resulting authenticated identity at the backend and enforces access rules. For web, use a same-origin secure HttpOnly cookie session design where appropriate; the approved web architecture uses a backend-for-frontend (BFF) so browser JavaScript does not handle OAuth tokens. For native mobile clients, use the OIDC Authorization Code flow with PKCE and platform-appropriate redirect handling; store required tokens with Expo SecureStore. Never store long-lived authentication tokens in browser LocalStorage.

The backend is authoritative for validation, authorization, ownership, and financial calculations. Client-side checks improve usability but do not replace server-side enforcement.

## 2. Approved technology stack

### Frontend

| Category | Approved technology | Purpose and usage |
|---|---|---|
| UI framework | React Native | Shared UI foundation for web and planned mobile applications. Use platform-specific components or files where behavior differs. |
| Development platform | Expo | Development, platform tooling, and support for web, Android, and iOS targets. |
| Programming language | TypeScript | Type-safe application and domain code across the frontend. |
| Routing | Expo Router | File-based navigation and route organization. |
| Styling | NativeWind | Tailwind-style utility-based styling for supported React Native targets. |
| Forms | React Hook Form | Form state, submission, and field lifecycle. |
| Validation | Zod | Shared, typed validation schemas and input feedback. The backend still repeats authoritative validation. |
| HTTP communication | Fetch API | Centralized communication with backend REST APIs. |
| Server-state management | TanStack Query | Fetching, caching, invalidation, synchronization, and request lifecycle state for server data. |
| Authentication | Keycloak with OIDC and PKCE | Standard identity integration. Use platform-appropriate redirect handling and secure session/token management. |
| Mobile secure storage | Expo SecureStore | Platform-appropriate storage for mobile authentication material that must persist. |
| Web session security | Secure HttpOnly cookie-based session design where appropriate | Protect browser sessions; avoid long-lived tokens in LocalStorage. Follow the approved BFF flow for web. |
| Icons | Lucide package compatible with React Native | Consistent iconography across supported platforms. |
| Charts | Cross-platform compatible chart library, selected during dashboard implementation | Financial analytics and visualization. Select it when dashboard needs are known; do not introduce it earlier. |
| Testing | Jest or compatible tooling and React Native Testing Library | Unit and component behavior tests suited to the Expo/React Native setup. |
| Linting | ESLint | Consistent code-quality checks. |
| Formatting | Prettier | Consistent formatting. |

Do not add another frontend state-management, styling, routing, or HTTP framework without an approved architectural decision. The selected chart package is intentionally deferred until implementation demonstrates the required chart types and platform support.

### Backend and identity

| Category | Approved technology | Purpose and usage |
|---|---|---|
| Programming language | Java LTS release compatible with the selected framework | Backend implementation and long-term support baseline. Select the exact LTS release during project initialization. |
| Application framework | Spring Boot | Runtime, application configuration, dependency management, and production integrations. |
| REST API framework | Spring Web MVC | HTTP endpoints, request handling, and JSON APIs. |
| Security | Spring Security | Authentication integration, request security, and backend authorization. |
| Identity provider | Keycloak | User identity, authentication, and identity lifecycle integration. |
| Authentication protocols | OAuth 2.0 and OpenID Connect | Standard authorization and identity flows between clients, Keycloak, and the backend. |
| Token format | JWT | Signed identity and authorization claims validated by the backend. |
| ORM | Hibernate | Object-relational mapping. |
| Persistence abstraction | Spring Data JPA | Repository and database access abstractions. |
| Database | PostgreSQL | Relational persistence, transactions, constraints, and query support. |
| Schema migrations | Flyway | Version-controlled, ordered schema changes. |
| Request validation | Spring Validation | Server-side validation at API boundaries. |
| Build tool | Maven | Dependency resolution, compilation, tests, and packaging. |
| Testing | JUnit 5, Mockito, and Spring Boot Test | Unit, slice, security, and application integration tests. |
| Database integration testing | Testcontainers | Integration testing against a real PostgreSQL instance in an isolated test environment. |
| API documentation | OpenAPI and Swagger UI | Machine-readable API contracts and interactive development documentation. |
| Logging | SLF4J and Logback | Application logging through a standard logging facade and runtime implementation. |
| Health and metrics | Spring Boot Actuator and Micrometer | Health reporting and application metrics. Protect operational endpoints. |
| Distributed tracing | OpenTelemetry when needed | Trace propagation and diagnostics if deployment or support needs justify it. |
| Code coverage | JaCoCo | Measuring test coverage and identifying untested code. |
| Code quality | Checkstyle or an equivalent agreed rule set | Enforcing agreed Java style and maintainability checks. |

### Cross-platform authentication behavior

Authentication logic belongs in a dedicated frontend integration layer and in Spring Security, not inside finance features.

- **Web:** use the Keycloak OIDC login through the Spring Boot BFF. Keep OAuth tokens server-side and use an opaque secure session cookie in the browser. Require CSRF protection for state-changing cookie-authenticated requests.
- **Android and iOS:** use the system-appropriate OIDC redirect flow with Authorization Code and PKCE. Store tokens that need persistence using Expo SecureStore. Never embed confidential client secrets in a mobile binary.
- **Backend:** validate JWT signature, issuer, audience, expiry, and other required claims. Derive the current user from the validated security context. Enforce resource ownership on every operation.
- **All clients:** logout, token renewal, expired sessions, failed authorization, and account states must have consistent user-facing behavior while respecting platform-specific security constraints.

The backend remains the authority even when the frontend hides unavailable actions or protects routes for a smoother experience.

## 3. Frontend engineering requirements

1. Use TypeScript throughout the frontend, including components, hooks, API boundaries, and route code.
2. Organize reusable UI components, feature components, API services, validation schemas, domain types, hooks, and application routes by clear responsibility.
3. Use TanStack Query for remote server state. Do not create custom caches or duplicate server data in unrelated global state.
4. Use React Hook Form with Zod for form state and client-side validation. The server repeats all security- and data-relevant validation.
5. Centralize HTTP requests, authentication-aware request behavior, serialization, and API error handling.
6. Keep authentication/session logic separate from finance and future business features.
7. Use platform-specific implementations when browser, Android, and iOS behavior differs. Share code when it remains clear and correct.
8. Reuse business logic, API contracts, validation rules, and UI components where practical; do not force reuse that makes a platform harder to use.
9. Design responsive web layouts and native mobile layouts for their actual interaction patterns. A web layout is not assumed to work unchanged on a phone.
10. Follow accessibility semantics, keyboard and screen-reader behavior, sufficient contrast, consistent spacing, and a coherent IGNIS design system.
11. Model loading, empty, success, validation, offline, and failure states explicitly.
12. Avoid putting business rules in route files or visual components. Keep components focused on presentation and user interaction.

## 4. Backend engineering requirements

1. Start with one deployable Spring Boot modular monolith.
2. Organize backend code primarily by business feature, rather than relying only on application-wide controller, service, and repository packages.
3. Keep controllers responsible for HTTP concerns: mapping, request/response translation, and status codes.
4. Use application services to coordinate use cases, transactions, and calls between domain and persistence boundaries.
5. Keep business rules independent of HTTP and persistence concerns where useful; apply this boundary in proportion to actual complexity.
6. Use repositories for persistence access and make transaction boundaries explicit.
7. Use request and response DTOs. Never expose persistence entities directly from the API.
8. Use Spring dependency injection and constructor injection for required collaborators.
9. Use Flyway for every schema change. Do not use automatic ORM schema updates in production.
10. Use centralized exception handling and a consistent, documented API error response format.
11. Document the REST API with OpenAPI and keep the contract aligned with actual behavior.
12. Add unit, integration, security, and API tests. Test important authorization and cross-user isolation cases.
13. Set appropriate timeouts, transaction boundaries, indexes, constraints, and locking strategies based on access patterns and financial correctness.
14. Derive the acting user from the authenticated security context. Never trust a user ID supplied by the client to establish identity or ownership.
15. Keep business modules independently understandable. Share code through explicit, stable interfaces rather than uncontrolled cross-module references.
16. Add structured logs, correlation/request IDs, health checks, and metrics with appropriate access controls.
17. Keep credentials, signing keys, secrets, and environment-specific configuration out of source control.
18. Select compatible stable dependency versions and manage them explicitly. Do not invent or float versions in implementation guidance.

## 5. Architecture and design principles

Apply principles and patterns when they solve a real problem. They are tools for keeping IGNIS understandable and safe, not a checklist requiring an abstraction for every class.

### SOLID

- **Definition:** Five object-oriented design guidelines: one reason to change per unit (SRP); extend behavior without rewriting stable code (OCP); preserve substitutability (LSP); keep interfaces focused (ISP); depend on abstractions at important boundaries (DIP).
- **Why it matters:** Finance rules, identity, and persistence should evolve without forcing unrelated features to change.
- **IGNIS example:** Keep expense calculation and validation in the finance use case rather than mixing them into a web controller.
- **Avoid:** Creating interfaces for every class or splitting cohesive logic into tiny types just to claim SOLID compliance.

### DRY (Don't Repeat Yourself)

- **Definition:** Keep one authoritative representation of a rule or piece of knowledge.
- **Why it matters:** Duplicate money, date, validation, or ownership rules drift and can lead to inconsistent financial behavior.
- **IGNIS example:** Reuse a shared backend amount/currency policy across transaction use cases; keep frontend schemas aligned while treating backend validation as authoritative.
- **Avoid:** Forcing unrelated concepts into one generic abstraction because their code looks similar today.

### KISS (Keep It Simple)

- **Definition:** Choose the simplest design that meets current requirements safely.
- **Why it matters:** A modular monolith and a small finance domain are easier to build, secure, and operate than speculative distributed systems.
- **IGNIS example:** Use synchronous REST and a PostgreSQL transaction for recording an expense.
- **Avoid:** Removing necessary validation, authorization, or integrity checks in the name of simplicity.

### Separation of Concerns

- **Definition:** Give HTTP handling, business rules, persistence, identity, and presentation distinct responsibilities.
- **Why it matters:** Changes to screens or database mapping should not silently change financial rules.
- **IGNIS example:** A controller maps an expense request to a use case; a repository persists the resulting domain data.
- **Avoid:** Adding a layer that only forwards calls without a responsibility or useful boundary.

### High Cohesion and Low Coupling

- **Definition:** Keep code that changes for the same reason together and minimize knowledge between independent parts.
- **Why it matters:** Finance features can change without a cascade through identity, shared utilities, and future life-management modules.
- **IGNIS example:** Keep transaction behavior under finance/transaction and expose a small public interface for dashboard summaries.
- **Avoid:** Turning every feature into a sealed silo or duplicating truly shared primitives.

### Dependency Injection and Inversion of Control

- **Definition:** Supply collaborators from the application composition layer rather than constructing infrastructure dependencies throughout business code.
- **Why it matters:** It makes dependencies visible and allows tests to replace boundaries such as repositories or external identity clients.
- **IGNIS example:** Inject a transaction repository into the transaction application service.
- **Avoid:** A global service locator, excessive indirection, or injecting dependencies that are not used.

### Domain-Driven Design

- **Definition:** Model important domain concepts and language around the business problem, with boundaries that reflect distinct rules.
- **Why it matters:** Money, accounts, categories, expenses, and transfers have different invariants; making them explicit reduces accidental misuse.
- **IGNIS example:** Define a transaction type and amount/currency rules clearly rather than relying on ambiguous field names.
- **Avoid:** Adopting aggregates, domain events, repositories, and ubiquitous-language ceremonies where simple CRUD has no meaningful domain complexity.

### Clean Architecture concepts

- **Definition:** Keep essential application rules independent from delivery and infrastructure details, with dependencies pointing toward stable policies.
- **Why it matters:** The same finance use case should not depend on React routes, HTTP request classes, or a particular ORM mapping.
- **IGNIS example:** Keep an expense creation use case callable from a REST controller while persistence remains behind a repository boundary.
- **Avoid:** Building several layers of interfaces and mappers for a small operation with no meaningful change boundary.

### Hexagonal Architecture and Ports and Adapters

- **Definition:** Define application-facing ports and use adapters for external systems such as HTTP, PostgreSQL, or Keycloak.
- **Why it matters:** It helps isolate identity, database, and delivery details where those boundaries matter.
- **IGNIS example:** Treat Keycloak integration as an identity adapter configured through Spring Security.
- **Avoid:** Creating ports for every internal method or building a plugin system for infrastructure that will not vary.

### Repository Pattern

- **Definition:** Provide a focused persistence boundary for retrieving and changing domain data.
- **Why it matters:** It keeps query details out of controllers and makes ownership and transaction queries reviewable.
- **IGNIS example:** A transaction repository exposes operations that query records for the authenticated owner and selected date range.
- **Avoid:** A generic repository that hides important SQL behavior or returns data across owners by default.

### Service Layer Pattern

- **Definition:** Coordinate a complete application use case, validation, authorization context, and transactional work in an application service.
- **Why it matters:** Financial operations may need several consistent persistence changes, not just a row save.
- **IGNIS example:** Create an expense, check its category ownership, and persist it within one transaction.
- **Avoid:** Anemic services that only forward a call, or a giant service that owns every finance workflow.

### DTO Pattern

- **Definition:** Use explicit data transfer objects at system boundaries instead of exposing persistence or domain objects.
- **Why it matters:** It prevents accidental field exposure and permits the public API to evolve deliberately.
- **IGNIS example:** An ExpenseRequest accepts allowed input fields; an ExpenseResponse returns only approved values.
- **Avoid:** Repeating identical types across every layer without a boundary need, or returning entities because mapping seems inconvenient.

### Strategy Pattern

- **Definition:** Encapsulate a behavior behind a common contract when multiple interchangeable behaviors genuinely exist.
- **Why it matters:** Future rules such as calculation policies may vary by an explicit product setting.
- **IGNIS example:** Introduce a strategy only if a documented feature requires multiple dashboard aggregation policies.
- **Avoid:** Designing for hypothetical future strategies when there is only one behavior.

### Adapter Pattern

- **Definition:** Translate an external interface into the shape expected by an internal boundary.
- **Why it matters:** External identity and future integrations should not leak provider-specific details into finance features.
- **IGNIS example:** Map validated Keycloak identity claims to IGNIS's authenticated principal and local profile.
- **Avoid:** Wrapping internal code in adapters that translate nothing or add no isolation.

### Test-Driven Development principles

- **Definition:** Use tests to clarify behavior, then implement and refactor against those expectations.
- **Why it matters:** Explicit tests are valuable for money arithmetic, authorization, ownership isolation, and failure handling.
- **IGNIS example:** Add a test showing a user cannot read or edit another user's expense before changing that query.
- **Avoid:** Writing brittle tests for implementation details or requiring strict red-green-refactor ceremony for every small change.

### Secure by Design

- **Definition:** Include security and privacy requirements in architecture and implementation from the start.
- **Why it matters:** Financial records and identity data need protection at every API and storage boundary.
- **IGNIS example:** Validate JWTs, require authorization on every protected operation, scope queries by owner, and use secure session handling.
- **Avoid:** Treating frontend route guards, hidden buttons, or possession of an identifier as authorization.

### Database Normalization and Integrity

- **Definition:** Structure relational data to reduce unintended duplication and use constraints to protect valid states.
- **Why it matters:** Transactions, categories, profiles, and ownership must remain consistent even if application code has a defect.
- **IGNIS example:** Use foreign keys and uniqueness constraints for owner-scoped categories, plus database transactions for multi-step updates.
- **Avoid:** Over-normalizing read-heavy data without need or relying only on application checks for critical invariants.

### API Versioning and Backward Compatibility

- **Definition:** Evolve public API contracts in a controlled way so existing clients can continue to function.
- **Why it matters:** Web and mobile releases may not deploy at the same time.
- **IGNIS example:** Version the REST API path (initially /api/v1) and add compatible optional fields where possible.
- **Avoid:** Breaking clients for cosmetic changes or adding a new API version for every minor enhancement.

### Observability and Production Readiness

- **Definition:** Make application health and behavior measurable through safe logs, metrics, health checks, and traces when needed.
- **Why it matters:** Operators need to diagnose failures without exposing financial details or credentials.
- **IGNIS example:** Emit a correlation ID and route-level latency metric while redacting tokens and sensitive request fields.
- **Avoid:** Logging full request bodies, collecting metrics with high-cardinality personal data, or adding tracing infrastructure before operational needs justify it.

## 6. Initial backend module structure

The following is an illustrative logical package structure inside one Spring Boot application:

```text
ignis-backend/
├── src/
│   ├── main/
│   │   ├── java/com/ignis/
│   │   │   ├── IgnisApplication.java
│   │   │   ├── shared/
│   │   │   │   ├── config/
│   │   │   │   ├── security/
│   │   │   │   ├── exception/
│   │   │   │   └── observability/
│   │   │   ├── identity/
│   │   │   └── finance/
│   │   │       ├── account/
│   │   │       ├── category/
│   │   │       ├── transaction/
│   │   │       └── dashboard/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   └── test/
├── pom.xml
└── README.md
```

This is a logical organization, not a requirement to create a separate deployable service or Maven module for each feature. Add future habit, fitness, nutrition, and productivity modules when implementation begins.

Within a feature, use api, application, domain, and infrastructure packages when they provide useful boundaries. Avoid empty abstractions and unnecessary package layers.

## 7. Financial data and security standards

These are non-negotiable requirements:

- Use Java BigDecimal and PostgreSQL NUMERIC for money. Never use binary floating-point types for stored or calculated currency values.
- Make currency explicit on financial records and define supported precision and rounding rules.
- Define clear semantics for income, expense, account balance, and transfer operations. Do not infer one operation type from a positive or negative amount without an explicit domain rule.
- Use database constraints and transactions to maintain financial integrity.
- Scope every user-owned record to its authenticated owner. Check ownership on reads, updates, and deletes, including nested resources such as categories.
- Never rely solely on frontend validation, frontend route protection, or client-supplied owner identifiers.
- Validate every API input on the backend and return safe, consistent errors.
- Never expose passwords, tokens, secrets, stack traces, or SQL details in API responses or logs.
- Apply least-privilege authorization. Keep platform or support roles separate from ordinary personal-data ownership.
- Protect secrets and environment-specific configuration outside source control.
- Consider duplicate submissions, client retries, and idempotency for operations where repeated execution could cause incorrect financial results.
- Use pagination, bounded date ranges where appropriate, and query limits for collection endpoints.
- Use HTTPS in deployed environments. Restrict CORS to known clients only when cross-origin access is required.
- Protect cookie-authenticated state-changing requests against CSRF. Set secure cookie attributes appropriate to the deployment.
- Keep identity claims and authentication tokens out of logs and analytics.
- Treat user-provided text as untrusted input in every rendering context.

## 8. Deferred technologies

Deferred means not part of the current architecture. It does not mean a technology is permanently prohibited. Add one only when actual requirements justify it and the decision is documented.

| Technology | Reason for deferral |
|---|---|
| Microservices | Split services only when independent deployment, ownership, or scaling needs justify the added network and operational complexity. |
| Apache Kafka or RabbitMQ | Add when asynchronous processing or event-driven integration is needed. |
| Redis | Add when caching or shared ephemeral state is justified. |
| Kubernetes | Add when deployment and orchestration requirements justify its operational cost. |
| CQRS | Add when read/write complexity warrants separate models. |
| Event Sourcing | Add only if historical event reconstruction is a genuine requirement. |
| Elasticsearch | Add if PostgreSQL no longer meets actual search requirements. |
| GraphQL | Reconsider only if REST proves insufficient for real client needs. |

The current application should not depend on these technologies.

## 9. Development and delivery standards

- Use consistent code formatting, naming, and agreed lint/check rules.
- Keep builds and tests automated and reproducible.
- Commit database changes as Flyway migrations; test migrations against PostgreSQL.
- Keep local, test, staging, and production configuration separate.
- Use secure secret management and review dependency vulnerabilities and upgrades.
- Keep OpenAPI documentation current with implemented endpoint behavior.
- Use safe structured logs, health checks, and metrics; avoid sensitive data in telemetry.
- Review code for behavior, security, ownership isolation, financial correctness, and maintainability.
- Keep pull requests focused, explain user-visible or architectural changes, and include relevant test evidence.
- Introduce CI/CD as the project progresses; do not bind the architecture to a deployment vendor before one is selected.
- Provide a reproducible local development setup and document prerequisites.
- Use dependency management and compatibility rules rather than inventing or floating versions.
- Keep web, Android, and iOS platform-specific behavior behind clear boundaries when needed.
- Make architecture changes explicit and update this document when an approved decision changes.

## 10. Implementation roadmap

1. Establish the Expo/React Native TypeScript frontend and Spring Boot backend project foundations.
2. Configure the Java LTS release, Spring Boot, Maven, PostgreSQL, and Flyway.
3. Establish Keycloak and Spring Security integration, including the web session/BFF path and native PKCE flow.
4. Create the initial finance data model and migration baseline.
5. Implement accounts, categories, and transactions with ownership enforcement.
6. Add request validation, authorization, centralized error handling, and tests.
7. Document REST APIs with OpenAPI.
8. Build dashboard summary APIs and select the compatible chart library during dashboard implementation.
9. Connect the frontend to the backend through the centralized API layer.
10. Add operational hardening, security review, and a reproducible deployment workflow.

Treat this as an order of work, not a promise to implement future life-management modules in Phase 1.

## 11. Decision and version management

- Keep this file as the official technology stack and engineering standards reference.
- Record material changes with the decision, reason, trade-offs, and affected sections.
- Do not introduce an alternative framework, provider, database, or deployment platform without an explicit approved decision.
- At project initialization, select exact compatible stable versions. Record them in the relevant build files and setup documentation, not as speculative versions here.
- Upgrade dependencies deliberately, review compatibility and security notices, and keep lockfiles or dependency management files committed where applicable.
- Revisit deferred technologies only when product, scale, reliability, or operational evidence supports the added complexity.

