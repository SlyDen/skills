# Swift Project Architecture Guidelines

## Table of Contents

- [Swift Project Architecture Guidelines](#swift-project-architecture-guidelines)
- [Goal](#goal)
- [1. Architecture Selection](#1-architecture-selection)
- [Small feature or prototype](#small-feature-or-prototype)
- [Medium application](#medium-application)
- [Large or long-lived application](#large-or-long-lived-application)
- [2. Domain Layer](#2-domain-layer)
- [Domain invariants](#domain-invariants)
- [Creation types](#creation-types)
- [Domain identifiers](#domain-identifiers)
- [3. Protocols and Ports](#3-protocols-and-ports)
- [4. Repository Rules](#4-repository-rules)
- [Queries vs repositories](#queries-vs-repositories)
- [5. Application Layer](#5-application-layer)
- [6. DTOs and Boundary Types](#6-dtos-and-boundary-types)
- [7. Error Design](#7-error-design)
- [8. Authorization](#8-authorization)
- [9. Transactions and Unit of Work](#9-transactions-and-unit-of-work)
- [10. Infrastructure Layer](#10-infrastructure-layer)
- [11. Database Access](#11-database-access)
- [Migrations](#migrations)
- [12. HTTP / Server Framework Boundary](#12-http--server-framework-boundary)
- [13. SwiftUI Client Architecture](#13-swiftui-client-architecture)
- [14. Swift Concurrency](#14-swift-concurrency)
- [Sendability](#sendability)
- [Actors](#actors)
- [Cancellation](#cancellation)
- [15. Dependency Injection and Composition](#15-dependency-injection-and-composition)
- [16. Package and Module Boundaries](#16-package-and-module-boundaries)
- [17. Feature Organization](#17-feature-organization)
- [18. Testing Strategy](#18-testing-strategy)
- [Domain tests](#domain-tests)
- [Use-case tests](#use-case-tests)
- [Infrastructure tests](#infrastructure-tests)
- [End-to-end tests](#end-to-end-tests)
- [19. API Design](#19-api-design)
- [20. Observability](#20-observability)
- [21. Security Rules](#21-security-rules)
- [22. Architecture Decision Heuristics](#22-architecture-decision-heuristics)
- [Duplication](#duplication)
- [23. Codex Workflow](#23-codex-workflow)
- [24. New Feature Decision Flow](#24-new-feature-decision-flow)
- [25. Review Checklist](#25-review-checklist)
- [26. Default Architectural Stance](#26-default-architectural-stance)


These instructions apply to Swift projects unless a more specific `AGENTS.md` exists deeper in the repository.

## Goal

Build Swift systems that are easy to change, test, and reason about without introducing architectural ceremony before it is justified.

Prefer:

- explicit dependency direction
- domain-focused code
- small, composable types
- Swift Concurrency (`async`/`await`, `Sendable`, actors where state isolation is required)
- value types by default
- protocols at meaningful boundaries, not for every concrete type
- feature-oriented organization
- framework-independent business logic
- simple solutions that can evolve

Avoid:

- framework types leaking into domain logic
- "ViewModel/Service/Repository" layers created by reflex
- one protocol per type purely for mocking
- giant dependency containers
- global mutable state
- database models doubling as domain models
- DTOs added where no boundary exists
- premature microservices
- generic abstractions that make call sites harder to understand
- architecture whose primary purpose is satisfying a diagram

---

# 1. Architecture Selection

Do not force the full architecture onto every project.

Choose the smallest architecture that preserves important boundaries.

## Small feature or prototype

Use:

```text
Feature/
├── Model.swift
├── FeatureService.swift
└── FeatureView.swift        # client apps only
```

Keep state and logic local.

Introduce abstractions only when:

- a dependency is external or expensive
- multiple implementations are real requirements
- a boundary improves testability materially
- domain rules need isolation
- transactionality matters

## Medium application

Prefer feature modules or feature folders:

```text
Sources/
├── Posts/
│   ├── Domain/
│   ├── Application/
│   └── Infrastructure/
├── Users/
│   ├── Domain/
│   ├── Application/
│   └── Infrastructure/
└── App/
    └── Composition/
```

Do not organize the whole repository as:

```text
Models/
Views/
ViewModels/
Services/
Repositories/
```

unless the application is genuinely tiny.

## Large or long-lived application

Use domain/application/infrastructure boundaries where they provide value:

```text
Domain
   ↑
Application
   ↑
Infrastructure / Presentation / Framework integration
```

Dependencies point inward.

The domain must not import infrastructure frameworks.

The application layer may depend on domain abstractions.

Infrastructure implements interfaces required by the inner layers.

Composition is performed at the executable/application boundary.

---

# 2. Domain Layer

The domain contains business concepts and invariants.

Domain types are not:

- database rows
- ORM models
- HTTP request/response objects
- UI models
- JSON payloads

Prefer structs and enums.

Example:

```swift
public struct Post: Sendable, Equatable {
    public let id: ID
    public private(set) var title: String
    public private(set) var body: String

    public struct ID: RawRepresentable, Hashable, Sendable {
        public let rawValue: UUID

        public init(rawValue: UUID) {
            self.rawValue = rawValue
        }
    }

    public enum ValidationError: Error, Sendable {
        case emptyTitle
        case titleTooLong
    }

    public mutating func rename(to title: String) throws {
        try Self.validate(title: title)
        self.title = title
    }

    private static func validate(title: String) throws {
        guard !title.isEmpty else {
            throw ValidationError.emptyTitle
        }

        guard title.count <= 255 else {
            throw ValidationError.titleTooLong
        }
    }
}
```

## Domain invariants

Enforce invariants at construction or mutation boundaries.

Prefer invalid states being impossible or difficult to represent.

Do not rely exclusively on:

- route validation
- form validation
- database constraints
- UI validation

Those may duplicate domain validation for usability or defense in depth, but the domain remains authoritative for business rules.

## Creation types

When a persisted entity requires an identifier but new values do not yet have one, prefer a distinct creation type rather than an optional identifier:

```swift
extension Post {
    public struct New: Sendable {
        public let title: String
        public let body: String
    }
}
```

Use a validating factory when construction has meaningful invariants.

Do not create factories for trivial data containers without a reason.

## Domain identifiers

Prefer typed IDs over raw `String` or `UUID` when accidentally mixing identifiers would be dangerous.

Example:

```swift
struct UserID: Hashable, Sendable {
    let rawValue: UUID
}

struct PostID: Hashable, Sendable {
    let rawValue: UUID
}
```

---

# 3. Protocols and Ports

Protocols represent architectural boundaries.

Good protocol candidates:

- persistence needed by application logic
- clock/date source
- ID generation
- authorization
- external API clients
- email/payment/storage systems
- transactional execution
- platform services

Do not introduce a protocol simply because a concrete type exists.

Prefer narrow capability-oriented protocols.

Bad:

```swift
protocol ApplicationDependencies {
    var users: UserRepository { get }
    var posts: PostRepository { get }
    var mail: MailService { get }
    var payments: PaymentService { get }
    // dozens more...
}
```

Better:

```swift
struct PublishPostUseCase {
    let posts: any PostRepository
    let clock: any Clock
}
```

or a narrowly scoped unit-of-work dependency when several repositories must share the same transaction.

---

# 4. Repository Rules

A repository abstracts persistence of domain concepts.

Repository interfaces belong with the layer that requires them, normally Domain or Application.

Example:

```swift
public protocol PostRepository: Sendable {
    func insert(_ post: Post.New) async throws -> Post
    func find(id: Post.ID) async throws -> Post?
    func update(_ post: Post) async throws -> Post
    func delete(id: Post.ID) async throws
}
```

Repositories must not expose:

- Fluent models
- SQL rows
- database connections
- ORM query builders
- HTTP response objects

Infrastructure implementations may use any of these internally.

## Queries vs repositories

Do not force every read through a rich domain repository.

For reporting, projections, lists, dashboards, search, and optimized read models, a dedicated query abstraction is often clearer:

```swift
protocol PostQueries: Sendable {
    func listPublished(
        limit: Int,
        cursor: String?
    ) async throws -> [PostSummary]
}
```

Use repositories for domain persistence.

Use query objects for read-optimized projections when appropriate.

This is a pragmatic CQRS-style separation, not a requirement to adopt full CQRS.

---

# 5. Application Layer

The application layer coordinates use cases.

A use case represents business intent, not a CRUD table operation.

Prefer names such as:

- `PublishPost`
- `RegisterUser`
- `ApproveInvoice`
- `ChangePassword`
- `ArchiveProject`

over vague types such as:

- `PostManager`
- `UserService`
- `DataHandler`

Example:

```swift
public struct PublishPost: Sendable {
    private let posts: any PostRepository
    private let authorizer: any Authorizer

    public init(
        posts: any PostRepository,
        authorizer: any Authorizer
    ) {
        self.posts = posts
        self.authorizer = authorizer
    }

    public func execute(
        subject: Subject,
        input: Input
    ) async throws -> Output {
        try await authorizer.require(
            subject,
            permission: .publishPost
        )

        let newPost = try Post.make(
            title: input.title,
            body: input.body
        )

        let post = try await posts.insert(newPost)

        return Output(post: post)
    }
}
```

Keep use cases focused.

A use case may:

- authorize an action
- coordinate multiple domain objects
- execute transactional work
- call external ports
- map input/output at the application boundary

A use case should not:

- parse HTTP headers
- construct framework responses
- execute raw SQL
- know ORM types
- format SwiftUI views
- contain framework routing logic

---

# 6. DTOs and Boundary Types

Use DTOs when crossing a meaningful boundary.

Useful boundaries include:

- HTTP/API boundary
- public package API
- process boundary
- security boundary
- application-to-presentation boundary
- read projection boundary

Do not duplicate every domain type into a DTO automatically.

Create DTOs when they prevent coupling or accidental data exposure.

Example:

```swift
struct User {
    let id: UserID
    let passwordHash: String
}

struct UserDetails: Sendable {
    let id: UserID
}
```

Never expose sensitive domain state simply because returning the domain type is convenient.

---

# 7. Error Design

Model errors at the layer where their meaning is known.

Examples:

Domain:

```swift
enum PostValidationError: Error {
    case emptyTitle
}
```

Application:

```swift
enum PublishPostError: Error {
    case forbidden
    case invalidPost(PostValidationError)
}
```

Transport:

```text
PublishPostError.forbidden -> HTTP 403
PostValidationError.emptyTitle -> HTTP 422
```

Do not make domain code depend on HTTP status codes.

Translate errors at boundaries.

Typed throws may be used when they improve the API, but do not force typed errors through large dependency graphs if doing so adds substantial ceremony.

---

# 8. Authorization

Authentication and authorization are separate concerns.

Authentication answers:

> Who is the caller?

Authorization answers:

> May this caller perform this action?

Transport or middleware may authenticate a request and produce a domain/application-friendly subject:

```swift
struct Subject: Sendable {
    let userID: UserID
}
```

Application use cases should enforce business authorization when permission is part of the use case.

Do not rely exclusively on route visibility or UI state for authorization.

Prefer typed permissions where practical:

```swift
enum Permission: String, Sendable {
    case publishPost = "post:publish"
    case deletePost = "post:delete"
}
```

Avoid scattering permission string literals across the codebase.

---

# 9. Transactions and Unit of Work

Use a transaction when multiple writes must succeed or fail atomically.

Examples:

- create order + order items
- publish post + tag associations
- transfer balance between accounts
- create account + required profile state

Do not wrap every read in a transaction.

Keep transaction boundaries at the application/use-case level.

The use case decides *what must be atomic*.

Infrastructure decides *how the database transaction works*.

A simple abstraction is preferable to a highly generic executor unless the project genuinely needs more flexibility:

```swift
protocol TransactionRunner: Sendable {
    associatedtype Context: Sendable

    func run<T: Sendable>(
        _ operation: @Sendable (Context) async throws -> T
    ) async throws -> T
}
```

If several repositories must share a connection, compose a narrow transaction scope:

```swift
struct CreateOrderScope: Sendable {
    let orders: any OrderRepository
    let inventory: any InventoryRepository
}
```

Avoid a single mega-scope containing every repository in the application.

---

# 10. Infrastructure Layer

Infrastructure contains implementations of external concerns.

Examples:

- PostgreSQL
- SQLite
- Fluent
- PostgresNIO
- Redis
- filesystem
- S3-compatible storage
- APNs
- SMTP
- third-party APIs
- Vapor/Hummingbird adapters where appropriate

Infrastructure may depend on Domain/Application.

Domain/Application must not depend on infrastructure.

Example:

```swift
struct PostgresPostRepository: PostRepository {
    let connection: PostgresConnection

    func insert(_ post: Post.New) async throws -> Post {
        // SQL / driver-specific implementation
    }
}
```

Map database representations to domain representations explicitly.

Do not make the ORM entity the domain model unless the application is deliberately simple and accepts that coupling.

---

# 11. Database Access

Keep SQL and database-specific query code close to infrastructure.

A useful structure:

```text
Infrastructure/
└── Database/
    ├── PostTable.swift
    ├── PostgresPostRepository.swift
    ├── PostgresPostQueries.swift
    └── Migrations/
```

Separate:

```text
Database row
    ↓ mapping
Domain model
    ↓ mapping
Application DTO / API response
```

Do not reuse one struct for all three solely to reduce code.

Duplication at architectural boundaries is often cheaper than coupling.

However, avoid mechanically creating duplicate representations when fields and semantics are truly identical and no boundary benefit exists.

## Migrations

Treat migrations as production code.

Migrations should be:

- deterministic
- reviewable
- forward-compatible when possible
- safe for realistic production data volume

Never silently modify an already-shipped migration unless the project explicitly permits it.

---

# 12. HTTP / Server Framework Boundary

Vapor, Hummingbird, or another server framework should remain an adapter around application logic.

A route handler should usually do only:

```text
request
  -> authenticate
  -> decode/validate transport schema
  -> map to use-case input
  -> execute use case
  -> map output/error to HTTP response
```

Route handlers should be thin.

Avoid:

```swift
router.post("posts") { request in
    // 150 lines of auth
    // business validation
    // SQL
    // email calls
    // response formatting
}
```

Prefer:

```swift
router.post("posts") { request in
    let subject = try request.authenticatedSubject()
    let requestDTO = try await request.decode(CreatePostRequest.self)

    let result = try await publishPost.execute(
        subject: subject,
        input: requestDTO.applicationInput
    )

    return CreatePostResponse(result)
}
```

Transport types may conform to `Codable`.

Domain types should conform to `Codable` only when serialization is genuinely part of their domain/public contract.

---

# 13. SwiftUI Client Architecture

For Apple client targets, use SwiftUI's native data flow.

Prefer:

- `@State` for local ephemeral state
- `@Binding` for two-way child interaction
- `@Observable` for shared feature/application state on modern OS targets
- `@Environment` for dependency injection where appropriate
- `.task` for lifecycle-aware asynchronous work

Do not create a ViewModel for every view.

Views may own asynchronous loading state when the logic is simple.

Extract an `@Observable` model when state or behavior is shared, substantial, or independently meaningful.

Example:

```swift
@Observable
@MainActor
final class AccountSession {
    private(set) var user: User?

    func signIn(
        using authentication: AuthenticationClient
    ) async throws {
        user = try await authentication.signIn()
    }
}
```

Keep API/database framework DTOs outside core domain logic where practical.

---

# 14. Swift Concurrency

Use structured concurrency by default.

Prefer:

```swift
async/await
withThrowingTaskGroup
actor
@MainActor
Sendable
```

Avoid new Combine pipelines unless reactive streams are genuinely the right abstraction.

## Sendability

Types crossing task or actor boundaries should be `Sendable`.

Prefer immutable value types.

Do not add `@unchecked Sendable` merely to silence the compiler.

If `@unchecked Sendable` is unavoidable, document the synchronization invariant.

## Actors

Use actors for shared mutable state that requires serialization.

Do not turn stateless services into actors without a reason.

## Cancellation

Async operations should cooperate with cancellation.

Avoid detached tasks unless isolation from the current task is explicitly required.

Do not use `Task.detached` as an escape hatch for actor-isolation errors.

---

# 15. Dependency Injection and Composition

Prefer initializer injection.

```swift
struct RegisterUser {
    let users: any UserRepository
    let hasher: any PasswordHasher
    let clock: any Clock
}
```

Compose concrete dependencies at the outermost application boundary:

```text
main/App
    creates DB client
    creates repositories
    creates external clients
    creates use cases
    attaches handlers/views
```

This is the composition root.

Avoid service locators and hidden global dependency lookup.

Environment-based dependency injection is appropriate for SwiftUI presentation dependencies, but avoid turning `Environment` into a universal service locator for domain code.

---

# 16. Package and Module Boundaries

Do not split code into Swift packages simply to mimic architectural layers.

Create a package/module boundary when it provides real value:

- independent compilation
- reusable library
- strict dependency enforcement
- independent ownership
- substantial feature boundary
- build-time improvement
- platform separation

For many applications, folders + access control are sufficient.

If separate modules are justified, a possible dependency graph is:

```text
PostDomain
    ↑
PostApplication
    ↑
PostInfrastructure
    ↑
ServerApp
```

Never create circular target dependencies.

Use `package` access where it provides cleaner internal APIs across package targets.

Keep `public` APIs intentionally small.

---

# 17. Feature Organization

Prefer feature-first organization.

Example:

```text
Sources/
├── Posts/
│   ├── Domain/
│   │   ├── Post.swift
│   │   └── PostRepository.swift
│   ├── Application/
│   │   ├── PublishPost.swift
│   │   └── PostDetails.swift
│   └── Infrastructure/
│       └── Database/
│           ├── PostTable.swift
│           └── PostgresPostRepository.swift
│
├── Accounts/
│   └── ...
│
└── App/
    ├── HTTP/
    └── Composition/
```

Closely related small types may live in the same file.

Do not optimize for one-type-per-file dogma.

---

# 18. Testing Strategy

Test behavior at the cheapest useful layer.

## Domain tests

Test:

- invariants
- transitions
- edge cases
- value semantics

No database or network.

## Use-case tests

Test orchestration with lightweight fakes/stubs.

Prefer simple test doubles over mocking frameworks when possible.

Example:

```swift
actor InMemoryPostRepository: PostRepository {
    private var posts: [Post.ID: Post] = [:]

    // implementation...
}
```

## Infrastructure tests

Use integration tests for:

- SQL
- migrations
- repository mappings
- transactions
- external protocol adapters

Do not try to prove SQL correctness using repository mocks.

## End-to-end tests

Use selectively for critical application flows.

Avoid a testing pyramid made mostly of slow end-to-end tests.

---

# 19. API Design

Prefer concrete types until polymorphism is required.

Prefer generics when the relationship is static and performance/type preservation matter.

Prefer `any Protocol` for runtime polymorphism at architectural boundaries.

Use opaque `some Protocol` return types when callers should not know the concrete type and existential storage is unnecessary.

Avoid unnecessary type erasure.

Prefer domain-specific names and strong types.

Avoid Boolean parameters whose meaning is unclear:

Bad:

```swift
update(user, true, false)
```

Better:

```swift
update(
    user,
    sendNotification: true,
    invalidateSessions: false
)
```

Better still, model meaningful options if combinations matter.

---

# 20. Observability

Infrastructure/application boundaries should make production diagnostics possible.

Use structured logging.

Include meaningful contextual identifiers where safe:

- request ID
- trace ID
- user/account ID where policy permits
- job ID
- resource ID

Never log:

- passwords
- secrets
- authorization tokens
- private cryptographic material
- full sensitive request payloads

Metrics and tracing belong at infrastructure/application boundaries, not inside pure domain entities.

---

# 21. Security Rules

Treat all external input as untrusted.

Use:

- parameterized SQL/query APIs
- explicit authorization
- server-side validation
- safe secret management
- least privilege
- cryptographically secure APIs where security requires randomness

Never:

- build SQL using unescaped string interpolation
- trust client-supplied authorization state
- store plaintext passwords
- invent custom cryptography
- expose domain/private fields automatically in API encoders

Security-sensitive behavior should be explicit and tested.

---

# 22. Architecture Decision Heuristics

Before adding an abstraction, ask:

1. What coupling does this remove?
2. What change becomes easier?
3. What invariant does this enforce?
4. What test becomes materially simpler?
5. Is there a real second implementation or boundary?
6. Would a concrete type be clearer today?
7. Can the abstraction be added later without expensive migration?

If there is no convincing answer, prefer the simpler design.

## Duplication

Do not apply DRY mechanically across boundaries.

Repeated mapping code can be acceptable when it preserves independence between:

- domain
- persistence
- transport
- UI

Prefer duplication over the wrong abstraction.

Within one conceptual layer, remove obvious accidental duplication when doing so improves clarity.

---

# 23. Codex Workflow

When modifying an existing project:

1. Read the root `AGENTS.md`.
2. Read any more specific nested `AGENTS.md` relevant to files being changed.
3. Inspect existing architecture before proposing a new pattern.
4. Match established conventions unless they violate an explicit requirement or create a serious defect.
5. Make the smallest coherent change.
6. Do not refactor unrelated code.
7. Build after structural changes.
8. Run focused tests first.
9. Run the broader test suite when practical.
10. Report what was changed and what validation ran.

Before introducing a new architectural layer, explain through code structure why it is needed.

Do not rewrite a working architecture into Clean Architecture merely because these guidelines describe it.

Evolution is preferred over migration-for-style.

---

# 24. New Feature Decision Flow

For each feature, reason in this order:

```text
1. What business behavior is required?
        ↓
2. Which domain rules/invariants exist?
        ↓
3. Does this need a distinct use case?
        ↓
4. Which external capabilities are required?
        ↓
5. Which boundaries require protocols?
        ↓
6. Must any operations be atomic?
        ↓
7. What persistence/API adapters implement the ports?
        ↓
8. How is everything composed?
        ↓
9. What is the cheapest useful test strategy?
```

Do not start from database tables or framework controllers unless the work is purely infrastructural.

---

# 25. Review Checklist

Before considering a feature complete, verify:

- Business rules are not accidentally embedded only in UI/routes/database code.
- Dependency direction points toward domain/application code.
- Framework-specific types do not leak inward without a deliberate reason.
- Repository APIs speak in domain/application terminology.
- Transaction boundaries match business atomicity requirements.
- Authorization is enforced server-side/application-side where required.
- Sensitive fields do not leak through DTOs.
- Concurrency is safe under Swift 6 checking.
- `Sendable` violations are fixed rather than suppressed.
- No unnecessary protocol or layer was introduced.
- Tests cover important domain behavior.
- Persistence code has integration coverage when practical.
- Naming expresses business intent.
- Public/package access levels are intentional.
- The implementation is simpler than plausible alternatives while meeting the requirements.

---

# 26. Default Architectural Stance

Use clean/hexagonal/onion principles as constraints, not as a template generator.

The preferred default is:

```text
Domain
- business entities/value objects
- invariants
- domain errors
- required persistence/service ports when appropriate

Application
- use cases
- authorization orchestration
- transaction boundaries
- application DTOs/projections

Infrastructure
- database implementations
- external service adapters
- framework-specific implementations

Presentation / Transport
- SwiftUI views or HTTP handlers
- request/response mapping
- presentation state

Composition
- concrete dependency wiring
```

For simple features, collapse layers physically while preserving logical separation.

For complex features, make boundaries explicit.

Architecture exists to make change safer and reasoning easier, not to maximize the number of files.
