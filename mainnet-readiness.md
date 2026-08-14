# Mainnet Readiness Review — Sundial Node

## Purpose

[`master-test-plan.md`](master-test-plan.md) defines testnet/staging validation
and is explicit that it "does not attempt to replace production-readiness
reviews, formal verification, external audit processes, or mainnet launch
assessments" (§1), and that its exit criteria cover "the defined testnet
validation cycle only" (§25). This document is that separate workstream for
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node).

It does not re-validate functional behavior — that is the Master Test Plan's
job. It exists to answer one question: **is there anything about running
`sundial-node` on mainnet, instead of testnet, that changes the risk profile
and hasn't been explicitly decided, built, or verified yet?** For an L2
sequencer node that holds operator keys and processes real transactions, most
of that gap is infrastructure, key custody, and config parity — not new
feature testing.

## Scope

In scope: [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)'s AWS deployment (ECS, RDS, EFS, ALB, Secrets
Manager, observability stack), its CI quality/security gates, and the
testnet→mainnet cutover sequence.

Out of scope: protocol-level fund-recovery guarantees, covered by
[`architecture/disaster-plan.md`](architecture/disaster-plan.md); functional
and regression testing methodology, covered by
[`master-test-plan.md`](master-test-plan.md) and
[`testing.md`](testing.md); `midgard-manager` and `midgard-sdk` except where
they gate a node release.

## Relationship to Other Documents

| Topic | Source of truth | This doc adds |
| --- | --- | --- |
| System architecture, fibers, data model | [`architecture.md`](architecture.md) | Nothing — reference only |
| Local/runtime env vars, `.env` contract | [`environment.md`](environment.md) | Which vars are secrets in AWS vs. `.env` locally |
| Functional/integration/regression/load testing | [`master-test-plan.md`](master-test-plan.md), [`testing.md`](testing.md) | Mainnet-only additions not in MTP scope |
| Monitoring stack setup | [`observability.md`](observability.md) | Alerting/paging readiness, not just dashboards |
| AWS topology, Terraform layout, env matrix | [`sundial-node/docs/aws.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docs/aws.md) | Where that doc and the checked-in tfvars have drifted, and what's still undecided |
| Operational commands (bootstrap/apply/logs/restart) | [`sundial-node/docs/deployment.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docs/deployment.md) | The go/no-go gate around using those commands against mainnet |
| Protocol-level disaster recovery, escape hatch | [`architecture/disaster-plan.md`](architecture/disaster-plan.md) | Nothing — reference only |

## Entry Criteria

Before starting this review, the current Master Test Plan cycle should meet
its own exit criteria (MTP §11): no unresolved Critical defects, no
unresolved Critical baseline security findings, and controlled load
completing without unrecoverable failure. This review assumes that baseline
and layers infrastructure- and launch-specific risk on top of it.

---

## 1. AWS Environment Parity

### 1.1 Infrastructure-as-Code Gaps (blocking)

As of this review, only the testnet side of the environment pair actually
exists in the repo:

| Artifact | Testnet | Mainnet |
| --- | --- | --- |
| `infra/aws/terraform/platform/envs/*.tfvars` | `testnet.tfvars` present | **missing** |
| `infra/aws/terraform/platform/backends/*.hcl` | `testnet.hcl` present | **missing** |
| `package.json` `infra:<env>:*` scripts (bootstrap/plan/apply/output/validate/logs/live/restart/release/destroy) | `infra:testnet:*` defined | **not defined** — `docs/deployment.md` already documents `npm run infra:mainnet:*` commands as if they exist |
| `.env.deploy.<env>.example` | referenced by `docs/deployment.md`, not actually checked in | referenced by `docs/deployment.md`, not actually checked in |

None of this requires new Terraform — `platform/*.tf` has no hardcoded
environment names, so the module is already environment-agnostic. The work
is: write `envs/mainnet.tfvars` and `backends/mainnet.hcl`, add the
`infra:mainnet:*` script family, and either create the `.env.deploy.*.example`
files or correct `deployment.md` to stop referencing files that don't exist.

**Before treating `docs/aws.md`'s Environment Matrix as the mainnet template,
reconcile it against the actual `envs/testnet.tfvars`** — they've already
drifted in at least two places:

| Field | `aws.md` says (testnet) | `envs/testnet.tfvars` actually has |
| --- | --- | --- |
| `grafana_domain` | `grafana.testnet.sundialprotocol.com` | `dashboard.testnet.sundialprotocol.com` |
| NAT strategy | `instance` | `nat_type = "gateway"` |

Any mainnet value copied from the doc instead of a verified source will
inherit this drift.

### 1.2 Capacity, Durability & Topology Parameters

Values below are testnet's actual settings from `envs/testnet.tfvars`. They
are testnet-appropriate — cheap and disposable — and none should be carried
into `mainnet.tfvars` without an explicit decision:

| Variable | Testnet value | Module default | Mainnet decision needed |
| --- | --- | --- | --- |
| `rds_instance_class` | `db.t4g.micro` | — | Sized for expected mainnet write load |
| `rds_multi_az` | `false` | — | Should almost certainly be `true` |
| `rds_backup_retention_days` | `1` | — | Set to actual recovery requirement |
| `rds_deletion_protection` | `false` (explicitly overridden) | `true` | Testnet opts *out* of the safe default — mainnet should not |
| `rds_skip_final_snapshot` | `false` | — | Confirm still `false` for mainnet |
| `ecs_instance_type` / `ecs_desired_count` / `ecs_min_size` / `ecs_max_size` | `m6i.large`, 2/2/2 | — | Sized for expected mainnet load, not copied from testnet capacity |
| `redis_node_type` | `cache.t3.micro` | — | Sized for expected mainnet load |
| `sundial_node_image` | hardcoded `us-west-2` ECR URI + `testnet-latest` tag | — | Needs `us-west-1` URI and a mainnet tag/promotion policy |

Already decided and documented in `aws.md`'s Environment Matrix (verify
against source before use, per §1.1): `aws_region` (`us-west-1`),
`availability_zones`, `vpc_cidr` (`10.3.0.0/16`), `network` (`Mainnet`), and
the Terraform state key
(`sundial-node/platform/mainnet/terraform.tfstate`).

**AWS account isolation** — not currently decided anywhere in the repo:
confirm whether mainnet deploys into the same AWS account as testnet or a
separate account. This affects the ECR repository location, IAM boundaries,
Secrets Manager isolation, and who can reach production credentials. Treat
this as a decision to make deliberately, not one that defaults to "same
account" because that's what the tfvars happens to inherit.

### 1.3 Secrets & Key Custody

Per `aws.md` and `envs/testnet.tfvars`, secrets resolve from
`sundial-node/<environment>/...` in Secrets Manager. The full set in use
today (aws.md's list plus the faucet keys referenced in
`envs/testnet.tfvars`):

```text
rds-master-password
l1-provider
l1-blockfrost-api-url
l1-blockfrost-key
l1-ogmios-key
l1-kupo-key
operator-seed-phrase
operator-seed-phrase-block-commitment
operator-seed-phrase-merge
grafana-admin-password
faucet-seed-phrase
faucet-api-key
testnet-genesis-wallet-seed-phrase-a/b/c
```

For mainnet, `sundial-node/mainnet/...`:

- `operator-seed-phrase*` control real operator funds. Before bootstrap,
  define: who generates them (offline/hardware, not typed into a laptop),
  who has read access in Secrets Manager, rotation policy, and a break-glass
  procedure if the primary holder is unavailable.
- The `testnet-genesis-wallet-seed-phrase-*` and `faucet-*` keys are demo/test
  scaffolding and should not exist under the mainnet secrets path at all —
  confirm this by omission rather than by disabling.

### 1.4 Public Network Surface Decisions

- **Grafana is anonymous-access public** on testnet, via a dedicated
  host-based ALB rule (`aws.md` §Service Placement, confirmed by
  `deployment.md`'s verification step: "Grafana URL opens without
  authentication"). This needs an explicit go/no-go for mainnet rather than
  inheriting the testnet posture by default.
- HTTPS-only enforcement (`enable_https_listener=true`) must be confirmed
  active before any mainnet DNS record goes live — the documented bootstrap
  model brings ECS and HTTPS up in a second apply, after ACM validation, so
  there's a window where this is `false` by design.
- DNS ownership for `sundialprotocol.com` during cutover: who holds registrar
  access, and what's the expected propagation/validation time to budget for.

### 1.5 State Backend Isolation

`aws.md` states the mainnet Terraform state key
(`sundial-node/platform/mainnet/terraform.tfstate`) is distinct from
testnet's, in the same S3 bucket / DynamoDB lock table. No code change is
implied — verify as a pre-apply step that the key path is correct and that
DynamoDB lock access is scoped as expected, so a testnet `apply` can never
target mainnet state by accident.

### Defense-in-depth: demo-only code paths

Independent of the infra-level `faucet_enabled` Terraform variable (§1.3),
the application layer already gates demo/testnet-only behavior on
`NETWORK === "Mainnet"`: the faucet is disabled
(`config.ts`: `faucetEnabled && ... && network !== "Mainnet"`), genesis UTXOs
are seeded empty (`config.ts`: `GENESIS_UTXOS: network === "Mainnet" ? [] : genesisUtxos`),
and the mempool network byte switches (`database/mempool.ts`). This is
already two independent layers (infra flag + code gate) — the checklist item
is to exercise `config.test.ts`'s `Mainnet` branch (and extend it if
coverage is thin) as an explicit pre-launch verification, not to add new
gating logic.

---

## 2. Testing Gate

The Master Test Plan and `testing.md` already define the unit / integration
/ e2e / regression / baseline-security / controlled-load layers, and MTP's
own scope statement excludes mainnet launch assessment (§1). What's added
here is specific to the testnet→mainnet transition:

1. **`NETWORK=Mainnet` config-path verification** — see above. Run the
   relevant unit/integration suite with `NETWORK=Mainnet` explicitly, not
   just `Preprod`, and confirm the faucet/genesis/mempool branches behave as
   expected.
2. **Load/soak at mainnet-shaped topology** — the existing controlled-load
   work is captured in
   [`scalability-stress-test-report.md`](scalability-stress-test-report.md)
   (MTP's own controlled-load addendum, §1). Extend or re-run it against a
   staging stack provisioned with the *mainnet* capacity parameters from
   §1.2, not testnet's `t4g.micro`/`m6i.large` sizing — a topology that
   passes at testnet capacity doesn't establish mainnet capacity is correct.
3. **Worker/process resilience under induced failure** — process
   kill/restart, dependency unavailability (RDS, L1 provider), and queue
   backpressure, run against the e2e harness
   (`tests/e2e/redis-stream-ingress.e2e.test.ts`,
   `tests/e2e/tx-ingress-pipeline.e2e.test.ts`) or an equivalent staging
   exercise. Not currently a distinct suite — worth deciding whether it's a
   new automated test tier or a manual game-day exercise before launch.
4. **Reviewed `tofu plan` dry-run against `envs/mainnet.tfvars`** — before
   the first real apply, a second engineer reviews the plan output. No
   automated equivalent exists for testnet either; this is a new gate
   specifically because a mainnet apply's blast radius (real RDS, real
   operator funds) is materially different from testnet's.

---

## 3. Quality & CI Gate

Already enforced on every PR touching `demo/**` (`.github/workflows/`):

| Gate | Workflow | Status |
| --- | --- | --- |
| Format / Lint / Type / Build | `quality.yml` | Required |
| OSV dependency scan | `quality.yml` | Required |
| Dependency audit | `security.yml` | Required |
| Trivy | `security.yml` | Required |
| Gitleaks | `security.yml` | Required |
| CodeQL (`security:codeql:check:node`) | `security.yml` (via `demo` scripts) | Required |
| Semgrep | `security.yml` | **Conditional** — only runs when the `ENABLE_SECURITY_SEMGREP` repo/org variable is set to `true` |

Decisions needed before mainnet, not after:

- Confirm Semgrep is enabled (not left opt-in) for the branch/repo mainnet
  ships from.
- `infra:format:check` (Terraform fmt) runs in `quality.yml`, but there is no
  CI gate that reviews a Terraform *plan diff* — that's the new gate added in
  §2.4, and it should be a required check, not an ad hoc step.
- Add branch protection / required reviewers specifically for
  `infra/aws/terraform/platform/envs/mainnet.tfvars` and
  `backends/mainnet.hcl` once they exist, given they're the actual
  mainnet blast-radius files.

---

## 4. Observability & Operational Readiness

`observability.md` and `aws.md`'s Service Placement table cover what's
provisioned (Prometheus, Loki, Alloy, Grafana, postgres-exporter, cAdvisor,
Tempo) and confirm the same stack is placed for mainnet. What's not yet
established anywhere in the repo:

- **Alert rules and paging.** The current stack is scrape configs and
  dashboards — nothing in `infra/aws/terraform/platform/` or `grafana/`
  defines alert rules or a notification channel. Dashboards being reachable
  is not the same as someone being paged when `/health/ready` starts
  failing. This needs to exist before mainnet traffic, not be added
  reactively after the first incident.
- **Rollback path parity.** Testnet has a scripted service-level rollback
  (`infra:testnet:destroy:node`, dry-run by default). This needs a mainnet
  equivalent once the `infra:mainnet:*` scripts exist (§1.1), plus an
  explicit decision on what "rollback" means for a stateful sequencer (safe
  to destroy/recreate the ECS service; not safe to lose RDS or EFS state).

---

## 5. Cutover Plan

Sequenced from `aws.md`'s bootstrap model and `deployment.md`'s command
matrix, made explicit as a runbook:

1. **Bootstrap** — state, secrets, image (dry-run, then commit).
2. **First apply** — shared resources only, `enable_ecs_services=false`,
   `enable_https_listener=false`.
3. **ACM DNS validation** — add validation records from Terraform output,
   confirm certificate reaches `ISSUED`.
4. **Second apply** — `enable_ecs_services=true`,
   `enable_https_listener=true`.
5. **Final DNS cutover** — `rpc.sundialprotocol.com` and
   `grafana.sundialprotocol.com` CNAMEs to the mainnet ALB.
6. **Verification** — `/health/ready` returns `200`; Grafana reachable per
   the §1.4 access decision; ECS logs show monitoring startup (per
   `deployment.md`'s existing checks).
7. **Rollback trigger** — defined failure conditions (e.g., `/health/ready`
   failing past N minutes, RDS unreachable) and a named decision-maker
   authorized to invoke the mainnet rollback path from §4.

---

## 6. Roles & Sign-off

Extending the role vocabulary already defined in MTP §18 rather than
introducing new titles:

| Role | Responsible for | Name | Date | Approved |
| --- | --- | --- | --- | --- |
| Validation Owner | Confirms MTP entry criteria (this doc's Entry Criteria) are met | | | |
| DevOps / Infrastructure | §1 (AWS parity), §5 (cutover execution) | | | |
| Security Reviewer | §1.3 (key custody), §1.4 (public surface), §3 (CI gates) | | | |
| Engineering | §2 (mainnet-specific testing), §1.5 (demo-code gating) | | | |
| Project / Product Stakeholders | Final go/no-go, §1.4 public-surface decisions | | | |

A launch should not proceed while any row in §1.1 (Infrastructure-as-Code
Gaps) is unresolved, or while any row above is unsigned.

## Related Docs

- [Architecture](architecture.md)
- [Environment](environment.md)
- [Observability](observability.md)
- [Testing](testing.md)
- [Master Test Plan](master-test-plan.md)
- [Scalability & Stress Test Report](scalability-stress-test-report.md)
- [Disaster Recovery Plan](architecture/disaster-plan.md)
- [`sundial-node/docs/aws.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docs/aws.md)
- [`sundial-node/docs/deployment.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docs/deployment.md)
