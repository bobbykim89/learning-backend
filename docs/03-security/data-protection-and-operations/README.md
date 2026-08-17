# Data protection and operations

Files are created when the topic is studied — see [CLAUDE.md](../../../CLAUDE.md).

- [Secrets management: Vault, cloud secret managers; never in env files in git](secrets-management.md)
- [Encryption at rest vs in transit vs application-level; envelope encryption, KMS, key rotation](encryption-and-key-management.md)
- [PII handling, data classification, tokenization, log redaction](pii-and-data-classification.md)
- [Rate limiting and abuse prevention as security controls](rate-limiting-as-security.md)
- [Supply chain: lockfiles, integrity hashes, SBOM, `npm audit`/`pip-audit`, Dependabot, typosquatting](supply-chain-security.md)
- [Container and runtime hardening: non-root, read-only FS, minimal images](container-hardening.md)
- [Security headers and HTTPS enforcement (HSTS, CSP as a backend concern)](security-headers.md)
- [Audit logging: what to record, tamper-resistance, retention](audit-logging.md)
- [Compliance orientation: GDPR/CCPA basics, SOC 2 concepts, data residency, right to erasure](compliance-basics.md)
- [Responsible disclosure, CVE monitoring, patch cadence](disclosure-and-patch-cadence.md)

> `rate-limiting-as-security.md` covers the control; the algorithms and their
> distributed coordination live in
> [Phase 10 — Rate limiting algorithms](../../10-scaling/rate-limiting-algorithms.md)
> and [Distributed rate limiting](../../10-scaling/distributed-rate-limiting.md).
