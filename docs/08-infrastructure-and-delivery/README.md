# Phase 8 — Infrastructure and Delivery

Containers, orchestration, IaC, and CI/CD.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Containers: images, layers, caching, multi-stage builds for Python and Node](container-images.md)
- [Image hygiene: slim/distroless bases, non-root users, `.dockerignore`, reproducible builds](image-hygiene.md)
- [Docker Compose for local development that mirrors production](docker-compose-local-dev.md)
- [Orchestration concepts: pods, deployments, services, ingress, configmaps, secrets](kubernetes-concepts.md)
- [Resource requests/limits, autoscaling (HPA), OOM kills, CPU throttling](resource-limits-and-autoscaling.md)
- [Simpler alternatives: ECS/Fargate, Fly.io, Render, Railway — and when they're the right call](managed-platforms.md)
- [Serverless: cold starts, execution limits, connection pooling problems, cost model](serverless.md)
- [Load balancers L4 vs L7, reverse proxies (Nginx, Caddy, Envoy), service mesh overview](load-balancers-and-proxies.md)
- [Networking: VPCs, subnets, security groups, private links, egress control](cloud-networking.md)
- [DNS operationally: TTLs, failover, blue/green cutover](dns-operations.md)
- [Infrastructure as Code: Terraform or Pulumi, state management, drift](infrastructure-as-code.md)
- [CI/CD pipelines: build once, promote artifacts, environment parity](cicd-pipelines.md)
- [Deployment strategies: rolling, blue/green, canary, progressive delivery with flags](deployment-strategies.md)
- [Ephemeral preview environments and seeded test data](preview-environments.md)
- [Secrets in CI, OIDC-based cloud auth, least-privilege pipelines](ci-secrets-and-oidc.md)
- [Cost engineering: egress charges, storage tiers, right-sizing, the cost of idle capacity](cost-engineering.md)
