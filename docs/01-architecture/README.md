# Phase 1 — Application Architecture and Code Design

Where dependency injection, testability, and "how do I structure this" live.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Layered architecture: transport → service/use-case → repository → data source](layered-architecture.md)
- [Why the layers exist: dependency direction, and what breaks when you skip them](why-layers-exist.md)
- [Inversion of Control and Dependency Injection — the concept, independent of frameworks](inversion-of-control-and-di.md)
- [Manual/constructor DI in plain Python and plain TypeScript (start here, no library)](manual-di.md)
- [DI in practice: FastAPI `Depends`, `dependency-injector`; NestJS, tsyringe, InversifyJS](di-frameworks.md)
- [Service lifetimes: singleton vs scoped vs transient, and request-scoped state](service-lifetimes.md)
- [DI tradeoffs: when a container is overkill, magic vs explicitness, startup cost](di-tradeoffs.md)
- [Ports and adapters (hexagonal architecture); when it pays off and when it's ceremony](ports-and-adapters.md)
- [SOLID, pragmatically — which principles matter daily, which are academic](solid-pragmatically.md)
- [Composition over inheritance; protocols/structural typing (Python `Protocol`, TS interfaces)](composition-and-protocols.md)
- [Domain modeling: entities, value objects, aggregates, invariants](domain-modeling.md)
- [DTOs vs domain models vs DB rows — why collapsing them hurts later](dtos-vs-domain-models.md)
- [Validation at the boundary: Pydantic v2, Zod — parse, don't validate](boundary-validation.md)
- [Error design: exception hierarchies vs result types, expected vs exceptional failures](error-design.md)
- [Error taxonomy for APIs: client vs server, retryable vs terminal, RFC 9457 problem details](api-error-taxonomy.md)
- [Configuration and the 12-factor app; typed config, fail-fast on startup](configuration-and-12-factor.md)
- [Feature flags as an architectural tool](feature-flags.md)
- [Project structure: layer-first vs feature-first, the modular monolith](project-structure.md)
- [When to split a service — and the strong case for _not_ doing microservices yet](when-to-split-a-service.md)
- [Middleware/interceptor pipelines: cross-cutting concerns done right](middleware-pipelines.md)
- [Writing an Architecture Decision Record (ADR) — capturing tradeoffs, not just choices](architecture-decision-records.md)
