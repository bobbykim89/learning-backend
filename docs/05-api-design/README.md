# Phase 5 — API Design and Contracts

Resources, pagination, versioning, realtime, and webhooks.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Resource modeling, URL design, and Richardson maturity levels](resource-modeling.md)
- [Pagination: offset vs keyset/cursor, stable ordering, total counts](pagination.md)
- [Filtering, sorting, sparse fieldsets, expansion — without reinventing GraphQL badly](filtering-and-field-selection.md)
- [Versioning strategies: URL, header, media type; deprecation policy and sunset headers](api-versioning.md)
- [Idempotency for unsafe methods; idempotency keys end-to-end](api-idempotency.md)
- [Bulk and batch endpoints; partial success semantics](bulk-endpoints.md)
- [Long-running operations: 202 + status resource, polling vs callbacks](long-running-operations.md)
- [OpenAPI-first design, codegen for clients and servers, contract testing (Pact)](openapi-first-design.md)
- [Framework comparison: FastAPI, Litestar, Django REST vs Express, Fastify, NestJS, Hono](framework-comparison.md)
- [GraphQL: schema design, resolvers, N+1 and DataLoader, pagination conventions](graphql-basics.md)
- [GraphQL in production: depth/complexity limits, persisted queries, caching difficulties, federation](graphql-in-production.md)
- [gRPC and Protobuf: streaming modes, schema evolution, when RPC beats REST](grpc-and-protobuf.md)
- [tRPC and type-safe RPC in a TypeScript monorepo](trpc-and-type-safe-rpc.md)
- [Realtime: polling vs long polling vs SSE vs WebSockets — tradeoffs and scaling each](realtime-transports.md)
- [WebSocket production concerns: auth, heartbeats, reconnection, fan-out via pub/sub, backpressure](websockets-in-production.md)
- [Webhooks you _send_: signing (HMAC), retries, ordering, replay protection, subscriber management](sending-webhooks.md)
- [Webhooks you _receive_: verification, idempotency, fast-ack + async processing](receiving-webhooks.md)
- [File handling: presigned URLs, multipart and resumable uploads, streaming, content validation](file-uploads-and-downloads.md)
- [API gateways: routing, auth offload, rate limiting, request transformation](api-gateways.md)
- [BFF pattern — where your frontend experience gives you an edge](bff-pattern.md)

> `api-idempotency.md` covers the request-handling contract; the storage-side
> treatment lives in [Phase 2 — Idempotency keys](../02-data/modeling-patterns/idempotency-keys.md).
