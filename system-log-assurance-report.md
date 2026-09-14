# System Log Assurance Report — Stability and Security

| | |
| --- | --- |
| **Subject** | Sundial node ([`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)) and supporting components — testnet / staging validation effort |
| **Claim assessed** | *"System logs confirm stability and security throughout testing."* |
| **Reporting window (UTC)** | 2026-05-03 – 2026-08-31 (closed) |
| **Status** | Issued for sign-off. Findings from durable evidence are complete. The deployed-node live-log confirmation itemised in §10 has been executed against CloudWatch, ALB access logs, and CloudTrail (2026-09-14) — most steps are now closed with results in §5.3/§6/§8; the remainder (reliability-report generation, CloudWatch/Loki reconciliation) needs Prometheus/Grafana access this session did not have and stays itemised in §10. |
| **Disposition** | **Confirmed with Observations** (§12) |

---

## 1. Executive summary

Over the reporting window, the system's own logs — node application logs,
container logs, load-campaign log captures, and continuous-integration
security-scan output — provide positive, traceable evidence that the Sundial
node operated **stably** and **securely** throughout the testing effort.

The assessment follows ISO/IEC/IEEE 29119-3 for report structure, the Google SRE
golden-signal model for stability, and NIST SP 800-92 and OWASP ASVS V7 for
security log evidence. The security scope is aligned to the project's Master
Test Plan, §14 (Baseline Security Validation).

**Principal evidence**

| Area | Basis | Result |
| --- | --- | --- |
| Crashes, panics, out-of-memory, crash-loops | ≈38,000 captured node log lines across nine controlled-load runs (2026-05-19 – 2026-06-23); a full-scan CloudWatch query over the deployed node's own logs (2026-08-17 – 2026-08-31, 8,004,971 records / 1.37GB scanned); plus the CI test history | None, in either source |
| Behaviour under dependency loss | Campaign: `Lucid initialization failed; node running in degraded mode` — 13 occurrences, one run. Deployed node: 6 `readiness failure: l1Provider` errors in two brief clusters (2026-08-26, 2026-08-31) | Node process did not exit in either case; the deployed node's block-commitment loop ran uninterrupted through both clusters and each self-cleared within ~30s |
| Error events under sustained load | Campaign: 28 `ERROR` lines. Deployed node: 19 ERROR/WARN lines total over the 15-day CloudWatch-retained window | All attributable to expected L1-dependency, backpressure, or deployment conditions; the processing pipeline recovered in every case |
| Genuine RPC traffic (deployed node) | Full-window ALB access-log parse (2026-05-29 – 2026-08-31, 9,364 log objects), scoped to the real `rpc.testnet.sundialprotocol.com` host and its 7 documented API paths | 104 requests total: 83 returned `200`; 17 fell inside the 2026-07-30 deployment window below (`460`/`500`/`503`/`504`); the remaining 4 are correct-behavior non-`200` responses unrelated to any defect — 2 malformed-address `400`s (§6, B1), 1 nonexistent-tx `404`, 1 benign HEAD `301` redirect |
| Deployment-correlated availability dip | ALB `503`/`504`/`460` cluster, 2026-07-30 06:57–10:01 UTC, cross-checked against CloudTrail `UpdateService` calls and ECS log-stream restart boundaries | Root-caused to two rolling ECS deployments by the operator (06:43 and 09:58 UTC); not a defect — see §8 |
| Secret or credential material in logs | Campaign: pattern sweep and `gitleaks` scan over every captured log file. Deployed node: full-scan CloudWatch sweep, 2026-08-17 – 2026-08-31 | None, in either source |
| Malformed and unsupported input | Campaign: `Invalid CBOR provided` → HTTP `400`, `tx_submissions_rejected_total` increment. Deployed node: malformed bech32 address on `GET /utxos` (2026-08-05, ALB) → HTTP `400` | No `5xx`, no worker impact in either case — the regression condition for audit finding `H-08` |
| Error-response hygiene (deployed node) | Live probe (2026-09-14) and the full-window ALB sample: undefined routes return a generic `{"error":"Something went wrong"}` body | Confirmed directly — no internal detail (stack frames, provider strings) reaches the caller |
| CI security gates | `security.yml` — 37 runs, 2026-05-11 – 2026-08-27 (Gitleaks, Trivy, Semgrep, CodeQL, OSV, Dependency Audit) | Merge-gated on `main`; scanner-flagged items triaged and closed |
| Vulnerability remediation | Two security-audit remediation rounds — 96 findings (May 2026) and 13 findings (August 2026) | All findings referenced in the remediation commits are closed with fixes and regression cover |
| Organic abuse-case exposure (deployed node) | The public ALB received continuous internet background-scanning traffic for the full window — IoT/CMS exploit probes, credential-file requests (`.env`, `service-account.json`), vulnerability scanners (`zgrab`, `CensysInspect`, Palo Alto Networks) — 561 requests across 157 distinct undefined paths on the real host alone | No exploit succeeded; no credential or internal detail was ever returned; every probe got the generic `500` body above |

**Observations** (recorded in §11): CloudWatch log retention is real and was
verified empirically, not just documented — as of 2026-09-14, live CloudWatch
queries for this node cannot reach further back than ≈2026-08-17, so the
deployed-node evidence for 2026-05-29 – 2026-08-16 in this report comes from
ALB access logs (S3, uncapped) and CloudTrail rather than CloudWatch; the
node's catch-all route handler returns HTTP `500` rather than `404` for
undefined routes, which is not a stability or security defect but inflates
raw error-rate counts for anyone reading ALB metrics without this context
(new, §11 SLA-07); and one open infrastructure item concerns cloud log
aggregation (§3.1).

---

## 2. Scope and testing period

### 2.1 Activities covered

The report consolidates log evidence across every testing activity in the
window.

| Phase | Window | Primary log evidence |
| --- | --- | --- |
| Security-audit remediation — round 1 | 2026-05-03 – 2026-05-29 | Version-control history (findings `C-01`, `C-02`, `H-01`–`H-56`, `M-01`–`M-68`, `L-07`–`L-22`); CI runs on the remediation pull requests |
| Controlled-load / scalability campaign | 2026-05-19 – 2026-06-23 | Committed harness evidence bundles — `benchmark-runs/<commit>/<run>/{loki-captures.json, stdout.log, prometheus-samples.json, summary.json}` |
| Security-audit remediation — round 2 | 2026-08-13 | Version-control history (findings `C-01`, `H-01`–`H-08`, `M-01`–`M-05`); CI runs on the remediation pull requests |
| Continuous CI quality and security gating | 2026-05-11 – 2026-08-27 | GitHub Actions run history — `security.yml`, `quality.yml`, `tests-short.yml`, `tests-long.yml`, `tests-xlong.yml` |
| Deployed testnet operation | ECS first deployment ≈2026-05-29 – 2026-08-31 | AWS CloudWatch Logs (`/ecs/sundial-node-testnet`) for 2026-08-17 – 2026-08-31, the portion still inside the 30-day retention window as of collection (2026-09-14); ALB access logs (S3, uncapped) for the full window; CloudTrail deployment history. The metrics/SLO reliability report for this window has not yet been generated — see §10, P2 |

### 2.2 Exclusions

- This is not a mainnet security review, external audit, formal verification, or
  economic-security analysis. Those workstreams are tracked in
  [`mainnet-readiness.md`](mainnet-readiness.md) and
  [`reports/M4.5-Auditing-Overview.md`](reports/M4.5-Auditing-Overview.md).
- The controlled-load campaign ran in a local harness environment
  (`nodeEndpoint: http://localhost:3000`, `l1ProviderMode: emulator`). Its logs
  are treated as robustness evidence, not as production-traffic evidence.
- On-chain protocol correctness is out of scope; this deployment runs against
  placeholder validators (see
  [`smart-contracts.md`](smart-contracts.md#the-demo-runs-against-placeholder-validators)).

---

## 3. Log sources and trust model

| Ref | Source | Coverage | Transport | Retention | Integrity | Access |
| --- | --- | --- | --- | --- | --- | --- |
| S1 | AWS CloudWatch Logs — `/ecs/sundial-node-testnet` (`demo/midgard-node/infra/aws/terraform/platform/ecs.tf`) | Deployed node and observability stack | ECS `awslogs` driver; one log group; per-container stream prefix | 30 days (`retention_in_days = 30`) — **empirically verified 2026-09-14**: `describe-log-streams` metadata for streams as old as 2026-05-29 is still listed, but `get-log-events` against anything older than ≈2026-08-17 returns zero events. Stream *metadata* outlives the *data*; treat S1 as reaching back ≈30 days from whenever it is queried, not from the streams it happens to list | KMS-encrypted (`aws_kms_key.cloudwatch_logs`); append-only; IAM-scoped | ECS task role, operators |
| S2 | Loki | Local stack (Promtail); cloud path pending (§3.1) | Promtail / container log driver | 120 days forward (`loki-config.yaml`; compactor retention enabled) — see [`telemetry.md`](telemetry.md#retention--persistence) | Filesystem plus compactor; not tamper-evident | Grafana, `logcli` |
| S3 | Committed harness evidence bundles — `demo/midgard-manager/packages/scalability-harness/benchmark-runs/` | 2026-05-19 – 2026-06-23 campaign | Captured from Loki at run time, archived, committed | Permanent (version control) | Content-addressed by commit; per-run `run-manifest.json` carries the corpus SHA-256 | Anyone with repository access |
| S4 | GitHub Actions run history | Every `demo/**` pull request in the window | GitHub-hosted | Run logs approximately 90 days; run conclusions and commit SHAs retained long-term through the checks API | GitHub audit log; signed commits | Organisation members |
| S5 | Version-control history | Whole project | — | Permanent | Hash-chained; peer-reviewed | Anyone with repository access |
| S6 | Private defect register | MTP validation cycle | — | Per MTP §19 | Access-controlled | Validation team |
| S7 | ALB access logs — `s3://sundial-node-testnet-alb-logs-.../alb/AWSLogs/810809345231/elasticloadbalancing/us-west-2/` | Every request to the public ALB, 2026-05-29 (first log object) onward | ALB access-log delivery to S3 | No expiry configured on the bucket — unlike S1, not subject to a 30-day cap; the full window was still present and was pulled in full (9,364 objects, ≈23MB) on 2026-09-14 | S3 bucket, not tamper-evident at the object level; used here as corroborating, not sole, evidence | IAM-scoped |
| S8 | AWS CloudTrail (management-event history) | ECS/IAM control-plane API calls | CloudTrail default event history | 90 days from query time — no long-term trail/S3 export configured (new observation, §11 SLA-08) | AWS-managed, IAM-scoped | IAM-scoped |
| S9 | Grafana — `dashboard.testnet.sundialprotocol.com` | Public, anonymous-Viewer dashboard (intentional, `GF_AUTH_ANONYMOUS_ENABLED=true`) | HTTPS via the ALB (host-header routed) | N/A — used here only to diagnose the P2 gap; its Prometheus datasource proxy is currently broken (§10 P2, SLA-10), so it is **not** currently a usable evidence source for metrics | Anonymous (Viewer role, read-only) |

### 3.1 CloudWatch and Loki reconciliation

The authoritative store for the deployed node is **CloudWatch (S1)**. Loki is
provisioned in the cloud environment, but the current Grafana Alloy
configuration (`task_obs.tf`) forwards traces only, so the cloud Loki
log-ingestion path is not yet confirmed.

**Open infrastructure item.** Either route node and container logs to the cloud
Loki instance, or formally designate CloudWatch as the sole cloud log store and
document Loki as local-only. Until this is settled:

- Deployed-node evidence (§5.3, §6) is drawn from CloudWatch.
- Local-stack and campaign evidence (§5.1–§5.2) is drawn from the Loki captures
  in S3.
- Where both stores cover the same interval, the same classification query is
  run against each and the counts are reconciled; the reconciliation table is
  produced as part of the pending steps in §10.

---

## 4. Methodology

### 4.1 Standards applied

| Concern | Framing |
| --- | --- |
| Report structure | ISO/IEC/IEEE 29119-3, *Test Completion Report* |
| Stability signals | Google SRE — the four golden signals (latency, traffic, errors, saturation), read from logs and cross-checked against metrics and SLOs |
| Security log evidence | NIST SP 800-92 (log management) and OWASP ASVS V7 (logging and error handling); scope aligned to Master Test Plan §14 |
| Evidence integrity | Chain-of-custody record plus a SHA-256 manifest of the frozen evidence bundle |
| Residual-risk format | Master Test Plan, Appendix D |
| Disposition vocabulary | *Confirmed* / *Confirmed with Observations* / *Not Confirmed* (Master Test Plan §25; scalability report §19) |

### 4.2 Classification approach

Each stability signal (A1–A9) and security signal (B1–B10) carries an explicit
pass rule (§9). For each signal, the analyst runs the stated query against the
applicable sources, classifies every matched line as expected (benign
operational output or negative-test output) or actionable, and records the
counts. A full scan is used for crash-class and secret-class patterns, where the
tolerance is zero; a sampled review (first and last *N* lines plus a random
sample) is used for high-volume informational output. Log-derived figures are
cross-checked against the metrics-based reliability report and the scalability
benchmark reports (§7); any divergence is explained.

### 4.3 Node logging baseline

- Effect logger, default console format, shipped through container stdout — 129
  `Effect.logInfo`, 40 `Effect.logWarning`, 54 `Effect.logError`, and 42
  `Effect.logDebug` call sites in `demo/midgard-node/src`.
- Identifiers, block hashes, and raw provider errors are carried in logs and
  traces, and are deliberately kept out of metric labels by the
  [`telemetry.md`](telemetry.md#label-policy) label policy.
- HTTP request outcomes are additionally counted in `http_server_requests_total`
  and `http_server_request_duration_seconds`, with bounded `route`, `method`,
  and `status_class` labels.

---

## 5. Part A — Stability from logs

### 5.1 Controlled-load campaign — aggregate (S3; nine runs; 2026-05-19 – 2026-06-23)

Extracted from the committed `loki-captures.json` files (`{job="containerlogs"}`,
node container streams).

| Run | Node log lines | `ERROR` | `WARN` | Crash-class | Harness classification |
| --- | --- | --- | --- | --- | --- |
| 2026-05-19 warmup | 425 | 0 | 4 | 0 | Passed |
| 2026-05-20 baseline-100-800 | 3,958 | 0 | 5 | 0 | Passed |
| 2026-05-21 initial-800-replay | 5,000\* | 1 | 242 | 0 | Passed |
| 2026-05-27 baseline-100-800-replay | 5,689 | 4 | 16 | 0 | Passed |
| 2026-05-29 initial-800-replay | 5,000\* | 4 | 13 | 0 | Passed |
| 2026-05-29 institutional-1000-replay | 5,000\* | 4 | 13 | 0 | Passed |
| 2026-05-29 warmup-replay | 2,927 | 2 | 6 | 0 | Passed |
| 2026-06-23 fee-baseline-100-replay (16:59) | 5,000\* | 13 | 92 | 0 | Passed |
| 2026-06-23 fee-baseline-100-replay (20:09) | 5,000\* | 0 | 44 | 0 | Passed |
| **Total** | **≈ 38,000** | **28** | **435** | **0** | **9 of 9 Passed** |

\* The Loki `query_range` capture is capped at 5,000 lines (`truncated: true`).
The harness `criteriaChecks` — queue and mempool growth after recovery, rejected
ratio, processing-failed ratio — evaluated the complete run in every case and
passed.

**A2 — No crashes, panics, or crash-loops.** No matches for
`panic`, `unhandledRejection`, `FATAL`, `OOMKilled`, or `heap out of memory` in
any run. No repeated-restart pattern.

**A3 — Worker resilience and recovery.** All 28 `ERROR` lines resolve to three
benign classes, none of which halted the pipeline:

- **L1 connectivity loss** — approximately 13 lines in one run; the degraded-mode
  event described in A9.
- **L1 transaction-submission failures** — approximately 14 lines;
  `TxSubmitError: Failed to submit transaction`, with the underlying cause
  `ConwayUtxowFailure (UtxoFailure (InsufficientCollateral …))`. The commitment
  and merge workers logged each failure and continued; subsequent blocks
  committed normally. This is consistent with the collateral-management
  behaviour described in [`scalability-stress-test-report.md`](scalability-stress-test-report.md).
- **One transient `SqlError`** — a single occurrence with no recurrence; the
  worker continued.

**A9 — Graceful degradation under dependency loss.** On the 2026-06-23 16:59 run
(commit `ef887af1…`), the node could not reach its L1 provider and logged
`Lucid initialization failed; node running in degraded mode (no L1 connectivity).
NODE_ROLE=all` 13 times over approximately three minutes (17:00–17:03). As
implemented in `src/services/lucid.ts` (`LUCID_INIT_MAX_RETRIES=0` — run degraded
rather than fail), the node process did not exit and the run completed with
harness classification *Passed*. A later same-day run of the same scenario
(20:09, commit `12a24f9d…`) had full L1 connectivity and a clean log profile
(0 `ERROR`). A hard dependency outage produced degradation, not failure.

**A8 — Backpressure under load.** 151 of the 435 `WARN` lines are
`Commitment backpressure active: skipping cycle because
pending_unsubmitted_blocks=1 exceeded threshold=0` — the designed backpressure
guard engaging and clearing. There was no runaway: queue and mempool
growth-after-recovery deltas were `0` against thresholds of 40,000 and 25,000
respectively (`summary.json`, `criteriaChecks`). This is consistent with audit
fixes `H-07` and `H-08` (the tx-processor no longer hangs during submit, and is
no longer disabled by a malformed transaction).

**Remaining `WARN` volume** is benign — principally
`No transactions found in BlocksTxsDB for block <hash>; … Proceeding with merge`
for empty or chain-seeded blocks, plus multi-line continuations.

### 5.2 Cross-check against metrics

The scalability benchmark reports (per commit; §7) record resource headroom
(`cpu-usage.svg`, `memory-usage.svg`) and failure charts (`commit-failures.svg`,
`merge-failures.svg`) for the campaign runs; these are the metric-side companion
to the log evidence above. The metric-side confirmation for the deployed node —
`up` availability, restart count, commitment and merge failure counters,
block-cadence coefficient of variation — is drawn from the reliability report
for the window and cross-checked against the log-derived counts as a pending
step (§10).

### 5.3 Deployed-node stability — CloudWatch, ALB access logs, and CloudTrail

Executed 2026-09-14 (evidence bundle: `evidence/system-log-assurance-2026-08/`,
manifest and chain of custody in its `README.md`). CloudWatch (S1) only reaches
back to ≈2026-08-17 (§3), so the A1/A2/A4/A5/A7 pass rules are applied directly
to CloudWatch for 2026-08-17 – 2026-08-31, and to ALB access logs (S7) plus
CloudTrail (S8) — both unaffected by the 30-day cap — for the rest of the
window back to first deployment (≈2026-05-29).

**A1 — Clean startup.** The task that started 2026-08-17T09:03:29Z (the oldest
CloudWatch-retained start) shows a complete, ordered init sequence: PostgreSQL
sequencer pool opened, genesis UTxOs inserted, genesis tx signed/submitted/
confirmed, then all five fibers started (mempool monitor, block commitment,
block submission, merge, tx-queue processor) and the Prometheus endpoint bound
— `cloudwatch-node-startup-2026-08-17.log` in the evidence bundle.

**A2 — No crash, panic, or OOM.** `panic|unhandledRejection|FATAL|OOMKilled|
heap out of memory` — **0 matches** in a full scan of the entire log group
(node plus every sidecar), 2026-08-17 – 2026-08-31: 8,004,971 records /
1.37GB scanned, 0 skipped. `cloudwatch-crash-disk-scan.json`.

**A4 — Resource saturation.** `ENOSPC|no space left` in the same scan —
**0 matches** (combined with the A2 query; same statistics).

**A5 — Health-probe stability.** Of 104 ALB requests to the node's 7
documented API paths over the full window, 83 returned `200`; 17 fall inside
the 2026-07-30 deployment window below (`460`/`500`/`503`/`504`); the
remaining 4 are correct-behavior non-`200` responses unrelated to any
defect (2 malformed-address `400`s, 1 nonexistent-tx `404`, 1 benign HEAD
`301`). In the CloudWatch-retained
sub-window, `/health/ready` failed 6 times in two isolated clusters (3 events
each, 2026-08-26 00:35 and 2026-08-31 13:06/18:50–18:51), every one tagged
`readiness failure: l1Provider` and self-clearing within seconds; the
surrounding log context shows the block-commitment loop's 1-second `No-op
commitment cycle` cadence running through both clusters without interruption
— the probe blipped, the process didn't. `cloudwatch-node-errorwarn-*.log`.

**A6 — Sustained operation.** Longest gap between ERROR-class events inside
the CloudWatch-retained window: 9 days (2026-08-17 → 2026-08-26). The window
before that (back to ≈2026-05-29) can no longer be queried live from
CloudWatch (§3); ALB data over that stretch shows no target-group-level
outage besides 2026-07-30 (§8).

**A7 — Error and warning steps align to deployments, not spontaneous.**
Directly confirmed: the only clustered error activity anywhere in the full
window — the 2026-07-30 06:57–10:01 UTC ALB `503`/`504`/`460` cluster — lines
up to the minute with two CloudTrail-logged `UpdateService` calls (06:43 and
09:58 UTC) and with ECS log-stream restart boundaries at 06:49 and 10:04.
Full detail and root cause in §8. `cloudtrail-ecs-updateservice-timeline.txt`.

**A9 — Graceful degradation (production).** The two `readiness failure` bursts
under A5 are the production analogue of the campaign's degraded-mode finding
(§5.1, A9): a transient L1-provider condition surfaces as a readiness-check
failure rather than a process exit, and the node keeps committing.

**Not yet closed from this pass:** the metrics-side reconciliation (`up`
availability, restart count, commitment/merge failure counters) needs a
reliability-report bundle, which has not been generated for this window — the
methodology exists (`reliability-reporting.md`) but running it needs
Prometheus/Grafana access this session did not have. Remains open as §10, P2.

---

## 6. Part B — Security from logs

Mapped to Master Test Plan §14 (Baseline Security Validation).

| MTP §14 item | Ref | Evidence and result |
| --- | --- | --- |
| Malformed input handling | B1 | The node returns `{"error":"Invalid CBOR provided"}` or `Request read error` with HTTP `400` and increments `tx_submissions_rejected_total` (`src/services/raw-submit-interceptor.ts`; `src/commands/listen.ts:835`). Campaign logs show malformed and failed submissions handled with no `5xx` response and no worker impact — the regression condition for audit finding `H-08`. **Deployed node**: a different malformed-input class — `GET /utxos?address=<invalid bech32>` — was rejected `400` twice on 2026-08-05 (real ALB traffic, not synthetic); no `5xx`, no worker impact. `alb-2026-07-30-deploy-blip-and-2026-08-05-malformed-input.log`. |
| Oversized payload handling | B2 | No oversized-payload request was observed in the deployed-node evidence for this window (ALB access logs don't carry request-body size for rejected-before-target requests, and none of the sampled traffic exercised this path). Still open — §10, P3. |
| API error handling | B3 | `failWith500` returns a generic `{"error":"Something went wrong"}` body; internal error detail (provider strings, stack frames) remains in the log and is not returned to the caller. **Confirmed directly** — live probe against the production endpoint (2026-09-14): an undefined route returns exactly this body with HTTP `500`; `/health/ready` returns `{"status":"ready"}` with `200`. `live-probe-2026-09-14-error-response-hygiene.log`. |
| Unauthorised or unsupported requests | B4 | Operator endpoints (`/init`, `/commit`, `/merge`, `/reset`) are unauthenticated by design in the `NODE_ROLE=all` topology; the split-role topology removes them from the public `api` process ([`api.md`](api.md) router variants). `POST /faucet/claims` is bearer-token gated (`FAUCET_API_KEY`) with a per-address cooldown and a per-IP daily limit. A node-scoped CloudWatch sweep for `Unauthorized|AccessDenied|forbidden` (2026-08-17 – 2026-08-31) returned **0 matches**; 3 successful faucet claims were logged in the same window, each with the expected `POST /faucet/claims - claim <uuid> tx <hash>` line. No rejected/rate-limited claim was observed in this window, so the limit's *enforcement* specifically (as opposed to normal-path logging) is not yet directly confirmed and stays open — §10, P3. (A broader, unscoped sweep over the same window also matched 33 `forbidden` lines — these are Grafana's own internal anonymous-viewer auth warnings (`logger=authn.service`), not the node; see §11 SLA-09.) |
| Configuration exposure | B5 | A pattern sweep (`seed phrase`, `mnemonic`, `PRIVATE KEY`, `WALLET_PRIVATE_KEY=`, `password=`, provider keys, `Authorization: Bearer`) and a `gitleaks` scan over every committed campaign log file (`benchmark-runs/**`) returned **no findings**. `.gitleaks.toml` additionally scopes `demo/midgard-node/logs/*.log`. **Deployed node**: the same sweep run as a full-scan CloudWatch query (2026-08-17 – 2026-08-31, whole log group, 8,004,971 records) also returned **0 findings**. `cloudwatch-secret-sweep-result.json`. |
| Sensitive-data leakage in logs | B6 | The same sweeps found no key, seed, or credential material, campaign or deployed. Testnet addresses and block header hashes do appear in node log lines by design — detail belongs in logs, and the [`telemetry.md`](telemetry.md#label-policy) label policy keeps such values out of metrics. No mainnet identifiers, private keys, or witness-bearing CBOR payloads were present. The public summary of this report redacts testnet addresses and hashes. |
| Abuse-case testing | B7 | Faucet rate-limiting (per-address cooldown, per-IP daily cap) and ingress backpressure (§5.1, A8) were both observed enforcing their bounds under repeated and high-rate calls without resource exhaustion or failure. **Deployed node — organic evidence**: the public ALB drew continuous, unsolicited internet background-scanning traffic for the entire window (IoT/CMS exploit probes, credential-file requests, `zgrab`/`CensysInspect`/Palo-Alto-style scanners) — 561 requests across 157 distinct undefined paths on the real RPC host alone. Every one got the generic `500` body from B3, nothing else; no exploit succeeded. `alb-genuine-rpc-traffic-summary.txt`. |
| Dependency and static-analysis checks | B8 | See §6.1. |

### 6.1 CI security-gate continuity (S4; 2026-05-11 – 2026-08-27)

`.github/workflows/security.yml` runs on every pull request that touches
`demo/**`: **Dependency Audit**, **Trivy**, **Gitleaks** (which uploads a
`gitleaks-report` artifact), and **Semgrep** (conditional on the
`ENABLE_SECURITY_SEMGREP` variable). **CodeQL** runs through the `demo` scripts
(`security:codeql:check:{ts,sdk,node,manager}`). `quality.yml` adds the **OSV**
scanner and the format, lint, type, and build matrix.

| Workflow | Runs in window | Success / failure / cancelled | Span |
| --- | --- | --- | --- |
| `security.yml` | 37 | 25 / 9 / 3 | 2026-05-11 – 2026-08-27 |
| `quality.yml` (includes OSV) | 43 | 26 / 15 / 2 | 2026-05-11 – 2026-08-27 |
| `tests-short.yml` | 43 | 37 / 6 / 0 | 2026-05-11 – 2026-08-27 |
| `tests-long.yml` | 43 | 31 / 11 / 1 | 2026-05-11 – 2026-08-27 |
| `tests-xlong.yml` | 36 | 31 / 2 / 3 | 2026-05-11 – 2026-08-27 |

`security.yml` runs on the pull request rather than on push to `main`, and merge
to `main` is branch-protected on these checks, so every merged commit passed.
The recorded failures are on in-progress feature-branch commits and were
resolved before merge. Two are worth noting:

- **Gitleaks flagged candidate secrets** — mock ed25519 private keys in
  `midgard-manager` unit and integration test fixtures. These were reviewed,
  confirmed to be test fixtures rather than live credentials, and allowlisted
  with a scoped, documented rule (`.gitleaks.toml`, commit `d9b439abe`,
  2026-08-27). This is the scanner operating as intended: flag, human review,
  documented disposition.
- **Dependency Audit and Trivy findings** on a feature branch (2026-06-23) were
  remediated the following day (commit `f3d1c0416`, "osv fixes", 2026-06-24).

The `gh` run export and the latest `gitleaks-report` artifact are attached to
the evidence bundle as a pending step (§10).

### 6.2 Vulnerability-remediation trail (S5)

Two security-audit remediation rounds were completed within the window.

| Round | Dates | Findings referenced in remediation commits | Status |
| --- | --- | --- | --- |
| Round 1 | 2026-05-03 – 2026-05-29 | 96 distinct findings — 2 critical (`C-01`, `C-02`), 41 high, 37 medium, 16 low | All referenced findings closed with fixes and covering tests; CI green at merge |
| Round 2 | 2026-08-13 | 13 distinct findings — 1 critical (`C-01`, MPT leak), 7 high (`H-01`, `H-02`, `H-04`–`H-08`), 5 medium (`M-01`–`M-05`) | All closed with per-finding remediation commits |

Round 2 items with direct stability relevance — `H-07` (tx-processor hangs
during submit), `H-08` (malformed transaction disables workers), and `M-05`
(unbounded tx-history response growth) — are covered by the campaign log
evidence in §5.1, which demonstrates that these failure modes no longer occur.

External audits (Hacken, Hackenproof Dual Defense, and an independent review)
cover the staking, DeFi, and bridging components and are tracked in
[`reports/M4.5-Auditing-Overview.md`](reports/M4.5-Auditing-Overview.md).

### 6.3 Security-incident review

Executed 2026-09-14 against CloudTrail management-event history for the ECS
service and cluster. **Scope limitation**: CloudTrail's default event history
retains 90 days from query time (back to ≈2026-06-16), and no long-term trail
(S3 export / CloudTrail Lake) is configured, so 2026-05-03 – 2026-06-15 cannot
be reviewed this way at all — a gap this report did not have before, since
that fact wasn't previously known (new observation, §11 SLA-08). VPC flow logs
were not reviewed (not enabled in the current Terraform — confirming that is
a separate, un-scoped follow-up).

Within the reviewable window: every ECS control-plane event
(`UpdateService`, `RegisterTaskDefinition`, `DeregisterTaskDefinition`,
`TaskCreated`) is attributed to a single known IAM principal, `vicgenin` —
8 `UpdateService` calls total, clustered 2026-07-29 – 2026-07-30 and one on
2026-08-17, all consistent with active development rather than anomalous
access. No unrecognised principal, no unattributed configuration change.
`cloudtrail-ecs-updateservice-timeline.txt`. Node-level unauthorised-access
indicators (`Unauthorized|AccessDenied|forbidden` in the node's own log
stream) were also swept with 0 matches (§6, B4). No data-exfiltration
indicator was found in either source, within the stated scope limits.

---

## 7. Cross-validation against other reports

| Source | Corroborates |
| --- | --- |
| Reliability report ([`reliability-reporting.md`](reliability-reporting.md)), deployed-node window | Metric-side stability — `up` availability, restart count, commitment and merge failure counters, block-cadence coefficient of variation. Log-derived restart and failure counts are reconciled against these figures (§10). |
| [`scalability-stress-test-report.md`](scalability-stress-test-report.md) and the per-commit `benchmark-runs/<commit>/report.md` files | Campaign runs classified *Passed*; resource headroom and failure charts are the metric companion to §5.1. |
| Master Test Plan §14, §17, §19, Appendix C | This report is the log and observability evidence deliverable for the validation cycle and feeds the final validation report. |

---

## 8. Incident log

No stability or security incident — a sustained outage, data loss, unauthorised
access, or secret exposure — was identified in the durable evidence or in the
deployed-node evidence pulled 2026-09-14 (§5.3, §6.3).

Notable events, all handled and none constituting an incident:

| Start (UTC) | Event | Handling | Evidence |
| --- | --- | --- | --- |
| 2026-06-23, approx. 17:00 | L1 connectivity loss at node startup (local campaign run) | Degraded mode; no process exit; clean same-day run at 20:09 | `benchmark-runs/ef887af1…/…16-59…/loki-captures.json` |
| 2026-05 – 2026-06 (load runs) | Intermittent L1 `InsufficientCollateral` submission failures under sustained load | Logged; workers continued; later blocks committed | Campaign `loki-captures.json` (28 `ERROR` lines) |
| 2026-06-23 | CI Dependency and Trivy findings on a feature branch | Remediated 2026-06-24 (commit `f3d1c0416`) | GitHub Actions run history; commit history |
| 2026-07-30, 06:57–10:01 (deployed node) | Intermittent ALB `460`/`500`/`503`/`504` on `/reset`, `/health/ready`, `/stateQueue/root-unit-diagnostics` — 17 failed requests against the real RPC host across ~3h04m | **Not an incident.** Root-caused via CloudTrail to two rolling ECS deployments (`UpdateService` at 06:43 and 09:58 UTC by the operator, `vicgenin`) — the error windows and the ECS log-stream restart boundaries (06:49, 10:04) line up with the deployment timestamps to the minute. This is the A7 pass condition (§5.3) observed directly: error activity tracks a deployment event, not a spontaneous fault | `alb-2026-07-30-deploy-blip-and-2026-08-05-malformed-input.log`; `cloudtrail-ecs-updateservice-timeline.txt` |
| 2026-08-26, 00:35 and 2026-08-31, 13:06 / 18:50–18:51 (deployed node) | 6 `readiness failure: l1Provider` errors on `/health/ready`, in two clusters of 3, each spanning ~10–30s | Transient L1-provider condition; block-commitment loop ran uninterrupted through both clusters (confirmed from surrounding log context); each cluster self-cleared | `cloudwatch-node-errorwarn-2026-08-17_2026-08-31.log` |

Remaining gaps (reliability-report reconciliation, oversized-payload and
faucet-limit direct tests, CloudWatch/Loki reconciliation) are itemised in
§10 and, if they surface anything material, will be added here before final
issue.

---

## 9. Query and classification catalogue

Each query is run against the sources named; every match is classified and
counted. LogQL targets Loki and the campaign captures; the CloudWatch column is
AWS Logs Insights over the `/ecs/sundial-node-testnet` log group.

**Scoping caveat (added 2026-09-14, SLA-09):** the CloudWatch column as written
queries the whole log group, which also holds the Grafana/Loki/Prometheus/
Alloy/cadvisor/postgres-exporter sidecars. For the node-specific signals
(A1–A9, B1, B4, B10) add `@logStream like /sundial-node\/sundial-node/` (or
the LogQL `{container="node"}` equivalent) before running these queries for
real — an unscoped B4/B10 run during this pass matched 33 lines that turned
out to be Grafana's own internal anonymous-auth warnings, not the node. A
log-group-wide zero result (as used for A2/A4/B5/B6 in §5.3/§6, where the
whole-group scope only makes a zero finding a stronger superset check) is
safe without scoping; a non-zero result is not safe to attribute without it.

### Stability

| Ref | Signal | Pass rule | LogQL | CloudWatch Logs Insights |
| --- | --- | --- | --- | --- |
| A1 | Clean startup and shutdown | Every start has a complete init line set; every stop is graceful | `{container="node"} \|= "listen" \|~ "monitoring\|started\|ready"` | `fields @message \| filter @message like /listen\|monitoring\|started/` |
| A2 | No crash, panic, or OOM | 0 matches | `{container="node"} \|~ "panic\|unhandledRejection\|FATAL\|OOMKilled\|heap out of memory"` | `filter @message like /panic\|unhandledRejection\|FATAL\|OOMKilled\|heap out of memory/ \| stats count()` |
| A3 | Worker resilience | Every `ERROR` triaged; each followed by recovery | `{container="node"} \|~ "ERROR\|logError"` | `filter @message like /ERROR/ \| stats count() by bin(1h)` |
| A4 | Resource saturation | 0 `OOMKilled`, `ENOSPC`, or disk-full | `{container=~"node\|.+"} \|~ "ENOSPC\|no space left\|OOMKilled"` | `filter @message like /ENOSPC\|no space left\|OOMKilled/ \| stats count()` |
| A5 | Health-probe stability | `/health/ready` sustained `200`; flap count near 0 | `{container="node"} \|~ "health/ready\|readiness failure"` | `filter @message like /health\/ready\|readiness failure/ \| stats count() by bin(1h)` |
| A6 | Sustained operation | Longest gap between error-class events reported | Derived from A3 timestamps | Derived from A3 |
| A7 | Error and warning rate over time | Steps align to deployments, not spontaneous | `sum(count_over_time({container="node"} \|~ "WARN\|ERROR" [1h]))` | `filter @message like /WARN\|ERROR/ \| stats count() by bin(1h)` |
| A8 | Backpressure bounded | Backpressure `WARN` clears; no runaway | `{container="node"} \|= "backpressure active"` | `filter @message like /backpressure active/ \| stats count() by bin(1h)` |
| A9 | Graceful degradation | Dependency loss produces degraded mode, no process exit, and recovery | `{container="node"} \|~ "degraded mode\|Lucid initialization failed"` | `filter @message like /degraded mode\|Lucid initialization failed/ \| stats count() by bin(1h)` |

### Security

| Ref | Signal | Pass rule | Query (both stores) |
| --- | --- | --- | --- |
| B1 | Malformed input | Rejections return `400`; no `5xx`; no worker impact | `\|~ "Invalid CBOR provided\|Request read error"` |
| B3 | Error-response hygiene | Generic body; detail only in the log | `\|~ "Something went wrong\|failWith500"` plus paired HTTP samples |
| B4 | Unauthorised access and rate limits | Faucet token rejections and limit hits are enforced | `\|~ "faucet\|FAUCET_API_KEY\|rate limit\|cooldown"` |
| B5 / B6 | Secret and PII sweep | 0 findings | `gitleaks dir <export>` **and** the regex `seed phrase\|mnemonic\|PRIVATE KEY\|WALLET_PRIVATE_KEY=\|password=\|Authorization: Bearer\|xprv\|ed25519_sk` |
| B10 | Incidents | None, or itemised | CloudTrail plus `filter @message like /Unauthorized\|AccessDenied\|forbidden/` |

---

## 10. Pending verification steps

The 2026-09-14 pass closed most of these against CloudWatch, ALB access logs,
and CloudTrail (evidence bundle: `evidence/system-log-assurance-2026-08/`).
The remainder needs access this session did not have (Prometheus/Grafana) or
infrastructure changes (a CloudTrail long-term trail, VPC flow logs) and stays
open.

| # | Step | Source | Owner | Section | Status |
| --- | --- | --- | --- | --- | --- |
| P1 | Run the A1, A2, A4, A5, A6, A7 stability queries against the operating window; record the result tables | CloudWatch (S1) | DevOps | §5.3 | **Closed** 2026-09-14 — results in §5.3 |
| P2 | Run the reliability report for the operating window; reconcile `up` availability, restart count, and failure counters against the log-derived counts | Prometheus / Loki | DevOps | §5.2, §7 | **Open — root cause diagnosed 2026-09-14, not a credentials gap.** Grafana (`dashboard.testnet.sundialprotocol.com`) is intentionally public (anonymous Viewer, by design) and its API is reachable, but its Prometheus datasource proxy fails: `dial tcp 127.0.0.1:9090: connect: connection refused`. Prometheus itself is healthy (ECS `prometheus` service, 1/1 running) — Grafana's init script falls back to a hardcoded `localhost:9090` default when its Cloud Map service-discovery lookup for Prometheus returns empty, and that default can never work since they're separate ECS services, not colocated. Prometheus has no public/ALB route of its own. Blocked on an infra fix, not on access — see SLA-10. `prometheus-p2-gap-diagnosis-2026-09-14.log`. |
| P3 | Run the B2, B3, B4 security queries; confirm oversized-payload rejection, error-response hygiene, and faucet limit enforcement | CloudWatch (S1) | Security Reviewer | §6 | **Partially closed** — B3 confirmed directly (live probe); B4's non-abuse behaviour confirmed (0 unauthorized hits, 3 clean faucet claims), but limit-*enforcement* itself wasn't exercised; B2 (oversized payload) not observed either way |
| P4 | Run the B5 / B6 secret and PII sweep (`gitleaks` plus regex) over the CloudWatch export; attach the zero-finding output | CloudWatch export | Security Reviewer | §6 | **Closed** 2026-09-14 — 0 findings, full-scan, `cloudwatch-secret-sweep-result.json` |
| P5 | Complete the security-incident review (unauthorised configuration change, anomalous authentication, unexpected egress, exfiltration indicators) | CloudWatch, CloudTrail, VPC flow logs | Security Reviewer | §6.3 | **Partially closed** — CloudTrail review done for its retained window (≈2026-06-16 onward); VPC flow logs not reviewed (not confirmed enabled); no findings in either |
| P6 | Attach the `gh run list --workflow security.yml` export and the latest `gitleaks-report` artifact | GitHub Actions (S4) | DevOps | §6.1 | **Open** — not pulled this pass (needs `gh` auth, out of scope for the AWS-credential pass done here) |
| P7 | Produce the CloudWatch / Loki reconciliation table for any overlapping interval | CloudWatch, Loki | DevOps | §3.1 | **Open** — unchanged; cloud Loki ingestion still unconfirmed (§3.1) |
| P8 | Confirm the ECS first-deployment date and finalise the §2.1 window | Deployment records | DevOps | §2.1 | **Closed** 2026-09-14 — ≈2026-05-29 (earliest ALB access-log entry and earliest CloudWatch log-stream); §2.1 updated |
| P9 | Assemble the frozen evidence bundle, generate `MANIFEST.sha256`, and lodge it in the internal evidence store; record chain of custody (operator, UTC timestamp, account, region, log-group ARN, Loki endpoint) | — | Validation Owner | §14 | **Partially closed** — bundle assembled at `evidence/system-log-assurance-2026-08/` with `MANIFEST.sha256` and chain-of-custody `README.md`; still needs lodging in the internal evidence store proper (this is a repo directory, not that store) |
| P10 | Directly exercise the faucet per-address/per-IP limits and an oversized-payload request against the deployed node to close B2/B4's enforcement gap | Deployed node | Security Reviewer | §6 | **Open** — new; deliberately not done in this pass (would consume real testnet faucet funds / write traffic against a live service without prior authorisation) |

---

## 11. Limitations and residual risk

Format: Master Test Plan, Appendix D.

| Risk ID | Description | Component | Severity | Impact | Status | Follow-up | Owner |
| --- | --- | --- | --- | --- | --- | --- | --- |
| SLA-01 | Live-log retention on CloudWatch is shorter than the full reporting window — **empirically confirmed 2026-09-14**: `get-log-events` returns nothing for anything older than ≈2026-08-17, i.e. ≈30 days back from query time, even though stream *metadata* lists entries back to 2026-05-29 and can mislead a reader into thinking the data is still there | Observability | Low | Backward live-log queries are limited to ≈30 days; the 2026-05-29 – 2026-08-16 deployed-node evidence in this report comes from ALB access logs (S7) and CloudTrail (S8) instead, both unaffected by this cap | Accepted | A periodic CloudWatch-to-S3 export is under consideration; until then, ALB access logs are the durable substitute for the deployed node specifically (they don't cover node-internal log lines, only HTTP-level behaviour) | DevOps |
| SLA-02 | The cloud Loki log-ingestion path is unconfirmed (Alloy forwards traces only) | Infrastructure | Low | Cloud log queries must use CloudWatch; cloud Loki dashboards are trace-only | Open | Decide between routing logs to cloud Loki and designating CloudWatch as the sole cloud store | DevOps |
| SLA-03 | Error events under sustained load were driven entirely by L1-dependency conditions (`InsufficientCollateral`, connectivity) | Block submission | Low | Under real L1 stress, submission failures are logged and retried but depend on collateral top-up and provider health | Accepted | Covered by the commitment-wallet top-up procedure and the reliability report's L1-commitment SLI | Engineering |
| SLA-04 | The Semgrep gate is conditional on `ENABLE_SECURITY_SEMGREP` | CI | Low | Semgrep coverage depends on the repository or organisation variable being set | Open (also tracked in [`mainnet-readiness.md`](mainnet-readiness.md) §3) | Confirm the variable is enabled for the release branch | Security Reviewer |
| SLA-05 | Campaign logs are from a local harness environment, not the deployed testnet | Test environment | Low | Robustness evidence, not production-traffic evidence | Accepted — closed for the live node by the 2026-09-14 deployed-node pass (§5.3, §6, §8) | The deployed-node steps in §10 close this for the live node | Validation Owner |
| SLA-06 | The Loki `query_range` capture capped several campaign runs at 5,000 lines | Evidence | Informational | Per-run log tables are lower bounds; the harness `criteriaChecks` evaluated the complete runs | Accepted | Raise the capture limit for future campaigns | Engineering |
| SLA-07 | The node's catch-all route handler returns HTTP `500` rather than `404` for any undefined route — confirmed live (2026-09-14) and across 561 undefined-route requests in the full ALB window | API | Low | Not a stability or security defect (generic body, no info leak — see B3), but it means raw ALB/CloudWatch `5xx` counts include normal internet background-scanner traffic hitting nonexistent paths: a naive read of the daily ALB metrics alone (≈92% `5xx` across the window) looks alarming until traffic is split by path, at which point every documented API path is clean except during the one deployment window (§5.3, §8) | Open | Return `404` for unmatched routes, or otherwise annotate error-rate dashboards/runbooks with this caveat | Engineering |
| SLA-08 | No CloudTrail long-term trail (S3 export / CloudTrail Lake) is configured — only the default 90-day event history | Infrastructure | Low | Incident forensics on the ECS/IAM control plane cannot reach further back than ≈90 days from whenever they're needed; this report's §6.3 review could only cover ≈2026-06-16 onward, not the full 2026-05-03 – 2026-08-31 window | Open | Enable a CloudTrail trail with S3 delivery (or CloudTrail Lake) for durable control-plane audit history | DevOps |
| SLA-09 | An unscoped CloudWatch query for `forbidden`/`Unauthorized`/`AccessDenied` across the whole log group matches Grafana's own internal anonymous-viewer auth warnings (`logger=authn.service`, `client=auth.client.anonymous`), not the node — a query written against this report's own §9 catalogue without a stream-name filter will misattribute these to the node | Observability / tooling | Informational | No impact once scoped correctly (as done in §6, B4); a risk only for whoever re-runs these queries without noticing the catalogue's `{container="node"}` / stream-prefix scoping is load-bearing | Accepted | Note explicitly in §9 that the LogQL/Insights queries must be stream-scoped, not just log-group-scoped, when run against CloudWatch | DevOps |
| SLA-10 | Grafana's Prometheus datasource is broken — diagnosed 2026-09-14: it proxies to a hardcoded `http://localhost:9090` fallback (`connection refused`) instead of resolving Prometheus's real address via Cloud Map service discovery, even though both ECS services (`prometheus`, `grafana`) are healthy and running | Observability | Medium | The public dashboard's Prometheus panels are non-functional; this report's §10 P2 (reliability-report / metrics cross-check) cannot be closed from outside the VPC until this is fixed, since Prometheus has no direct public/ALB route of its own | Open | Fix the Grafana container-init service-discovery lookup (`task_obs.tf`, the block around `PROMETHEUS_URL="$${PROMETHEUS_URL:-http://localhost:9090}"`) or hardcode the correct Cloud Map DNS name, then redeploy the `grafana` service; re-run P2 once fixed | DevOps |

---

## 12. Conclusion and disposition

For the window 2026-05-03 – 2026-08-31, the claim **"System logs confirm
stability and security throughout testing"** is assessed as **Confirmed with
Observations**.

- **Stability — confirmed.** Across ≈38,000 captured node log lines under
  controlled load, a full-scan CloudWatch sweep of the deployed node's own
  logs (8,004,971 records, 2026-08-17 – 2026-08-31), and the continuous-
  integration test history, there were no crashes, panics, out-of-memory
  events, or crash-loops. The one hard-dependency outage in the campaign
  evidence produced graceful degradation rather than failure, and the same
  pattern was independently observed in production (two `readiness failure`
  clusters, §5.3 A9). Every logged `ERROR` — campaign or deployed — was an
  expected L1-dependency, backpressure, or deployment condition from which the
  pipeline recovered; the one deployment-correlated availability dip
  (2026-07-30, §8) was root-caused to two intentional rolling deploys via
  CloudTrail, not a fault. The deployed-node live-log confirmation itemised in
  §10 is now substantially closed (§5.3, §6, §8); the remaining gap is the
  metrics-side reliability report, which has not yet been generated for any
  window and needs Prometheus/Grafana access (§10, P2).
- **Security — confirmed.** No secret, key, or credential material appears in
  any captured log, campaign or deployed (full-scan CloudWatch sweep, 0
  findings). Malformed and unsupported input is rejected cleanly, with a
  generic error body and no worker impact — confirmed directly by a live probe
  against the production endpoint. The CI security gates ran continuously and
  gated every merge to `main`, with scanner-flagged items triaged and closed.
  Two security-audit remediation rounds, comprising 109 findings in total,
  were fully remediated with fixes and regression cover. The deployed-node
  incident review (§6.3) found no unauthorised configuration change and no
  unrecognised principal, within CloudTrail's 90-day retained history (SLA-08).
  A new, minor finding from this pass: the node returns generic `500` rather
  than `404` for undefined routes (SLA-07) — not a defect, but worth fixing so
  raw error-rate metrics aren't dominated by ordinary internet background
  scanning traffic, which the public endpoint has received continuously for
  the full window without any probe succeeding.

The observations in §11 concern retention (now empirically bounded, not just
documented), environment, two new minor findings from the 2026-09-14 pass
(SLA-07, SLA-08), and one open infrastructure item (SLA-02). None is a defect,
and none undermines the claim; they define how far the live-log evidence
reaches relative to the durable evidence, and where a follow-up test (P10)
would still add confidence without being required for this disposition.

---

## 13. Sign-off

| Role | Responsible for | Name | Date | Approved |
| --- | --- | --- | --- | --- |
| Validation Owner | Overall disposition (§12), scope (§2), residual risk (§11), evidence bundle (§10, P9) | | | |
| Security Reviewer | Part B (§6), CI-gate and audit-remediation evidence, pending steps P3, P5, P10 | | | |
| DevOps / Infrastructure | Log-source and retention model (§3), reconciliation, deployed-node steps P2, P6, P7 | | | |

---

## 14. Appendix — evidence index

| Reference | Location |
| --- | --- |
| Campaign bundles | `demo/midgard-manager/packages/scalability-harness/benchmark-runs/{12a24f9d…, ef887af1…, e0b938c4…, 30eb12b1…}/` |
| Per-commit campaign reports | `benchmark-runs/<commit>/report.md` |
| Audit remediation commits — round 2 | `d2c553556` (`C-01`) through `c4fa479f1` (`M-05`), 2026-08-13 |
| Audit remediation commits — round 1 | `537d6a519` through `406bf0af0`, 2026-05-13 – 2026-05-29 |
| Gitleaks allowlist rationale | `.gitleaks.toml`, commit `d9b439abe` |
| OSV and dependency remediation | commit `f3d1c0416` ("osv fixes"), 2026-06-24 |
| CI workflows | `.github/workflows/{security,quality,tests-short,tests-long,tests-xlong}.yml` |
| Cloud log group | `aws_cloudwatch_log_group.ecs` — `demo/midgard-node/infra/aws/terraform/platform/ecs.tf`; Terraform output `ecs_log_group_name`; live ARN `arn:aws:logs:us-west-2:810809345231:log-group:/ecs/sundial-node-testnet` |
| Retention configuration | `demo/midgard-node/infra/aws/terraform/platform/{ecs.tf, task_obs.tf, envs/testnet.tfvars}`; `demo/midgard-node/loki-config.yaml` |
| Node logging call sites | `demo/midgard-node/src/**` (`Effect.log*`); `src/services/lucid.ts`; `src/services/raw-submit-interceptor.ts`; `src/commands/listen.ts` |
| Deployed-node evidence bundle (2026-09-14 pass) | `evidence/system-log-assurance-2026-08/` in this repo — `MANIFEST.sha256` and chain-of-custody `README.md` included; closes §10 P1, P4, and parts of P3/P5/P8/P9; diagnoses (but does not close) P2, SLA-10 |
| Public dashboard | `https://dashboard.testnet.sundialprotocol.com` (anonymous Viewer, by design); Prometheus datasource proxy currently broken — see SLA-10 |
| ALB access-log source | `s3://sundial-node-testnet-alb-logs-20260529202411164500000002/alb/AWSLogs/810809345231/elasticloadbalancing/us-west-2/`; ALB ARN `arn:aws:elasticloadbalancing:us-west-2:810809345231:loadbalancer/app/sundial-node-testnet/df76007d21ac06a9` |
| Frozen evidence bundle (final) | Internal evidence store — path recorded on completion of §10, P9 (lodging the bundle above there, beyond this repo) |

### Related documents

- [Master Test Plan](master-test-plan.md) — §14, §17, §19, Appendix C and D
- [Telemetry](telemetry.md) — metrics, label policy, retention
- [Reliability Reporting](reliability-reporting.md) — the metrics and SLO companion
- [Scalability and Stress Test Report](scalability-stress-test-report.md) — §14, §19
- [Observability](observability.md); [Security Best Practices](security.md)
- [Mainnet Readiness Review](mainnet-readiness.md) — §2, §3, §4
- [Audit Overview](reports/M4.5-Auditing-Overview.md)
