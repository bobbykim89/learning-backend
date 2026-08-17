# Phase 9 — Testing and Quality

Test strategy, integration testing, and static analysis.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Test strategy: pyramid vs trophy; what each level is actually for](test-strategy.md)
- [Test doubles: stub, mock, fake, spy — and how DI makes them unnecessary as often as possible](test-doubles.md)
- [Unit testing business logic with no I/O; designing for that](unit-testing-business-logic.md)
- [Integration testing against real dependencies with Testcontainers](integration-testing.md)
- [Database test strategy: transactional rollback, truncation, migrations in tests, factories](database-testing.md)
- [Deterministic tests: freezing time, seeded randomness, controlling concurrency](deterministic-tests.md)
- [Testing async code and background workers](testing-async-code.md)
- [Contract testing between services; consumer-driven contracts](contract-testing.md)
- [Property-based testing: Hypothesis (Python), fast-check (TS)](property-based-testing.md)
- [Snapshot and approval testing — the good and the trap](snapshot-testing.md)
- [Mutation testing to measure suite quality](mutation-testing.md)
- [Load, soak, spike, and stress tests as part of the pipeline](performance-tests-in-ci.md)
- [Static analysis: mypy/pyright strict, TypeScript strict mode, ruff, ESLint](static-analysis.md)
- [Making illegal states unrepresentable with the type system](types-for-correctness.md)
- [Test flakiness: causes, quarantine, and fixing rather than retrying](test-flakiness.md)
