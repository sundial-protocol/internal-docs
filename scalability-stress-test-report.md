**Document Metadata**

- **Author:** Vic Genin

- **Date:** 2026-05-05

**Versioning History**

| Version | Date       | Author    | Description                              |
| :------ | :--------- | :-------- | :--------------------------------------- |
| 1.0     | 2026-04-01 | Vic Genin | Initial scalability and stress test plan |
| 2.0     | 2026-05-28 | Vic Genin | Testnet/mainnet tier split               |

# Scalability and Stress Test Plan for Sundial Testnet

## 1. Introduction

This document is a scalability and stress testing plan extension to the
Sundial Master Test Plan. It defines the target-driven load validation approach
for the Sundial testnet implementation based on the Midgard L2 node and
transaction load generation tooling.

The purpose of this plan is to establish a repeatable, evidence-based
method for evaluating whether Sundial can sustain high L2 transaction submission
volumes, continue building and submitting block commitments, preserve service
stability during peak demand, and expose sufficient metrics for engineering and
community-facing performance reporting.

The plan is structured to support:

- controlled load generation through a reporting harness, using
  `l2/midgard-manager` where sufficient;

- runtime observation through `l2/midgard-node` telemetry and Grafana;

- benchmarking reports under heavy network load;

- side-by-side comparison before and after optimizations;

- dashboard evidence showing transaction volume, backlog, block production,
  worker duration, resource usage, and failure counts.

This document is not the final benchmark report and is not a mainnet capacity
certification. It defines how load will be generated, which metrics will be
used, how much transaction volume should be applied, which acceptance criteria
must be met, and which evidence must be collected before a scalability claim is
made. The actual scalability report is produced after execution from the
evidence package, metric exports, logs, and run summaries described here.

## 2. Test Objectives

The objectives of scalability and stress testing are to:

1. Measure Sundial node behavior under increasing L2 transaction submission
   volume.

2. Identify the highest sustained transaction submission rate that the testnet
   environment can process without significant backlog growth, commitment
   failures, merge failures, or service instability.

3. Confirm that transaction acceptance, mempool processing, block commitment,
   block submission, and merge workflows continue operating under load.

4. Measure latency and throughput indicators before, during, and after load.

5. Record resource usage trends for CPU, memory, and network throughput.

6. Establish a repeatable benchmark package for before/after optimization
   comparisons.

7. Produce dashboard-ready evidence suitable for the final scalability report,
   internal review, and
   community-accessible performance summaries.

8. Define the observability, evidence, and reporting requirements that must be
   satisfied before stronger production scalability claims can be made.

## 3. Scope

The scope covers scalability and stress testing of the active l2 workspace:

- `l2/midgard-node` as the running Sundial/Midgard node;

- `l2/midgard-manager/packages/tx-generator` as the transaction load generator,
  with extension or harness wrapping where required;

- Prometheus, Grafana, Loki, Tempo, and cAdvisor observability stack;

- L2 transaction submission through `POST /submit`;

- in-memory transaction queue processing;

- `MempoolDB` growth and drain behavior;

- block commitment worker behavior;

- submitted and merged block progress;

- node container resource usage.

The scope includes both steady-state load tests and stress tests that ramp
traffic until a defined stop condition or stability limit is reached.

## 4. Out of Scope

The following areas are outside the scope of this plan:

- mainnet production capacity certification;

- external L1 provider capacity guarantees;

- long-duration economic-security analysis;

- formal audit conclusions;

- permanent public alerting and paging policy;

- production cost modeling beyond observed testnet transaction and commitment
  costs;

- protocol-level changes to block lifecycle semantics, merge semantics, or
  canonical encoding.

## 5. Scalability Targets and Load Driver Requirements

The scalability test owner defines the performance target. The load generator is
an implementation detail used to produce traffic. Any load driver used for
formal execution must satisfy the load-driver contract, reporting requirements,
and evidence requirements in this plan.

The target basis comes from:

- `internal-docs/architecture/scaling.md`, which states an initial Sundial L2
  testnet target of approximately `800 TPS`, a testnet industrial target of
  `1,000 TPS`, a mainnet baseline of `2,000 TPS` comparable to established
  payment network benchmarks, and a practical mainnet estimate of `5,000 TPS`
  on commodity general-purpose infrastructure;

- `internal-docs/reports/M2.1-Technical-Requirements.md`, which identifies
  optimized transaction processing for institutional volumes, advanced mempool
  management for predictable execution, and scalable infrastructure designed for
  growing demand as Milestone 2 technical requirements;

- the latency model in `internal-docs/architecture/scaling.md`, which defines
  transaction latency as the time from submission and producer availability to
  block inclusion, with an L1-backed worst-case inclusion upper bound of
  approximately `20 seconds`.

### 5.1 Target Classes

The test plan covers both testnet validation targets and mainnet performance targets. Testnet tiers establish confidence in the architecture under controlled conditions. Mainnet tiers demonstrate the throughput levels required for production deployment.

| Target Class              | Environment | Purpose                                                                                                                                | Target Signal                                                                                             |
| :------------------------ | :---------- | :------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------- |
| Testnet sustained target  | Testnet     | Demonstrate and document the initial L2 testnet throughput target.                                                                     | Sustained `800 TPS` with stable service behavior.                                                         |
| Testnet industrial target | Testnet     | Prove stable operation above the testnet baseline and validate mempool, queue, commitment, and resource behavior.                      | Sustained `1,000 TPS` test window.                                                                        |
| Mainnet baseline target   | Mainnet     | Validate throughput comparable to established payment network benchmarks.                                                              | Sustained `2,000 TPS` with stable service behavior.                                                       |
| Practical mainnet target  | Mainnet     | Validate the architecture's practical mainnet TPS estimate under controlled conditions or document the bottlenecks preventing it.      | Sustained `5,000 TPS` target window or bottleneck evidence.                                               |
| Latency target            | Both        | Confirm that durably accepted mempool transactions are included in committed blocks inside the architecture's stated latency envelope. | p95 mempool-accepted-to-committed inclusion latency `<= 20s`, where instrumentation supports measurement. |
| Cost target               | Both        | Confirm that L1 commitment cost remains stable or improves per committed L2 transaction as block volume increases.                     | L1 fee per committed L2 transaction remains stable or decreases under larger block volumes.               |

### 5.2 Load Driver Contract

The load driver must be capable of generating the requested benchmark profile,
regardless of whether that is achieved by extending Midgard Manager, adding a
separate harness, or running coordinated workers across multiple hosts.

The load driver contract is:

- produce one-to-one, multi-output, and mixed transaction profiles;

- support fixed duration, fixed target TPS, fixed transaction count, warm-up,
  cool-down, spike, step-ramp, and saturation-discovery modes;

- distinguish attempted, generated, submitted, enqueued, durably accepted,
  rejected, timed-out,
  and node-unavailable transaction counts;

- emit machine-readable JSONL per submission and a summary JSON/CSV artifact per
  run;

- record per-request client latency, HTTP status, response class, transaction
  type, transaction ID, CBOR byte size, retry count, and error class;

- support deterministic seeds and reusable transaction corpora so before/after
  optimization runs can be compared fairly;

- make availability checks configurable so probe traffic does not distort the
  benchmark;

- expose load-driver metrics or artifacts so load-driver host saturation can be
  ruled out;

- exit with a status code that reflects the run outcome.

### 5.3 Reporting-Grade Load Driver Requirements

Any load driver or harness used for formal scalability execution must produce
reporting-grade evidence. The formal benchmark report must not depend on
unstructured console output or inferred transaction counts.

The required capabilities are:

- fixed run controls for duration, transaction count, warm-up, and cool-down;

- machine-readable JSONL and summary output;

- per-request latency and status accounting;

- accurate enqueued/durable-accepted/submitted/failure counters based on
  individual node
  responses;

- bounded retry behavior with explicit retry and final outcome accounting;

- configurable availability probing;

- deterministic mixed-mode generation and replayable transaction corpora;

- load-driver Prometheus metrics or structured host-resource evidence;

- multi-process or multi-host coordination for high-TPS targets;

- replay mode for identical before/after transaction corpora.

Node-side Prometheus metrics remain the authoritative source for enqueued,
durably accepted, rejected, committed, and merged transaction counts.
Load-driver artifacts provide client-side pacing, latency, error, and
generated/submitted transaction evidence.

## 6. Metrics to Use

Metrics must be captured from the node telemetry endpoint and Grafana dashboard.
The node metrics endpoint is exposed on the Prometheus exporter port, commonly
`http://localhost:9464/metrics` when `PROM_METRICS_PORT=9464`.

The Grafana dashboard is defined in `l2/midgard-node/grafana/dashboard.json`.

### 6.1 Primary Scalability Metrics

| Area                      | Metric or Query                                                 | Meaning                                                                 | Success Signal                                                      |
| :------------------------ | :-------------------------------------------------------------- | :---------------------------------------------------------------------- | :------------------------------------------------------------------ |
| Enqueued submissions      | `tx_submissions_enqueued_total`                                 | Count of valid L2 submissions accepted by `POST /submit` into tx queue. | Increases in line with generated load.                              |
| Enqueue rate              | `rate(tx_submissions_enqueued_total[$__rate_interval])`         | Enqueued submissions per second.                                        | Sustains target rate during test window.                            |
| Durable mempool accepted  | `tx_submissions_mempool_accepted_total`                         | Count of submitted txs durably written to `MempoolDB`.                  | Tracks enqueued volume with bounded processing lag.                 |
| Durable acceptance rate   | `rate(tx_submissions_mempool_accepted_total[$__rate_interval])` | Durably accepted submissions per second.                                | Sustains target rate and tracks enqueue rate within tolerance.      |
| Rejected submissions      | `tx_submissions_rejected_total`                                 | Malformed or non-hex CBOR rejected at the HTTP boundary.                | Remains `0` for valid generated load.                               |
| In-memory backlog         | `tx_queue_size`                                                 | Queue size before tx processing.                                        | Does not grow without recovery.                                     |
| Mempool depth             | `mempool_tx_count`                                              | Transactions held in `MempoolDB`.                                       | Rises under load and drains through commitments.                    |
| Committed tx volume       | `commit_block_tx_count_total`                                   | Transactions included in committed blocks.                              | Catches up to durable accepted volume after processing lag.         |
| Commit drain rate         | `rate(commit_block_tx_count_total[$__rate_interval])`           | Committed transactions per second.                                      | Meets or approaches durable acceptance rate over sustained windows. |
| Committed blocks          | `commit_block_count_total`                                      | Number of committed Midgard blocks.                                     | Continues increasing under load.                                    |
| Block commit rate         | `rate(commit_block_count_total[$__rate_interval])`              | Committed blocks per second.                                            | Stable and non-zero during active commitment windows.               |
| Transactions per block    | `commit_block_txs_per_block`                                    | Tx count in the most recently committed block.                          | Stable within expected block sizing behavior.                       |
| Block commitment duration | `commit_block_duration_seconds`                                 | Latest block commitment worker duration.                                | Does not trend upward beyond benchmark threshold.                   |
| Block event size          | `commit_block_events_size_bytes`                                | Total event byte size in the latest block.                              | Tracks payload size without failure.                                |
| Block commitment failures | `commit_block_commitment_failures_total`                        | Worker failures, timeouts, SDK/CML errors.                              | Remains `0`.                                                        |
| Merged blocks             | `merge_block_count_total`                                       | Blocks merged into confirmed state.                                     | Continues increasing when merge conditions are met.                 |
| Merge failures            | `merge_block_failures_total`                                    | Merge transaction failures.                                             | Remains `0`.                                                        |

Deprecated alias note:

- `tx_submissions_accepted_total` is deprecated and must not be used for new
  benchmark analysis or report generation. Use
  `tx_submissions_enqueued_total` and
  `tx_submissions_mempool_accepted_total` instead.

### 6.2 Infrastructure Metrics

| Area                 | Metric or Query                                                    | Meaning                                            | Success Signal                                   |
| :------------------- | :----------------------------------------------------------------- | :------------------------------------------------- | :----------------------------------------------- |
| Node scrape health   | `up{job="midgard_nodes"}`                                          | Prometheus can scrape the node.                    | Remains `1`.                                     |
| Running containers   | `count(container_last_seen{image!=""})`                            | Observability stack and node container visibility. | No unexpected container loss.                    |
| CPU usage            | `sum(rate(container_cpu_user_seconds_total{image!=""}[5m]) * 100)` | Aggregate container CPU usage.                     | No sustained saturation without throughput gain. |
| Per-container CPU    | `rate(container_cpu_user_seconds_total{image!=""}[5m]) * 100`      | CPU usage by container.                            | Node/Postgres bottleneck can be identified.      |
| Memory usage         | `sum(container_memory_usage_bytes{image!=""})/1024/1024`           | Aggregate memory usage in MiB.                     | No unbounded growth or OOM behavior.             |
| Per-container memory | `container_memory_usage_bytes{image!=""}`                          | Memory by container.                               | Stable after load stops.                         |
| Network receive      | `irate(container_network_receive_bytes_total{image!=""}[5m])`      | Container inbound traffic.                         | Correlates with load volume.                     |
| Network transmit     | `irate(container_network_transmit_bytes_total{image!=""}[5m])`     | Container outbound traffic.                        | Correlates with responses and telemetry.         |

### 6.3 Latency and Cost Evidence

Latency and cost evidence must be captured from both load-driver artifacts and
node-side telemetry. The formal benchmark evidence set must include:

| Evidence                          | Source                                                                                                 | Reporting Use                                   |
| :-------------------------------- | :----------------------------------------------------------------------------------------------------- | :---------------------------------------------- |
| Submission duration               | Load-driver JSONL request timings or interim manager batch log output                                  | Client-side submit latency and pacing evidence. |
| Durable-accepted-to-committed lag | Difference between `tx_submissions_mempool_accepted_total` and `commit_block_tx_count_total` over time | Backlog and eventual processing latency proxy.  |
| Queue and mempool drain time      | Time for `tx_queue_size` and `mempool_tx_count` to return to baseline after load stops                 | Recovery latency.                               |
| Block commitment duration         | `commit_block_duration_seconds`                                                                        | Node-side commitment latency proxy.             |
| L1 commitment fees                | Commitment transaction evidence from L1 provider or tx logs                                            | Cost metric for block commitment path.          |
| L2 generated transaction fees     | Transaction corpus metadata and Lucid protocol params                                                  | Report separately from L1 commitment fees.      |

Additional observability requirements:

- bounded-label HTTP request counters and duration histograms for
  `POST /submit`;

- enqueued-to-mempool and mempool-accepted-to-committed duration tracking where
  traceable without high-cardinality metric labels;

- explicit L1 commitment fee/cost metrics after successful submission.

## 7. Performance Benchmarks and Acceptance Criteria

Formal benchmark thresholds must be approved before execution. The following
thresholds are recommended for testnet scalability reporting.

| Category                  | Target                                                                                                                      |
| :------------------------ | :-------------------------------------------------------------------------------------------------------------------------- |
| Availability              | Node API remains available during the test and recovery windows.                                                            |
| Scrape health             | `up{job="midgard_nodes"}` remains `1`.                                                                                      |
| Submission rejection rate | `tx_submissions_rejected_total` remains `0` for valid generated load.                                                       |
| Commitment failures       | `commit_block_commitment_failures_total` remains `0`.                                                                       |
| Merge failures            | `merge_block_failures_total` remains `0`.                                                                                   |
| Queue recovery            | `tx_queue_size` returns to baseline within 10 minutes after load stops.                                                     |
| Mempool recovery          | `mempool_tx_count` returns to pre-test baseline or documented expected residual within 30 minutes after load stops.         |
| Commitment progress       | `commit_block_count_total` continues increasing during sustained load windows.                                              |
| Testnet sustained target  | `800 TPS` sustained tier passes before higher targets are claimed.                                                          |
| Testnet industrial target | `1,000 TPS` sustained tier passes.                                                                                          |
| Mainnet baseline target   | `2,000 TPS` sustained tier passes before practical mainnet claims are made.                                                 |
| Practical mainnet target  | `5,000 TPS` tier either passes or produces clear bottleneck evidence with remediation owners.                               |
| Inclusion latency         | p95 mempool-accepted-to-committed latency is `<= 20s` where instrumentation supports per-transaction or cohort measurement. |
| Throughput stability      | Durable accepted tx/s and committed tx/s do not degrade by more than 20% across the steady-state window after warm-up.      |
| Cost stability            | L1 commitment fee per committed L2 transaction remains stable or improves as block volume increases.                        |
| Resource stability        | CPU and memory remain below environment-specific saturation limits and do not show unbounded growth.                        |
| Service recovery          | No manual restart is required after the test unless the test is explicitly a destructive saturation run.                    |

The following classification should be used:

| Result                   | Definition                                                                                              |
| :----------------------- | :------------------------------------------------------------------------------------------------------ |
| Passed                   | All benchmark targets met with complete evidence.                                                       |
| Passed with Observations | No critical failure, but non-blocking degradation, backlog, or observability limitation observed.       |
| Failed                   | Service instability, unrecovered backlog, commitment failure, merge failure, or target breach occurred. |
| Blocked                  | Required environment, load-driver, harness, or telemetry capability was unavailable.                    |

## 8. Recommended Transaction Volumes

The recommended volumes are target-driven. They are not limited to any single
CLI surface. The selected load driver and harness must produce each profile with
adequate pacing, observability, reproducibility, and evidence before the run is
treated as a formal benchmark.

Each tier should be run at least once before optimization and once after
optimization if side-by-side comparisons are required.

### 8.1 Formal Benchmark Tiers

| Tier                     | Environment | Purpose                                                                                                                 | Transaction Profile          | Target Rate                     | Duration | Approximate Tx Volume |
| :----------------------- | :---------- | :---------------------------------------------------------------------------------------------------------------------- | :--------------------------- | :------------------------------ | :------- | :-------------------- |
| Instrumentation warm-up  | Both        | Confirm environment, harness, dashboards, logs, and reports.                                                            | one-to-one                   | `100 TPS`                       | 10 min   | 60,000 tx             |
| Testnet TPS validation   | Testnet     | Demonstrate the initial L2 testnet throughput target.                                                                   | one-to-one                   | `800 TPS`                       | 30 min   | 1,440,000 tx          |
| Testnet industrial       | Testnet     | Prove stable operation above the testnet baseline.                                                                      | one-to-one                   | `1,000 TPS`                     | 30 min   | 1,800,000 tx          |
| Mainnet baseline         | Mainnet     | Validate throughput comparable to established payment network benchmarks (approximately Visa average sustained volume). | one-to-one and mixed         | `2,000 TPS`                     | 30 min   | 3,600,000 tx          |
| Practical mainnet target | Mainnet     | Validate or falsify the architecture's practical mainnet TPS estimate.                                                  | one-to-one first, then mixed | `5,000 TPS`                     | 30 min   | 9,000,000 tx          |
| Peak spike               | Mainnet     | Simulate short peak-demand bursts above the sustained target.                                                           | one-to-one and mixed         | `2x last passed sustained tier` | 5 min    | depends on tier       |

### 8.2 Mixed Payload Tiers

Mixed payload runs apply to mainnet tiers only (mainnet baseline and practical
mainnet target). Testnet tiers use one-to-one transactions throughout to keep
the testnet validation surface minimal and results straightforward to interpret.

Mixed payload runs should be used after one-to-one runs pass at the same target
rate. Mixed mode is intended to represent more realistic transaction shape
variation and must include both simple transfer-style transactions and
multi-output transactions.

The first mixed benchmark profile should use:

- 70% one-to-one transactions;

- 30% multi-output transactions;

- the same target TPS, duration, and acceptance criteria as the corresponding
  one-to-one tier;

- deterministic generation seed recorded in the evidence package;

- actual enqueued, durable accepted, and committed counts verified through
  node metrics.

If mixed mode produces materially different CBOR sizes, block event sizes,
commitment duration, or resource usage, those differences must be highlighted in
the final report instead of averaging them away.

### 8.3 Saturation Discovery

Saturation discovery should be run only after formal benchmark tiers have been
completed and reviewed.

The recommended ramp is:

1. Run the highest passed sustained tier for 10 minutes.

2. Increase target rate by 25% for 10 minutes.

3. Continue increasing by 25% while stability and recovery criteria pass.

4. When the run reaches or exceeds `5,000 TPS`, keep increasing only if a
   higher exploratory ceiling is explicitly requested. Reaching throughput in
   the 20,000+ TPS range requires L1 protocol enhancements such as data
   availability improvements and state-channel scaling mechanisms
   (for example, Hydrozoa/Gummiworm-style L1 settlement channeling).

5. Record the first bottleneck and classify whether it is load-driver-side,
   node-side, database-side, L1-provider-side, or infrastructure-side.

Stop the saturation test immediately if:

- `commit_block_commitment_failures_total` increases;

- `merge_block_failures_total` increases;

- the node API becomes unavailable;

- the Prometheus scrape target goes down;

- memory grows without stabilization;

- queue or mempool backlog does not recover during the recovery window;

- block commitment duration exceeds the approved threshold for three
  consecutive commitment cycles.

## 9. Test Environment

Scalability and stress tests should be executed in an isolated staging or
testnet environment with the observability stack enabled.

Required services:

- Midgard node started with monitoring enabled;

- PostgreSQL runtime database;

- configured L1 provider boundary for the target environment;

- Prometheus scraping the node exporter;

- Grafana dashboard loaded from `l2/midgard-node/grafana/dashboard.json`;

- cAdvisor metrics available for container resource usage;

- Loki and Tempo available for log and trace evidence where configured;

- reporting-grade load driver or harness installed and validated. This may use
  Midgard Manager internally, but the harness owns run control, evidence
  collection, and report output.

Required environment evidence:

- Git commit or release identifier;

- environment name;

- node configuration summary with secrets redacted;

- machine or container resource allocation;

- Prometheus scrape target status;

- Grafana dashboard URL or exported dashboard snapshot;

- test wallet mode or key reference, with secrets excluded;

- L1 provider mode and endpoint type, with secrets excluded.

### 9.1 Hardware Profile and Cloud Equivalents

Sundial L2 node scalability validation should be performed on commercially
available commodity general-purpose compute, not on specialized high-end
servers, HPC, or accelerator-backed infrastructure. The protocol
direction is to preserve a path to decentralized operation by keeping the node
profile affordable and accessible for independent operators while still
documenting how throughput changes as operator resources improve.

The baseline local hardware profile captured for this plan is:

| Resource     | Baseline Profile                                                                     |
| :----------- | :----------------------------------------------------------------------------------- |
| CPU          | Mainstream desktop-class CPU, Intel 8th+ generation or comparable, 6+ physical cores |
| Architecture | x86_64                                                                               |
| Memory       | 32 GB RAM                                                                            |
| OS           | Ubuntu 24.04 class Linux host                                                        |
| Storage      | 500 GB SSD-backed storage class                                                      |

Comparable cloud shapes for baseline and repeatable testnet benchmarking should
use general-purpose instances in the same approximate class, with SSD-backed
persistent storage sized for the run evidence, database growth, MPT storage, and
logs.

| Provider                    | Comparable Shape                                             | Published Shape                                                                               | Use in This Plan                                                  |
| :-------------------------- | :----------------------------------------------------------- | :-------------------------------------------------------------------------------------------- | :---------------------------------------------------------------- |
| AWS EC2                     | `m6i.2xlarge`                                                | 8 vCPU, 32 GiB memory, EBS-only storage, up to 12.5 Gbps network, up to 10 Gbps EBS bandwidth | x86 general-purpose baseline comparable to the local host class.  |
| AWS EC2                     | `m8i-flex.2xlarge` or equivalent newer general-purpose shape | 8 vCPU, 32 GiB memory, EBS-only storage, up to 15 Gbps network, up to 10 Gbps EBS bandwidth   | Newer general-purpose baseline where available.                   |
| Google Cloud Compute Engine | `n2-standard-8`                                              | N2 standard class uses 4 GB memory per vCPU; 8 vCPU maps to 32 GB memory                      | x86 general-purpose baseline comparable to the local host class.  |
| Google Cloud Compute Engine | `e2-standard-8`                                              | 8 vCPU, 32 GB memory, no local SSD, up to 16 Gbps egress bandwidth                            | Cost-optimized general-purpose baseline for affordability checks. |

Provider specification references:

- AWS EC2 general-purpose instance specifications:
  <https://aws.amazon.com/ec2/instance-types/general-purpose/>

- Google Cloud Compute Engine general-purpose machine specifications:
  <https://cloud.google.com/compute/docs/general-purpose-machines>

Hardware reporting rules:

- every benchmark run must record exact CPU model or cloud machine type, vCPU or
  core count, memory, storage type, storage size, free disk space before and
  after the run, OS image, kernel version, and network placement;

- baseline scalability claims should start from commodity general-purpose
  compute in the 6-8 core and approximately 32 GB RAM class;

- higher-throughput experiments may scale hardware only as an explicit variable,
  and the final report must separate software/protocol gains from hardware
  scaling gains;

- results produced on specialized compute, bare metal, unusually large memory
  nodes, local NVMe-only storage, or high-bandwidth network SKUs must be marked
  as operator-optimized runs rather than baseline decentralization runs;

- the decentralization-oriented target is an affordable operator profile, not a
  requirement for oversized centralized infrastructure.

## 10. Test Data Management

The load driver should use controlled generated data only.

For exploratory benchmarks, ephemeral test wallets and synthetic UTxOs are
acceptable. For formal repeatable runs, use controlled test keys, deterministic
seeds, and reusable transaction corpora stored according to internal
secret-handling practices.

Test data must record:

- transaction type;

- one-to-one ratio for mixed tests;

- target TPS;

- actual attempted TPS;

- load profile shape;

- intended test duration;

- actual start and stop times;

- generated, submitted, enqueued, durably accepted, rejected, timed-out, and
  node-unavailable
  transaction counts from harness artifacts;

- enqueued, durably accepted, and rejected transaction counts from Prometheus.

Generated transaction files must not be treated as authoritative evidence of
node acceptance. Node acceptance must be verified through
`tx_submissions_enqueued_total`,
`tx_submissions_mempool_accepted_total`, API responses, logs, and downstream
commitment metrics.

## 11. Test Execution Procedure

Each scalability test run should follow this sequence:

1. Run the harness preflight checklist and require `Passed` before formal load:
   scenario validity, node probe, Prometheus scrape health, required metrics
   presence, artifact directory writability, and tx-generator invocability.

2. Record environment baseline and Git commit.

3. Confirm node API availability.

4. Confirm Prometheus scrape health.

5. Confirm Grafana dashboard panels are populated.

6. Capture pre-test values for primary and infrastructure metrics.

7. Start the selected load profile through the reporting harness.

8. Record exact start time, harness configuration, target TPS, transaction
   profile, and intended duration.

9. Monitor enqueued submissions, durable mempool acceptances, rejected
   submissions, queue size, mempool count, committed tx count, block count,
   commitment duration, failure counts, CPU, memory, and network usage.

10. Stop the load driver at the planned duration.

11. Continue monitoring through the recovery window.

12. Capture post-test metric values, dashboard screenshots or exports, logs,
    load-driver artifacts, and report output.

13. Classify the run as Passed, Passed with Observations, Failed, or Blocked.

14. Record defects and follow-up actions.

## 12. Benchmark Result Template

Each run must be summarized using the following format.

| Field                                      | Value |
| :----------------------------------------- | :---- |
| Run ID                                     |       |
| Date and time                              |       |
| Environment                                |       |
| Git commit                                 |       |
| Node image or package version              |       |
| Harness command or config                  |       |
| Target class                               |       |
| Transaction mode                           |       |
| Load profile                               |       |
| Intended duration                          |       |
| Actual duration                            |       |
| Target tx/s                                |       |
| Attempted tx count                         |       |
| Load-driver submitted tx count             |       |
| Load-driver failed tx count                |       |
| Prometheus enqueued tx count delta         |       |
| Prometheus durable accepted tx count delta |       |
| Prometheus rejected tx count delta         |       |
| Committed tx count delta                   |       |
| Committed block count delta                |       |
| Merged block count delta                   |       |
| p95 mempool-accepted-to-committed latency  |       |
| L1 fee per committed L2 tx                 |       |
| Peak tx queue size                         |       |
| Peak mempool tx count                      |       |
| Peak block commitment duration             |       |
| Peak CPU usage                             |       |
| Peak memory usage                          |       |
| Commitment failure delta                   |       |
| Merge failure delta                        |       |
| Recovery time                              |       |
| Result                                     |       |
| Evidence references                        |       |

## 13. Before and After Optimization Comparison

Side-by-side comparisons must use identical load profiles, environment class,
duration, and transaction mode.

| Metric                                   | Before Optimization | After Optimization | Change | Notes |
| :--------------------------------------- | :------------------ | :----------------- | :----- | :---- |
| Enqueued tx/s p50 window average         |                     |                    |        |       |
| Enqueued tx/s p95 window average         |                     |                    |        |       |
| Durable accepted tx/s p50 window average |                     |                    |        |       |
| Durable accepted tx/s p95 window average |                     |                    |        |       |
| Committed tx/s p50 window average        |                     |                    |        |       |
| Committed tx/s p95 window average        |                     |                    |        |       |
| Total enqueued tx                        |                     |                    |        |       |
| Total durable accepted tx                |                     |                    |        |       |
| Total committed tx                       |                     |                    |        |       |
| Rejected tx                              |                     |                    |        |       |
| Peak queue size                          |                     |                    |        |       |
| Peak mempool size                        |                     |                    |        |       |
| Queue recovery time                      |                     |                    |        |       |
| Mempool recovery time                    |                     |                    |        |       |
| Peak commitment duration                 |                     |                    |        |       |
| Average commitment duration              |                     |                    |        |       |
| Commitment failures                      |                     |                    |        |       |
| Merge failures                           |                     |                    |        |       |
| Peak CPU usage                           |                     |                    |        |       |
| Peak memory usage                        |                     |                    |        |       |
| L1 commitment fee total                  |                     |                    |        |       |
| L1 commitment fee per committed tx       |                     |                    |        |       |

Optimization comparisons must include notes describing what changed between
runs. Examples include node code changes, database changes, worker tuning,
infrastructure changes, L1 provider changes, transaction shape changes, or
configuration changes.

## 14. Dashboard and Public Reporting Package

The community-accessible dashboard or graph package should include low-risk,
non-sensitive panels only.

Recommended public dashboard panels:

- enqueued transactions per second;

- durable mempool accepted transactions per second;

- committed transactions per second;

- total enqueued transactions;

- total durable mempool accepted transactions;

- total committed transactions;

- transactions per committed block;

- block commitment duration;

- mempool transaction count;

- transaction queue size;

- committed blocks;

- merged blocks;

- block commitment failures;

- merge failures;

- CPU and memory usage;

- network receive and transmit rates.

Do not expose:

- private keys;

- raw transaction CBOR;

- wallet addresses unless explicitly approved for testnet publication;

- transaction hashes as metric labels;

- arbitrary request paths;

- raw exception messages as metric labels;

- infrastructure secrets or provider keys.

Community-facing summaries should state:

- environment type;

- date of benchmark;

- software version or commit;

- transaction mode;

- duration;

- total enqueued transactions;

- total durable mempool accepted transactions;

- total committed transactions;

- sustained enqueued tx/s;

- sustained durable accepted tx/s;

- sustained committed tx/s;

- peak queue and mempool depth;

- failure counts;

- known limitations.

## 15. Defect Management

Any failure, degradation, or observability gap found during scalability testing
must be recorded in the private defect register.

Suggested severity:

- Critical: node unavailable, commitment path halted, unrecoverable backlog,
  data inconsistency, or required metrics unavailable during formal test.

- High: sustained throughput collapse, repeated worker failures, merge failures,
  significant memory growth, or missing evidence for a primary benchmark claim.

- Medium: partial degradation, slow recovery, noisy but recoverable errors, or
  dashboard inconsistency.

- Low: documentation gap, minor dashboard labeling issue, or non-blocking test
  runner usability issue.

Each defect should include:

- run ID;

- component;

- severity;

- observed metric or log evidence;

- reproduction command;

- expected behavior;

- actual behavior;

- recommended owner;

- resolution or follow-up.

## 16. Risks and Mitigations

| Risk                                                     | Impact                                       | Mitigation                                                                                   |
| :------------------------------------------------------- | :------------------------------------------- | :------------------------------------------------------------------------------------------- |
| Load driver host becomes the bottleneck                  | Understates node capacity                    | Record load-driver CPU/memory and use separate or multiple load-driver hosts for high tiers. |
| Node telemetry lacks HTTP latency histograms             | Latency claims are incomplete                | Use load-driver timings now; add node latency histograms before stronger public claims.      |
| L1 provider variability affects commitment timing        | Test results may vary by provider condition  | Record provider mode and run repeated benchmark windows.                                     |
| Unbounded load profile can overload environment abruptly | Hard failure instead of controlled discovery | Use staged tiers, step ramps, and stop criteria before exploratory saturation.               |
| Mixed mode transaction count is approximate              | Expected tx volume may differ slightly       | Validate actual enqueued and durable accepted tx deltas from Prometheus.                     |
| Offline generated files may be mistaken for accepted txs | Inflated report totals                       | Treat only node metrics and successful responses as enqueue/acceptance evidence.             |
| Resource metrics may aggregate observability containers  | Harder bottleneck analysis                   | Use both aggregate and per-container panels.                                                 |

## 17. Exit and Acceptance Criteria

Scalability testing is complete when:

- warm-up, testnet TPS validation, testnet industrial, mainnet baseline,
  practical mainnet target, and at least one mixed payload run have been
  executed or explicitly marked blocked;

- required metrics were captured for every completed run;

- no unresolved Critical defects remain;

- no unexplained commitment or merge failure occurred during accepted formal
  runs;

- queue and mempool recovery behavior was measured;

- before/after comparisons were completed where optimization claims are made;

- `800 TPS` testnet sustained tier was completed before stronger scalability
  claims are made;

- the `5,000 TPS` practical mainnet target either passed or was converted into
  a documented bottleneck/remediation report;

- dashboards or exported graphs are available for review;

- limitations and observability constraints are documented.

The final conclusion must distinguish between:

- proven behavior under the tested environment and load profile;

- inferred behavior requiring further confirmation;

- future production-readiness work.

## 18. Traceability Matrix

| Requirement                                | Validation Method                                          | Primary Evidence                                                                                             |
| :----------------------------------------- | :--------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| Load testing under high transaction volume | One-to-one and mixed benchmark tiers                       | Harness artifacts, `tx_submissions_enqueued_total`, `tx_submissions_mempool_accepted_total`, Grafana exports |
| Peak demand simulation                     | Peak spike and saturation discovery                        | Enqueued tx/s, durable accepted tx/s, queue size, mempool count, resource metrics                            |
| Stability under stress                     | Recovery windows and failure metrics                       | Failure counters, API availability, scrape health, logs                                                      |
| Transaction speed stability                | Durable accepted tx/s, committed tx/s, commitment duration | Prometheus rates and `commit_block_duration_seconds`                                                         |
| Cost stability                             | L1 commitment fee evidence and committed tx count          | L1 tx records, fee-per-committed-tx calculation                                                              |
| Testnet TPS validation                     | Testnet TPS validation tier                                | Sustained `800 TPS` run evidence                                                                             |
| Practical mainnet TPS estimate             | Practical mainnet target tier                              | Sustained `5,000 TPS` run or bottleneck report                                                               |
| Benchmark reports                          | Run template and comparison table                          | Completed report tables and evidence references                                                              |
| Before/after comparison                    | Repeat same load profile on two builds                     | Side-by-side metric table                                                                                    |
| Community dashboard                        | Grafana dashboard or exported graphs                       | Public-safe dashboard panels                                                                                 |

## 19. Final Report Conclusion Format

The final scalability conclusion should use the following format:

- Summary of executed load profiles.

- Highest sustained durable accepted tx/s that passed all criteria.

- Highest sustained committed tx/s observed.

- Total transactions enqueued, durably accepted, and committed.

- Peak queue and mempool depth.

- Recovery time after load stopped.

- Commitment and merge failure counts.

- Resource usage summary.

- Fee or cost summary where available.

- Before/after optimization result, if applicable.

- Residual risks and observability limitations.

- Final disposition: Passed, Passed with Observations, Failed, or Blocked.

## Appendix A: Harness and Load Driver Requirements

The formal scalability harness and selected load driver must provide the
following capabilities before reports are treated as industry-standard benchmark
evidence.

| Capability               | Required Behavior                                                                                                                         | Owner                        |
| :----------------------- | :---------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------- |
| Fixed run profiles       | Run by target TPS, duration, transaction count, warm-up, cool-down, spike, and ramp configuration.                                        | Harness or manager extension |
| Structured artifacts     | Emit per-request JSONL plus summary JSON and CSV for every run.                                                                           | Harness or manager extension |
| Accurate accounting      | Count attempted, generated, submitted, enqueued, durably accepted, rejected, timed out, retried, and unavailable transactions separately. | Manager extension            |
| Latency statistics       | Compute p50, p90, p95, p99, max, and time-windowed latency summaries.                                                                     | Harness                      |
| Retry semantics          | Implement and report bounded retry behavior without hiding final failure outcomes.                                                        | Manager extension            |
| Availability probing     | Support startup-only, periodic, per-request, and disabled availability checks.                                                            | Manager extension            |
| Deterministic generation | Record random seeds and support replayable transaction corpora.                                                                           | Manager extension            |
| Multi-worker execution   | Coordinate multiple processes or hosts when target TPS exceeds one host's capacity.                                                       | Harness                      |
| Prometheus collection    | Query Prometheus for node and infrastructure metrics across exact run windows.                                                            | Harness                      |
| Grafana evidence         | Export dashboard snapshots or panel images for report attachments.                                                                        | Harness                      |
| Cost collection          | Pull L1 commitment fee data and calculate fee per committed L2 transaction.                                                               | Harness                      |
| Report rendering         | Produce markdown, HTML, or PDF reports from run artifacts and metric queries.                                                             | Harness                      |

The implementation sequence should be:

1. Provide fixed run controls and structured output.

2. Fix accurate per-transaction success/failure accounting and retry behavior.

3. Add deterministic generation and replay support.

4. Build the external harness that launches runs, gathers Prometheus/Grafana/log
   evidence, and renders reports.

5. Add multi-host coordination only when single-host load generation becomes
   the measured bottleneck.

## Appendix B: Evidence Package Index

The evidence package should include:

- exact harness configuration and load-driver console output;

- Prometheus metric export or query screenshots;

- Grafana dashboard screenshots or JSON export;

- node logs for the test window;

- worker logs for commitment and merge activity;

- cAdvisor resource panels;

- L1 commitment transaction records where applicable;

- defect register entries;

- final run summary table;

- before/after comparison table where applicable.
