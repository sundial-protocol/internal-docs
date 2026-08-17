# Threat Model (STRIDE) - Midgard Node

## Purpose

This document threat-models the [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) architecture using STRIDE:
Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service,
and Elevation of Privilege.

The model is intentionally network-independent. The trust boundaries analyzed
here are the stable node architecture:

```text
client / API ingress -> Redis Streams -> tx-processor -> Postgres/MPT state ->
sequencer fibers -> L1 provider -> Cardano L1
```

The same boundaries exist whether `NETWORK` is `Preprod`, `Preview`, `Custom`,
or `Mainnet`; the mainnet-specific difference is impact, not architecture.
That is why this review belongs before mainnet launch.

## Scope

In scope:

- [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) HTTP ingress, especially `POST /submit` and operational
  endpoints served by `listen.ts`.
- Redis Streams transaction ingress queue.
- `tx-processor` role and mempool acceptance path.
- Sequencer fibers: user-event sync, block commitment, block submission, and
  merge.
- Postgres projections and on-disk Merkle-Patricia-Trie state as they affect
  the transaction pipeline.
- L1 provider interaction through Lucid using Blockfrost or Kupo/Ogmios.
- AWS deployment boundaries where they protect the above path.

Out of scope:

- Full validator-level protocol correctness and fund-recovery semantics.
- Third-party L1 provider internal security.
- End-user wallet security.
- Formal verification of Aiken/Plutus validators.
- [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk) client threat modeling except where its output crosses the
  node ingress boundary.

Related documents:

- [Security Best Practices](../security.md)
- [Midgard Node API](../api.md)
- [Mainnet Readiness Review](../mainnet-readiness.md)
- [Observability](../observability.md)
- [Smart Contract Interactions](../smart-contracts.md)

## System Overview

The node can run as a monolith (`NODE_ROLE=all`) or split into role-specific
processes:

| Role | Main responsibility | Key dependencies |
| --- | --- | --- |
| `api` | HTTP ingress, health endpoints, submit queue producer | Redis, Postgres for health/read routes, L1 provider for readiness/balance/commit routes |
| `tx-processor` | Redis consumer-group processing, CBOR parse, mempool validation and insert | Redis, Postgres |
| `sequencer` | User-event sync, block commitment, block submission, merge | Postgres, MPT storage, L1 provider, operator keys |
| `all` | All of the above in one process | All dependencies |

Primary data flow:

1. A client sends raw hex CBOR to `POST /submit`.
2. The raw submit interceptor validates only that the body is hex and appends
   it to a Redis Stream.
3. A tx processor consumes entries via a Redis consumer group, reclaims stale
   pending entries, parses CBOR in worker threads, validates transactions, and
   inserts accepted transactions into `MempoolDB` and related projection tables.
4. The sequencer periodically fetches L1 user events, builds commitment
   windows, applies events to the ledger/MPT, stores unsubmitted block
   commitments, submits commitment transactions through the configured L1
   provider, and marks block/projection state forward.
5. Merge logic submits L1 state-queue merge transactions as confirmed blocks
   become available.

## Security Objectives

- Only valid, network-appropriate L2 transactions become accepted mempool
  entries.
- A malformed or hostile submission cannot corrupt queue, database, or ledger
  state.
- Accepted transactions and L1 events are applied once, in deterministic order,
  with durable recovery across process crashes.
- Operator signing keys and provider credentials remain confidential.
- Public ingress cannot trigger privileged operator actions unless explicitly
  authorized by deployment controls.
- Dependency failure or overload degrades safely and observably.
- Logs, metrics, and persisted records are sufficient to reconstruct important
  state transitions.

## Assets

| Asset | Why it matters |
| --- | --- |
| Operator seed phrases | Can sign L1 commitment, merge, and operator transactions. |
| L1 provider API keys / URLs | Control access to chain data and submission path. |
| Redis Stream and dead-letter stream | Durable ingress queue and retry source of truth before mempool acceptance. |
| Postgres tables | Authoritative off-chain projection for mempool, blocks, ledger state, user events, and history. |
| MPT on disk / EFS | Ledger root material used for block commitments. |
| Signed L1 commitment CBOR | Can be submitted or replayed to L1; must remain consistent with block status. |
| Health, metrics, logs, traces | Operational evidence and incident response surface. |
| Public HTTP endpoint | Main attacker entry point. |

## Trust Boundaries

| Boundary | Crosses from | Crosses to | Trust change |
| --- | --- | --- | --- |
| B1 API ingress | Internet / client / reverse proxy | Node HTTP server | Untrusted caller enters application process. |
| B2 Submit queue | API process | Redis Streams | Application accepts durable work item. |
| B3 Queue processing | Redis | tx-processor worker pool | Durable bytes become parsed transaction semantics. |
| B4 Mempool persistence | tx-processor | Postgres + MempoolLedgerDB | Validated work mutates durable state. |
| B5 Sequencer state | sequencer | Postgres + MPT | Off-chain state becomes committed block material. |
| B6 L1 provider | sequencer / Lucid | Blockfrost or Kupo/Ogmios | Internal state depends on external chain data/submission response. |
| B7 L1 chain | L1 provider | Cardano network | Signed commitment becomes public chain state. |
| B8 Ops plane | operators / CI / Terraform | AWS ECS/RDS/Redis/EFS/Secrets | Humans and automation can change runtime, secrets, and persistence. |

## STRIDE Analysis

### Spoofing

| ID | Threat | Impact | Existing controls | Mainnet control requirement |
| --- | --- | --- | --- | --- |
| S-1 | Anonymous caller submits transactions as any client. | Queue/mempool can be filled by unauthenticated actors. | Submitted transactions still need CBOR parsing and Phase A/B validation before mempool acceptance. | Put API gateway/WAF/reverse proxy authentication and rate limits in front of public deployments. Decide whether public anonymous `POST /submit` is intentional for mainnet. |
| S-2 | Anonymous caller invokes side-effecting operator endpoints. | Unauthorized commitment/merge/reset/init-style actions or operational disclosure. | Split `NODE_ROLE=api` removes many full-router routes, but still serves `GET /commit`, root diagnostics, and commitment-wallet balance. | Before mainnet, require authz at ALB/API gateway for all side-effecting and operator-observability endpoints. Prefer not serving `GET /commit` on public ingress. |
| S-3 | Malicious or misconfigured L1 provider impersonates chain state. | Node may commit against stale or false view, miss events, or mis-handle idempotent submission recovery. | Provider readiness check queries protocol parameters; submit path re-checks produced UTxOs after idempotent-looking errors. | Use authenticated provider endpoints, TLS, provider allowlisting, monitoring for provider lag, and an operator runbook for provider failover. Consider cross-checking critical chain reads against a second provider before mainnet. |
| S-4 | Compromised ECS task or IAM principal retrieves secrets as the node. | Operator keys and provider credentials compromised. | AWS deployment injects secrets through Secrets Manager, not plain task environment; task execution role is scoped to enumerated secret ARNs. | Separate mainnet account or tighter IAM boundary; least-privilege human access to Secrets Manager; secret access alerts; documented key rotation/break-glass process. |

### Tampering

| ID | Threat | Impact | Existing controls | Mainnet control requirement |
| --- | --- | --- | --- | --- |
| T-1 | Client tampers with submit payload. | Invalid or hostile CBOR reaches queue. | Submit path requires hex; tx processor parses CBOR in worker pool; malformed CBOR is retried/dead-lettered without crashing the whole processor. | Add ingress body-size limits before the node; track malformed/dead-letter rates with alerts. |
| T-2 | Queue entry is altered or manually injected in Redis. | Processor accepts work that bypassed HTTP ingress controls. | Processor treats every queue entry as untrusted and repeats parsing/validation before persistence. | Restrict Redis network access to node roles only; require TLS/auth where managed Redis supports it; avoid exposing Redis in local/public deployments. |
| T-3 | Duplicate or conflicting transaction IDs race during mempool insert. | Ledger projection inconsistency or double application. | `validateAndInsertMultiple` runs inside a SQL transaction, takes a Postgres advisory transaction lock, canonicalizes duplicates by tx hash, and rejects conflicting CBOR for identical tx IDs. | Alert on duplicate-conflict rejection reasons; keep advisory-lock coverage in regression tests. |
| T-4 | Tampered Postgres or MPT state affects block commitments. | Sequencer may commit an incorrect ledger root. | Commitment work is persisted through `BlocksDB`; submission DB updates are transactional; recovery reconciles `SUBMITTING` blocks on restart. | Enforce backups, immutable/audited DB access, production DB access logging, and periodic consistency checks between Postgres projections, MPT root, and L1 state queue. |
| T-5 | Signed L1 commitment CBOR is modified between build and submit. | Invalid signature, invalid block commitment, or submission of unintended transaction. | Submission path completes/signs persisted CBOR, persists newly signed artifacts, uses L1 pre-check for already-landed transactions, and handles idempotent submit errors. | Limit direct DB write access; hash/signature audit of stored L1 CBOR; alert on repeated sign/submit failures. |
| T-6 | Terraform/runtime config is changed to unsafe network or provider values. | Mainnet node may point at wrong provider, wrong addresses, or wrong secrets. | Config validates required production values and positive/non-negative operational knobs; `NETWORK=Mainnet` disables genesis UTxOs. | Require reviewed plan/apply, environment-specific tfvars/backends, config-drift detection, and a preflight that confirms network/provider/script address expectations. |

### Repudiation

| ID | Threat | Impact | Existing controls | Mainnet control requirement |
| --- | --- | --- | --- | --- |
| R-1 | Submitter denies submitting a transaction. | Hard to attribute abusive traffic or support disputes. | Redis returns stream entry IDs; metrics count accepted/rejected submissions. | Preserve ALB access logs with client identity/proxy metadata; include request IDs/correlation IDs from ingress through Redis message metadata. |
| R-2 | Operator action cannot be tied to a principal. | Unauthorized manual `commit`, `merge`, `reset`, or repair is hard to investigate. | Application logs endpoint activity, but no application authentication identifies callers. | Put authenticated control plane in front of operator endpoints; log authenticated principal, source IP, and request ID. |
| R-3 | Queue processing failure lacks enough evidence. | Dead-letter triage is slow; false positives may be hard to distinguish from attack traffic. | Dead-letter stream stores original stream, original message id, CBOR, delivery count, failure time, and reason. | Add retention/export policy for dead-letter entries; alert on reason categories and preserve samples for incident review. |
| R-4 | L1 submit ambiguity after timeout. | Operator may not know whether a commitment landed or should be retried. | Submission marks blocks `SUBMITTING`, reverts on restart, checks produced UTxOs on L1, and treats known spent-input errors as idempotent-success candidates only after re-check. | Document operational decision tree and alert on blocks stuck in `UNSUBMITTED`/`SUBMITTING`/submit-failure loops. |

### Information Disclosure

| ID | Threat | Impact | Existing controls | Mainnet control requirement |
| --- | --- | --- | --- | --- |
| I-1 | Public API exposes ledger, block, state-queue, or wallet information. | Transaction history, internal state, and operator wallet balance may be visible. | `NODE_ROLE=api` excludes several read/debug routes, but still exposes root diagnostics and commitment-wallet balance. | Decide what public observability is acceptable; remove or protect balance/diagnostic endpoints on public hosts. |
| I-2 | Secrets leak through environment, logs, traces, or crash output. | Operator key/provider credential compromise. | AWS uses Secrets Manager injection; config errors report field names and placeholders rather than secret values in several paths. | Review logs/traces for accidental request-body/secret capture; redact sensitive env names/values; restrict CloudWatch/Loki/Grafana access. |
| I-3 | Grafana/metrics/log endpoints reveal system state. | Attackers can tune DoS or infer operator balances, queue depth, failures. | AWS docs describe observability plane as internal; local Compose exposes observability ports for development. | Ensure production observability is not public or is authenticated; avoid anonymous Grafana on mainnet unless explicitly approved. |
| I-4 | `POST /submit` body appears in logs or traces. | User transaction details can leak before inclusion. | Current submit path does not intentionally log full CBOR on success. | Verify reverse proxy, WAF, tracing, and error logging do not capture raw request bodies by default. |

### Denial of Service

| ID | Threat | Impact | Existing controls | Mainnet control requirement |
| --- | --- | --- | --- | --- |
| D-1 | Large or high-rate submit bodies exhaust API memory, Redis, or network. | API process or Redis becomes unavailable. | Hex check rejects non-hex; queue depth/pending/lag metrics exist. | Enforce request body-size limits and rate limits before the node; configure Redis memory/backpressure policy; alert on stream lag/depth. |
| D-2 | Malformed CBOR flood consumes worker pool and retry budget. | Tx processors spend CPU parsing bad inputs; dead-letter stream grows. | Worker-thread parsing isolates parsing work; malformed messages are retried up to `TX_QUEUE_MAX_DELIVERY_ATTEMPTS` then dead-lettered. | Tune max delivery attempts for mainnet; alert on malformed/dead-letter rates; consider early CBOR-size/shape screening at ingress. |
| D-3 | Valid but heavy transaction batches stress Phase A/B validation and DB writes. | Mempool lag, high DB load, delayed commitments. | Drain batch size, parse concurrency, consumer worker count, SQL chunking, and advisory locking are configurable. | Load/soak test mainnet-shaped limits; cap per-request size; monitor DB latency and tx acceptance duration. |
| D-4 | Redis outage or data loss interrupts ingress. | Accepted submissions may not be processed, or new submissions fail. | `/health/ready` checks Redis; local Redis uses AOF; processor recreates missing consumer group. | For AWS ElastiCache, decide snapshot/replication/Multi-AZ posture; document RPO/RTO for queued submissions. |
| D-5 | L1 provider slowness or outage blocks readiness, user-event sync, or submission. | Commitments and merges stall. | Readiness probes are timeout-bounded; Lucid init retries with capped/jittered backoff; submit/sign stages have timeouts and recovery. | Provider failover plan, provider SLO alerts, and circuit-breaker/backoff review for all high-volume chain reads. |
| D-6 | Sequencer commits faster than L1 submission can drain. | Unsubmitted backlog grows; stale signed commitments accumulate. | `COMMITMENT_MAX_UNSUBMITTED_BLOCK_BACKLOG` backpressure skips commitment cycles; backlog metrics exist. | Alert on backlog and skipped cycles; tune commitment windows; rehearse recovery for prolonged L1 outage. |
| D-7 | Database or MPT storage failure blocks processing. | Mempool/commitment/submission state cannot progress. | Readiness checks database; SQL transactions keep multi-table changes atomic; MPT recovery is covered in security docs. | Production backups, restore drills, EFS/RDS alarms, and periodic projection/MPT consistency checks. |

### Elevation of Privilege

| ID | Threat | Impact | Existing controls | Mainnet control requirement |
| --- | --- | --- | --- | --- |
| E-1 | Public caller reaches operator-only behavior through HTTP routes. | Caller influences commitment timing or recovery operations. | Split router removes `init`, `merge`, `reset`, repair, and debug routes from `NODE_ROLE=api`; full router remains broad. | Treat all side-effecting routes as privileged; enforce authz externally or remove them from public/API role before mainnet. |
| E-2 | Compromised `api` role pivots to sequencer privileges through shared credentials/config. | Ingress compromise becomes signing compromise. | Roles can be split into separate processes, but current AWS task definition is a single node task shape. | Mainnet runs split roles with distinct task definitions, IAM, secrets, and network permissions; `api` does not receive operator seed phrases unless its code path requires them. |
| E-3 | Tx processor compromise writes arbitrary accepted state. | Invalid mempool/ledger projection feeds sequencer. | Processor validates before persistence; DB transaction/advisory lock narrows race conditions. | Use DB users with role-specific privileges if feasible; add audit logging for writes to critical tables; isolate processor from operator signing secrets. |
| E-4 | CI/deployment principal changes image or secrets. | Malicious code executes with production credentials. | Terraform and CI practices are documented; AWS Secrets Manager and ECR image variables are explicit. | Enforce protected branches/tags, image digest pinning or signed images, least-privilege deploy role, and mandatory review for mainnet applies. |

## Boundary-Specific Notes

### B1: API ingress

The HTTP surface is the highest-risk external boundary. The code has useful
input validation for hashes, addresses, pagination, and raw submit hex, but it
does not provide general authentication, authorization, rate limiting, or body
limits. The split `api` router reduces exposure but still contains privileged
or operator-sensitive routes (`GET /commit`, root-unit diagnostics, and
commitment-wallet balance).

Mainnet determination: public ingress is constrained to intended public routes
only. Every side-effecting endpoint is removed from the public role or
protected by an authenticated operator control plane.

### B2-B4: Redis to tx processor to mempool

The queue boundary is designed as at-least-once delivery, not exactly-once.
This is appropriate for transaction CBOR because tx hashes make duplicate
submissions naturally idempotent when validation and persistence are correct.
Redis entries are still untrusted input: all queue items, even ones written by
the API process, must be parsed and validated before DB mutation.

Important existing controls:

- Redis consumer group with stale pending reclaim.
- Delivery-attempt cap and dead-letter stream.
- Worker-thread CBOR parse isolation.
- Phase A/B validation before mempool acceptance.
- SQL transaction plus advisory transaction lock around validation/insert.
- Duplicate tx hash canonicalization and conflicting-CBOR rejection.

Mainnet control state at this boundary requires ingress body and rate limits,
Redis isolation, queue-depth alerting, and dead-letter review.

### B5: Sequencer state

The sequencer converts durable off-chain state into L1 commitment material. Its
main risks are state corruption, non-deterministic ordering, and partial
progress after crashes. Existing controls include SQL transactions for
submission-side projection updates, block status transitions, commitment
backpressure, MPT-backed roots, and startup reconciliation of `SUBMITTING`
blocks back to `UNSUBMITTED`.

Mainnet control state at this boundary requires alerts for stuck statuses,
periodic Postgres/MPT/L1 consistency checks, and tested restore procedures.

### B6-B7: L1 provider and Cardano L1

The node trusts the configured provider for chain reads and submission. The
implementation treats provider failures as expected operational conditions in
several places: readiness probes are bounded, Lucid initialization backs off,
sign/submit have timeouts, and ambiguous submit failures are resolved through
L1 UTxO pre-checks.

Mainnet control state at this boundary requires provider failover and a
documented determination on whether critical chain reads are independently
cross-checked.

### B8: Operations plane

The operations plane can override every application-level control by changing
images, task definitions, secrets, network rules, database state, or Terraform
state. Existing AWS design uses private subnets, security groups, Secrets
Manager, encrypted storage/logging, ALB access logs, and no bastion SSH model.

Mainnet control state at this boundary requires separation: isolated account or
IAM boundary, protected deployment roles, reviewed plans, and split
role-specific tasks so the public API process does not carry sequencer signing
authority.

## Mainnet Threat-Model Determination

Sundial has a documented node threat model with defined trust boundaries,
existing controls, and explicit mainnet control requirements.

Public mainnet exposure remains blocked until the following conditions are
implemented:

- Put authentication/authorization in front of all side-effecting and
  operator-observability endpoints, or remove those routes from the public
  ingress role.
- Enforce request body-size limits and rate limits before `POST /submit`.
- Run split roles with least-privilege runtime secrets; the `api` and
  `tx-processor` roles do not receive operator signing seeds unless their code
  path requires them.
- Confirm production Redis/RDS/EFS durability and restore posture, including
  queue RPO/RTO.
- Define and test L1 provider failover.
- Ensure production Grafana, metrics, logs, and traces are authenticated or
  internal-only.
- Review mainnet Terraform plan and runtime config with a second engineer.

Additional mainnet control requirements:

- Add request/correlation IDs from ALB/API ingress into Redis metadata and
  processor logs.
- Alert on queue depth, pending count, consumer lag, dead-letter count,
  malformed-CBOR rejection rate, unsubmitted block backlog, submit failures,
  sign timeouts, and provider readiness failures.
- Add periodic consistency checks across Postgres projections, MPT roots, and
  L1 state queue.
- Preserve dead-letter samples and ALB access logs under an incident-response
  retention policy.
- Pin or sign production images and deploy by immutable digest or controlled tag
  promotion.

## Residual Risk

Even with the mitigations above, Midgard node remains an off-chain sequencer
system holding operator signing authority. A compromise of the sequencer
runtime, production deployment principal, or operator seed material is a
high-impact event. The architecture therefore relies on layered controls:
network isolation, least-privilege secrets, audited deployment changes,
runtime monitoring, durable state recovery, and protocol-level safeguards
outside the scope of this document.
