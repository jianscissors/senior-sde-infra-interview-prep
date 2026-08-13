# Design a Secrets Management System (like a simplified Vault)

## Clarifying questions to ask first
- What kinds of secrets: static (API keys, TLS certs) only, or also dynamic/short-lived credentials (database passwords, cloud IAM tokens)?
- What's the access control model — per-service identity, per-team, or both?
- Is audit logging a hard requirement (usually yes, for anything touching secrets)?
- How do services authenticate to the secrets store in the first place (the bootstrapping problem)?

## Requirements
### Functional
- Store secrets encrypted; allow authorized clients to read them.
- Support secret rotation, and revocation of a specific secret or a specific client's access.
- Log every access for audit purposes.

### Non-functional
- Secrets must be encrypted both at rest and in transit — no plaintext secret ever touches disk unencrypted or crosses the network unencrypted.
- Least-privilege access: a service should only be able to read the specific secrets it needs, not the entire store.
- High availability — this is a hard dependency for every other service's startup path, so its own downtime has an outsized blast radius.

## High-level architecture
1. **Storage backend**: secrets stored encrypted at rest, typically using envelope encryption — each secret is encrypted with a per-secret data key, and that data key is itself encrypted with a master key held in a separate key management system (KMS) or hardware security module (HSM).
2. **Access control layer**: authenticates the requesting service's identity, checks a policy (which paths/secrets this identity may read), and only then decrypts and returns the secret.
3. **Audit log**: every read/write/auth attempt (success or failure) is appended to an immutable, separately-secured log.
4. **Dynamic secrets engine** (optional but expected at senior level): generates short-lived credentials on demand (e.g., a database username/password valid for 1 hour) rather than storing a long-lived static credential.

## Deep dives

### Encryption at rest and in transit
- **In transit**: all client↔server communication over mTLS, so both sides authenticate each other and the payload is encrypted end to end.
- **At rest**: envelope encryption — a root/master key (held in an HSM or cloud KMS, never directly accessible to the secrets service's own storage) encrypts per-secret (or per-shard) data encryption keys, which in turn encrypt the actual secret values. This limits blast radius: compromising the storage layer alone doesn't expose plaintext secrets without also compromising the KMS.

### Access control (least privilege)
- Policies are attached to a client's authenticated identity (not to a shared static token), scoped to specific secret paths, and support fine-grained actions (read-only vs read-write vs list). A compromised single service's identity should only expose the secrets that service legitimately needs — this is what limits blast radius when (not if) one service gets compromised.

### Dynamic/short-lived credentials vs static secrets
- Static secrets (a database password stored once, used indefinitely) are a standing liability — if leaked, they're valid until someone notices and manually rotates them. Dynamic secrets are generated on-demand per client, scoped to a short TTL, and auto-expire/auto-revoke — a leaked dynamic credential has a small, bounded window of usefulness. This is the single biggest practical security improvement a secrets manager offers over "just encrypt a config file," and interviewers are usually listening for you to bring it up unprompted.

### Rotation without downtime
- Rotating a secret must not cause a thundering-herd of failures for services mid-use of the old value. Standard pattern: support **dual validity** during a rotation window — both the old and new secret/credential are valid simultaneously for a bounded overlap period, giving every consumer time to pick up the new value before the old one is revoked.

### Audit logging
- Every access attempt — success and failure — is logged with requester identity, secret path, timestamp, and outcome, to an append-only, tamper-evident log stored separately from the secrets themselves (so compromising the secrets store doesn't also let an attacker erase their tracks).

### The bootstrapping problem
- A service needs to authenticate to the secrets store before it can fetch its first secret — but how does it prove its identity without already holding a secret? The standard answer is to piggyback on a platform-provided identity that already exists independent of this system: a cloud IAM role attached to the compute instance, or a Kubernetes service account token, both of which the underlying platform issues and verifies without the service needing to pre-provision a credential itself. The secrets service trusts the platform's identity assertion instead of requiring a bootstrap secret.

## Key tradeoffs
- Dynamic secrets are much safer but add complexity (an engine that must know how to create/revoke credentials in every downstream system — databases, cloud providers, etc.) and add a dependency on the secrets service being available at request time, not just at deploy time.
- Fine-grained per-path policies are more secure but harder to manage at scale than coarser role-based grants; most systems land on a hybrid (roles composed of path patterns).

## Failure modes
- Secrets service is down at a service's startup: services that need secrets to boot will fail to start — mitigate with a short-lived local cache of previously-fetched secrets (encrypted at rest locally) so a restart during a brief outage doesn't cascade into a wider outage, while still expiring that cache aggressively.
- KMS/HSM (the root of trust) is unavailable: nothing can be decrypted at all — this is intentionally the hardest dependency to make highly available, since it's also the highest-value target; cloud KMS services typically offer very high published availability specifically because so many systems have this exact single point of trust.

## Likely follow-ups
- "How do you revoke a specific service's access immediately, company-wide?" → revoke its identity/policy centrally; for dynamic secrets already issued, actively invalidate them (not just let them expire) if the credential store/database supports out-of-band revocation.
- "How would you handle secret sprawl — secrets accidentally committed to source control or logged?" → that's a detection/scanning problem layered on top (secret-scanning in CI, log scrubbing), not something the storage system itself prevents; worth mentioning as a complementary control.
