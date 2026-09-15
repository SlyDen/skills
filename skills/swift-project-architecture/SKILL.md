---
name: swift-project-architecture
description: Design, implement, refactor, or review Swift project architecture using pragmatic modern Swift principles. Use for Swift, SwiftUI, Swift Package Manager, server-side Swift, Vapor, Hummingbird, persistence, dependency injection, feature/module boundaries, domain modeling, repositories, use cases, DTOs, transactions, testing strategy, or Swift 6 concurrency architecture. Apply proportional architecture rather than mechanically generating Clean Architecture layers.
---

# Swift Project Architecture

Apply modern, pragmatic Swift architecture. Optimize for clear dependency direction, strong domain modeling, native Swift/SwiftUI data flow, Swift Concurrency safety, and the smallest architecture that makes the system easy to change and test.

The user's explicit requirements and the repository's applicable `AGENTS.md` instructions take precedence over this skill.

## Workflow

1. Inspect the existing repository before proposing architecture changes. Read applicable `AGENTS.md`, `Package.swift`, target structure, feature layout, persistence/network boundaries, and relevant tests.
2. Identify the business behavior and invariants first. Do not begin by inventing controllers, repositories, ViewModels, DTOs, or packages.
3. Choose the smallest architecture that preserves the boundaries the feature actually needs.
4. Keep dependency direction inward: framework/presentation/infrastructure code may depend on application/domain code; domain code must not depend on frameworks, databases, HTTP, or UI concerns.
5. Prefer feature-oriented organization. Add package/module boundaries only when they provide real dependency enforcement, reuse, ownership, platform separation, compilation, or build-time value.
6. Use modern Swift: value types by default, `async`/`await`, structured concurrency, `Sendable`, actors for genuinely shared mutable state, and explicit isolation. Never use `@unchecked Sendable` merely to silence diagnostics.
7. For SwiftUI, prefer native state management: `@State`, `@Binding`, `@Observable`, `@Environment`, and `.task`. Do not create a ViewModel for every view.
8. Introduce protocols at meaningful boundaries, not one protocol per concrete type. Prefer initializer injection and a clear composition root over service locators or hidden globals.
9. Keep persistence and transport representations from leaking into domain logic. Add DTOs/mappings when they protect a real boundary; avoid ceremonial duplication when no boundary benefit exists.
10. Put business atomicity at the application/use-case level and database transaction mechanics in infrastructure.
11. Test behavior at the cheapest useful layer: pure domain tests, use-case tests with lightweight fakes, integration tests for persistence/migrations/transactions, and selective end-to-end tests.
12. Make the smallest coherent change. Do not refactor unrelated code or migrate a working architecture merely to match a diagram.

## Architecture Decision Rule

Before adding an abstraction, ask what coupling it removes, what change it makes safer, what invariant it enforces, or what test it materially simplifies. If there is no convincing answer, prefer the concrete and simpler design.

Use Clean/Hexagonal/Onion concepts as dependency constraints, not as a template generator.

## Detailed Guidance

Read [references/architecture-guide.md](references/architecture-guide.md) when the task needs detailed guidance or examples for domain modeling, application use cases, repositories and queries, DTOs, error translation, authorization, transactions, infrastructure/database access, server HTTP boundaries, SwiftUI, Swift Concurrency, dependency injection, SPM modules, testing, API design, observability, security, or review checklists.

For small changes, do not load the entire reference unless needed. Apply the workflow above and inspect only the relevant reference sections.
