# BCP/DR Plan with RTO/RPO - Sundial Node

Generated: 2026-08-17

## Purpose

This document defines the business continuity and disaster recovery plan for
[`demo/midgard-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node),
including recovery paths, backup posture, RTO/RPO
objectives, and the benchmark evidence currently available.

The node already exposes operational
recovery endpoints and stores durable recovery state across Redis, Postgres,
MPT LevelDB/EFS, and L1 state-queue data. The RTO/RPO values below distinguish
between measured pipeline recovery evidence and restore objectives that still
require production-like backup/restore drills.

Current determination: the testnet node has a documentable BCP/DR posture and
measured recovery evidence for several failure classes. It is not yet in a
final mainnet BCP/DR SLA state.

## Scope

In scope:

- [`demo/midgard-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)
  API, tx processor, and sequencer roles.
- Redis Streams ingress queue and dead-letter stream.
- Postgres protocol tables and projections.
- LevelDB-backed Merkle-Patricia-Trie state mounted on EFS in AWS.
- L1 provider and Cardano state-queue interaction needed for reset, repair,
  commitment, merge, and cold-start seeding.
- Operator recovery endpoints: `/reset`,
  `/stateQueue/root-unit-diagnostics`, and
  `/stateQueue/repair-root-units`.

Out of scope:

- End-user wallet recovery.
- Third-party L1 provider internal recovery.
- Mainnet validator-level escape-hatch/fund recovery design except where it
  affects node operation.
- KYC/onboarding systems outside
  [`demo/midgard-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node).

Related documents:

- [`../compliance/data-handling-addendum.md`](data-handling-addendum.md)
- [`../compliance/threat-model-stride.md`](threat-model-stride.md)
- [`../mainnet-readiness.md`](../mainnet-readiness.md)
- [`../security.md`](../security.md)
- [`demo/midgard-node/README.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/README.md)
- [`demo/midgard-manager/packages/scalability-harness/benchmark-runs/30eb12b19e9b3e4a3dbf90ee93309d3a5aed7a3c/report.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-manager/packages/scalability-harness/benchmark-runs/30eb12b19e9b3e4a3dbf90ee93309d3a5aed7a3c/report.md)

## Service Criticality

[`demo/midgard-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)
is the off-chain coordinator for L2 transaction admission,
mempool persistence, block commitment, L1 submission, state-queue merge, and
read/query surfaces. Service interruption affects:

- New transaction submission and transaction-status queries.
- Ingress queue processing.
- Mempool-to-block commitment progress.
- L1 state-queue submission and merge progress.
- Operator observability and recovery diagnostics.

The design goal is fail-closed recovery: local state must not be cleared before
the corresponding on-chain reset/repair action succeeds, and partial submission
states must be reconciled before normal sequencing resumes.

## Recovery Objectives

The following objectives are current operational targets for testnet/demo
operation. They are not a mainnet SLA until validated with mainnet-shaped
traffic, provider latency, finality behavior, and backup restore drills.

| Recovery scenario | RTO objective | RPO objective | Current evidence / caveat |
| --- | ---: | ---: | --- |
| Process/container restart with existing storage intact | 15 minutes | 0 for committed Postgres/MPT state; queued Redis entries depend on Redis durability | Startup reconciles durable state and can seed `BlocksDB` from chain when empty. This needs a timed restart drill in AWS. |
| Pipeline backlog drain after sustained load | 10 minutes at 800 TPS; 5 minutes at 1,000 TPS under measured test profile | 0 residual queue/mempool backlog at recovery checkpoint | Measured in the scalability report: `initial-800-replay` drained mempool to 0 within a 600s recovery window; `institutional-1000-replay` drained mempool to 0 within a 300s recovery window. |
| Full `/reset` of node protocol state | 30 minutes target | Intentional reset to clean protocol state after on-chain burn succeeds | `/reset` has a 600s spend-and-burn timeout and only clears local databases/MPT after the on-chain step succeeds. Total operator RTO depends on L1/provider response and post-reset `/init`. |
| Targeted state-queue root-unit repair | 20 minutes target | 0 state-queue root units immediately after repair, then exactly 1 after `/init` | `/stateQueue/repair-root-units` has a 900s timeout, burns duplicate root units, verifies count 0, and requires `/init` to mint a fresh root unit. |
| Postgres point-in-time restore | 60 minutes target for testnet; mainnet target TBD | Testnet: up to RDS backup/PITR window; mainnet target TBD | Terraform configures automated RDS backup retention from `rds_backup_retention_days`; testnet is 1 day. Mainnet readiness explicitly requires a separate retention decision. |
| EFS/MPT storage loss | 60 minutes target when Postgres and L1 are intact | 0 for reconstructable committed state; mempool MPT recovery needs drill evidence | MPT paths are on encrypted EFS. No AWS Backup policy for EFS is defined in this repo. Recovery should prefer restoring EFS or rebuilding from authoritative Postgres/L1 state, then verifying roots. |
| Redis loss before tx-processor admission | 30 minutes to resume service | Potential loss of queued-but-unprocessed submissions | AWS ElastiCache config currently sets `snapshot_retention_limit = 0`; Redis is not the long-term source of truth. After durable mempool acceptance, Postgres is authoritative. |
| Regional/provider outage | Mainnet target TBD | Depends on latest durable state restored in alternate region/provider | Not measured. Requires provider failover and infrastructure restore drill before claiming mainnet readiness. |

## Measurement Caveat

The measured RTO/RPO evidence comes from
[`demo/midgard-manager/packages/scalability-harness/benchmark-runs/30eb12b19e9b3e4a3dbf90ee93309d3a5aed7a3c/report.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-manager/packages/scalability-harness/benchmark-runs/30eb12b19e9b3e4a3dbf90ee93309d3a5aed7a3c/report.md).
Those numbers were measured at testnet/demo scale under an emulator-backed
test profile, not real mainnet traffic. They may not hold once mainnet traffic,
provider latency, block finality pressure, adversarial submissions, and
production infrastructure constraints are real.

These values are current engineering evidence and current testnet operating
objectives. They are not final mainnet SLA values because the same recovery
paths have not yet been timed under mainnet-like load and production
backup/restore exercises.

## Backup Strategy

### Postgres

Postgres is the primary durable off-chain store for accepted transactions,
blocks, block-to-transaction mappings, user events, address history, ledger
projections, and faucet claims when enabled.

Current controls:

- RDS storage encryption is enabled.
- Automated backup retention is controlled by `rds_backup_retention_days`.
- Final snapshots are supported through `skip_final_snapshot = false` and
  `final_snapshot_identifier`.
- Deletion protection is supported by `rds_deletion_protection`.
- CloudWatch PostgreSQL log export is enabled.

Current testnet setting:

- `rds_backup_retention_days = 1`.
- `rds_multi_az = false`.
- `rds_deletion_protection = false`.
- `rds_skip_final_snapshot = false`.

Mainnet requirement:

- Set a mainnet-specific backup retention period based on the approved RPO.
- Enable Multi-AZ unless an explicit risk acceptance says otherwise.
- Keep deletion protection enabled.
- Keep final snapshots enabled.
- Run and record a restore drill before relying on the RTO.

### Redis Streams

Redis holds transaction CBOR after HTTP submission and before tx-processor
admission, plus dead-letter entries for messages that exceed retry limits.

Current controls:

- Redis consumer groups provide at-least-once processing.
- Stale pending entries are reclaimed.
- Failed messages are retried up to `TX_QUEUE_MAX_DELIVERY_ATTEMPTS`, then
  copied to a dead-letter stream.
- Readiness checks include Redis availability.

Current limitation:

- The AWS ElastiCache resource has `snapshot_retention_limit = 0`. Therefore,
  Redis must be treated as a transient queue for RPO purposes. Submissions
  already inserted into Postgres/MempoolDB are protected by Postgres recovery;
  submissions only present in Redis may need client resubmission after Redis
  loss.

Mainnet requirement:

- Decide whether Redis snapshotting, replication, or a different queue
  durability target is required for the approved submission RPO.
- Preserve enough ingress metadata for operators to identify the resubmission
  window after Redis loss.

### MPT LevelDB / EFS

The node stores ledger and mempool Merkle-Patricia-Trie data at
`LEDGER_MPT_DB_PATH` and `MEMPOOL_MPT_DB_PATH`. In AWS this state is mounted on
encrypted EFS with a dedicated KMS key and EFS access point.

Current controls:

- EFS encryption at rest is enabled.
- KMS key rotation is enabled.
- The EFS access point uses a fixed UID/GID and scoped root directory.
- MPT operations are paired with SQL transactions through checkpoint/commit or
  revert logic.
- `/reset` deletes and recreates MPT stores only after on-chain reset succeeds.

Current limitation:

- This repo does not define an AWS Backup policy for EFS snapshots.

Mainnet requirement:

- Define EFS backup retention or document why MPT rebuild is sufficient.
- Test a restore or rebuild procedure and compare Postgres, MPT roots, and L1
  state-queue state before declaring recovery complete.

### L1 State and Provider Data

L1 is the external source for state-queue UTxOs, user events, submitted
commitments, and merge status. The node depends on Lucid provider access through
Blockfrost or Kupo/Ogmios.

Current controls:

- Readiness checks verify provider availability.
- L1 calls use bounded timeouts in key operational paths.
- `BlocksDB` can be seeded from the on-chain state queue when empty.
- Root-unit diagnostics can detect duplicate or missing state-queue root units.

Mainnet requirement:

- Define provider failover criteria.
- Monitor provider lag and submission errors.
- Consider cross-checking critical reads against a second provider.

### Secrets and Configuration

Operator seed phrases, provider keys, database credentials, and Grafana secrets
are injected through AWS Secrets Manager in the AWS deployment path.

Mainnet requirement:

- Document break-glass access.
- Test secret rotation for provider keys and non-fund-bearing credentials.
- Keep operator signing material isolated from public API roles where the
  deployment topology allows role separation.

## Existing Recovery Paths

### 1. Standard Restart / Automatic Reconciliation

Use this path after process crash, ECS task replacement, deploy rollback, or
transient provider/database unavailability when storage is intact.

Operator steps:

1. Confirm health and readiness status.
2. Confirm Redis, Postgres, L1 provider, and MPT paths are available.
3. Watch queue depth, pending Redis messages, mempool size, unsubmitted block
   backlog, submitted/confirmed/merged block status, and worker errors.
4. Allow startup reconciliation and cold-start seeding to complete.
5. Declare recovery complete once readiness is healthy, no critical worker is
   stuck, and queue/mempool/backlog metrics are draining or back to normal.

Completion criteria:

- API readiness is healthy.
- Redis consumer group is processing or idle.
- No block is permanently stuck in `SUBMITTING`.
- Queue and mempool depth are stable or draining.
- L1 provider calls are succeeding.

### 2. Full Reset (`GET /reset`)

Use this path for deliberate reset of node protocol state, unrecoverable local
state mismatch, or pre-benchmark/pre-replay cleanup. Do not use it for ordinary
process restarts.

Behavior:

- Acquires the global reset/repair lock.
- Spends and burns authenticated validator UTxOs on chain.
- Fails closed if the on-chain spend-and-burn step fails or times out.
- Clears local Postgres protocol tables.
- Deletes ledger and mempool MPT stores.
- Clears the transaction ingress queue on a best-effort basis.
- Resets in-memory counters.

Timeout:

- On-chain spend-and-burn timeout is 600 seconds.

Operator steps:

1. Restrict access to operator endpoints.
2. Pause or block public submit traffic if the environment is public.
3. Call `GET /reset`.
4. If the endpoint returns `409 Conflict`, wait for the active reset/repair to
   finish and retry only after confirming lock state.
5. After success, call the initialization path required for the environment.
6. Verify state-queue health, database row counts, MPT paths, and readiness.

Completion criteria:

- Reset endpoint returned success.
- Local state has been cleared.
- MPT stores have been recreated.
- `/stateQueue/root-unit-diagnostics` is healthy after reinitialization.
- Submit/commit/merge paths resume in the expected state.

### 3. State-Queue Root-Unit Diagnostics

Use this path before reset/repair, during startup investigation, and after a
repair or initialization.

Endpoint:

```bash
curl -s http://localhost:3000/stateQueue/root-unit-diagnostics | jq .
```

Healthy response:

- `status: "ok"`.
- `count: 1`.
- Exactly one `outRefs` entry.

Unhealthy response:

- `status: "invalid"` with `count: 0` or `count > 1`.
- `status: "error"` with HTTP 503 when provider/query fails.

Operator decision:

- If `count > 1`, use targeted root-unit repair when full `/reset` would be
  too slow or too destructive.
- If `count = 0`, run the normal initialization path if the environment is
  expected to be initialized.
- If provider query fails, fix provider/network access before running repair.

### 4. Targeted Root-Unit Repair

Use this path when the state queue has duplicate root units and a full reset is
unnecessary or too slow for deep historical state.

Endpoint:

```bash
curl -i http://localhost:3000/stateQueue/repair-root-units
```

Behavior:

- Acquires the same global reset/repair lock used by `/reset`.
- Queries root-unit UTxOs at the state-queue address.
- Skips repair if root-unit count is `0` or `1`.
- If count is greater than `1`, burns all duplicate root-unit UTxOs in a single
  repair transaction.
- Verifies that remaining root-unit count is `0`.
- Requires `GET /init` after repair to mint a fresh single root unit.

Timeout:

- Repair timeout is 900 seconds.

Operator steps:

1. Run root-unit diagnostics and preserve the output in the incident record.
2. Call `GET /stateQueue/repair-root-units`.
3. If the endpoint returns `409 Conflict`, wait for the active reset/repair to
   finish.
4. Re-run root-unit diagnostics and confirm `count: 0`.
5. Call `GET /init`.
6. Re-run diagnostics and confirm `status: "ok"` and `count: 1`.
7. Resume normal sequencing and monitor merge/commitment progress.

Completion criteria:

- Duplicate root units are gone.
- Initialization minted exactly one root unit.
- Commitment and merge paths continue without root-unit ambiguity.

### 5. Postgres Restore

Use this path when RDS storage is corrupted, accidentally modified, or
unavailable beyond normal failover/restart.

Operator steps:

1. Freeze public writes and operator mutations.
2. Identify the restore point from RDS automated backup or final snapshot.
3. Restore to an isolated RDS instance first where practical.
4. Point a staging node at the restored database and verify schema, latest block
   status, mempool counts, and address/history projections.
5. Compare restored state against L1 state-queue data.
6. Restore or rebuild MPT state and compare roots.
7. Promote restored database to the node only after validation.
8. Resume traffic and monitor queue, mempool, commitment, submission, and merge
   metrics.

Completion criteria:

- Database is restored to an approved point.
- No impossible block status transitions are present.
- MPT and L1 state-queue checks are consistent.
- Health/readiness checks pass.

### 6. Redis Loss / Queue Recovery

Use this path when Redis data is lost or the stream is unrecoverable.

Operator steps:

1. Preserve logs and metrics around the outage window.
2. Restore Redis service.
3. Confirm tx processors recreate or rejoin the consumer group.
4. Identify the time window where submissions may have been accepted by HTTP
   but not yet inserted into Postgres.
5. Ask clients or load drivers to resubmit transactions from that window when
   applicable.
6. Monitor duplicate handling, dead-letter growth, and mempool acceptance.

Completion criteria:

- Redis readiness passes.
- Queue processing is active or idle.
- Mempool acceptance resumes.
- Affected clients have a clear resubmission window.

## Incident Workflow

1. Detect and classify the incident.
2. Freeze unsafe writes when data integrity is uncertain.
3. Preserve evidence: logs, metrics, traces, endpoint responses, database
   counts, L1 transaction ids, and provider errors.
4. Choose the least destructive recovery path.
5. Execute recovery with one operator driving and one operator reviewing where
   possible.
6. Validate completion criteria before reopening public traffic.
7. Write a post-incident record with RTO/RPO actually observed.

## Monitoring and Evidence

Minimum signals to capture during recovery:

- API `/health` and `/health/ready`.
- Redis stream depth, pending count, retry/dead-letter count, and processor
  lag.
- Mempool size and final mempool after recovery.
- Block commitment, submission, confirmation, and merge counters.
- Commitment and merge failure counters.
- Blocks stuck in `UNSUBMITTED`, `SUBMITTING`, `SUBMITTED`, or `CONFIRMED`.
- State-queue root-unit diagnostics.
- L1 provider errors, timeouts, and submission tx ids.
- RDS CPU/storage/connection/log errors.
- EFS availability and MPT open/commit/revert errors.

## Current Measured Evidence

Benchmark commit:

- `30eb12b19e9b3e4a3dbf90ee93309d3a5aed7a3c`

Environment:

- Node endpoint: `http://localhost:3000`.
- Prometheus endpoint: `http://localhost:9090`.
- L1 provider mode: `emulator`.
- Host: `dev3` Linux x64, 6 CPUs, 31.0 GB RAM.
- Transaction profile: `one-to-one`.

Measured results:

| Run | Load window | Recovery window | Recovery result |
| --- | ---: | ---: | --- |
| `baseline-100-800-replay` | 180s per tier at 100, 200, 400, 800 TPS | 90s per tier | Final queue 0 and final mempool 0 after each tier. |
| `initial-800-replay` | 1800s at 800 TPS | 600s | Committed 1,292,478 transactions at 718.03 committed tx/s; final mempool 0; 0 commitment failures; 0 merge failures. |
| `institutional-1000-replay` | 1800s at 1,000 TPS | 300s | Committed 1,571,744 transactions at 873.17 committed tx/s; final mempool 0; 0 commitment failures; 0 merge failures. |
| `warmup-replay` | 600s at 100 TPS | 120s | Final mempool 0 under `maxRecoveryMempoolSize: 0`; 0 commitment failures; 0 merge failures. |

Interpretation:

- The best currently measured backlog-drain RTO is less than or equal to 300
  seconds for the sustained 1,000 TPS replay profile.
- The sustained 800 TPS validation recovered within a 600-second recovery
  window.
- The measured RPO for queue/mempool backlog at the checkpoint is 0 residual
  backlog, not a guarantee that every client submission has final user-visible
  inclusion under all future conditions.
- Accepted-to-committed p95 latency at sustained high throughput remained high:
  195s at 800 TPS and 270s at 1,000 TPS. This affects user experience and
  should be tracked separately from recovery success.

## Mainnet BCP/DR Determination

Sundial has a documented and testnet-measured BCP/DR posture for the current
node environment.

Sundial is not yet in a final mainnet BCP/DR SLA state.

The remaining conditions for that state are:

- Timed ECS restart drill with existing RDS/EFS/Redis intact.
- Timed RDS point-in-time restore drill.
- EFS restore or MPT rebuild drill with root comparison.
- Redis loss drill and client resubmission procedure.
- Provider failover drill.
- Mainnet-like sustained load and recovery benchmark with realistic provider
  latency and finality pressure.
- Access-control confirmation for all operator endpoints.
- Approved mainnet RDS backup retention, Multi-AZ posture, and deletion
  protection settings.
- EFS backup retention decision or approved MPT rebuild posture.
- Post-incident template that records observed RTO, observed RPO, and evidence
  links.

## Review Cadence

Review this document:

- Before each mainnet-readiness milestone.
- After any change to Redis, Postgres, MPT, state-queue, reset, repair, or
  provider-failover behavior.
- After every recovery drill or production incident.
- Whenever benchmark evidence is refreshed.
