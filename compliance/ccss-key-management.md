# CCSS Key Management Readiness - Sundial Node

Generated: 2026-08-17

## Purpose

This document maps the current and target key-management posture for
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)
to the CryptoCurrency Security Standard (CCSS) v9.0.

It is a public-facing CCSS-mapped key-management determination for the current
testnet deployment and the stricter mainnet operating model. It is not a CCSS
certification or auditor attestation. Testnet currently uses environment-backed
signing configuration suitable for non-production operation; mainnet operation
requires a stricter custody path, stronger access controls, and supporting
operational evidence before any CCSS level can be represented.

## CCSS Reference

CCSS is maintained by the CryptoCurrency Certification Consortium (C4). C4
describes CCSS as a standard for cryptocurrency systems and notes that systems,
not whole companies, are certified against Level 1, Level 2, or Level 3. The
official overview also states that CCSS covers ten security aspects across
Cryptographic Asset Management and Operations, and that the lowest result
across the ten aspects determines the overall level.

Primary references reviewed for this document:

- C4 CCSS overview:
  <https://cryptoconsortium.org/cryptocurrency-security-standard-documentation/overview/>
- C4 CCSS v9.0 table:
  <https://cryptoconsortium.org/ccss-table-v9/>
- C4 FAQ on CCSS levels:
  <https://cryptoconsortium.org/ccss-posts/how-do-ccss-levels-work/>

This document records the current control state, the target mainnet control
state, and the gap between them. Any CCSS level claim depends on the mainnet
control set being implemented and evidenced.

## Scope

In scope:

- `demo/midgard-node` runtime roles: `api`, `tx-processor`, `sequencer`, and
  `all`.
- Operator L1 signing material used by the node:
  `L1_OPERATOR_SEED_PHRASE`,
  `L1_OPERATOR_SEED_PHRASE_FOR_BLOCK_COMMITMENT`, and
  `L1_OPERATOR_SEED_PHRASE_FOR_MERGE_TX`.
- Mainnet deployment and AWS runtime boundaries that affect the above keys.
- Testnet/demo-only signing material where it appears in mainnet deployment
  examples or runtime configuration.
- L1 provider credentials where their compromise affects signing or chain-state
  decisions.
- Key compromise response, access control, audit evidence, and operational
  monitoring for the CCSS trusted environment.

Out of scope:

- End-user wallet custody.
- Internal security of Blockfrost, Kupo, Ogmios, AWS, or other third-party
  service providers.
- Formal CCSS certification.
- Smart-contract audit conclusions, except as a dependency called out under
  CCSS aspect 2.01.
- Entity-wide SOC 2, ISO 27001, legal, HR, or finance controls that do not
  touch the `demo/midgard-node` CCSS trusted environment.

Related repository documents:

- [Mainnet Readiness Review](../mainnet-readiness.md)
- [Security Best Practices](../security.md)
- [Threat Model (STRIDE) - Midgard Node](threat-model-stride.md)
- [BCP/DR Plan with RTO/RPO - Sundial Node](bcp-dr-rto-rpo.md)
- [Data Handling Addendum](data-handling-addendum.md)
- [Smart Contract Interactions](../smart-contracts.md)

## Executive Summary

Current determination: Sundial has a documented testnet key-management posture
and a defined mainnet custody target, but it is not in a CCSS-claimable
mainnet state.

The AWS deployment path improves over a local `.env` file by injecting
sensitive values through AWS Secrets Manager. That is appropriate for the
current testnet posture. Mainnet custody requires documented signer isolation,
role-scoped runtime access, key-holder workflows, backup controls, and
key-compromise procedures.

Mainnet control findings:

| ID | Finding | CCSS impact | Evidence |
| --- | --- | --- | --- |
| CCSS-B1 | Mainnet deployment example still mirrors the testnet-style seed-phrase placeholders for operator and genesis wallets. | Mainnet custody is not yet separated cleanly from testnet/demo signing material. | `demo/midgard-node/.env.deploy.mainnet.example` includes `L1_OPERATOR_SEED_PHRASE*` and `TESTNET_GENESIS_WALLET_SEED_PHRASE_A/B/C`. |
| CCSS-B2 | Current signing flow is application-managed, which is acceptable for testnet/demo operation. | Mainnet signing isolation is not yet at the intended custody standard. | `demo/midgard-node/src/services/config.ts` reads seed phrases; `demo/midgard-node/src/services/lucid.ts` selects wallets from seeds. |
| CCSS-B3 | Role separation exists at process level, but the AWS task definition injects signing secrets into the single node task. | API compromise can become signing compromise unless mainnet deploys split roles with separate secrets/IAM. | `demo/midgard-node/infra/aws/terraform/platform/task_sundial_node.tf` injects all operator seeds into `sundial-node`. |
| CCSS-B4 | Mainnet key ceremony, key-holder onboarding/offboarding, backup controls, and Key Compromise Policy need to be finalized before launch. | Mainnet operating evidence for a CCSS-aligned custody program is incomplete. | Current docs call out key custody as a mainnet readiness workstream; this document defines the operating control set. |

Current CCSS determination:

- The current testnet posture is documented and bounded.
- The current mainnet posture is not CCSS-claimable.
- The current documented target aligns most naturally with a Level 1-capable
  mainnet custody program once the blocking remediations in this document are
  completed and evidenced.
- Institutional mainnet operation aligns more closely with a Level 2 design
  target, including threshold or equivalent signer isolation for material-value
  wallets and documented approved communication channels.
- Level 3 remains a future operating-maturity state rather than a current or
  near-term representation.

## Current Key Inventory

This inventory records key classes and secret classes currently visible in the
repo. It is not a list of real secret values.

| Class | Current variable / secret name | Purpose | Mainnet classification | Current status |
| --- | --- | --- | --- | --- |
| Operator general signing key | `L1_OPERATOR_SEED_PHRASE` / `operator-seed-phrase` | General L1 operator wallet used by node workflows. | Critical signing material | Environment-backed in testnet; mainnet requires an approved custody path or a tightly scoped documented exception. |
| Block commitment signing key | `L1_OPERATOR_SEED_PHRASE_FOR_BLOCK_COMMITMENT` / `operator-seed-phrase-block-commitment` | Signs block-commitment transactions. | Critical signing material | Environment-backed in testnet; mainnet requires isolated signer or threshold custody where feasible. |
| Merge transaction signing key | `L1_OPERATOR_SEED_PHRASE_FOR_MERGE_TX` / `operator-seed-phrase-merge` | Signs state-queue merge transactions. | Critical signing material | Environment-backed in testnet; mainnet requires isolated signer or threshold custody where feasible. |
| Testnet genesis wallets | `TESTNET_GENESIS_WALLET_SEED_PHRASE_A/B/C` / `testnet-genesis-wallet-seed-phrase-a/b/c` | Seeds demo/test L2 ledger UTxOs. | Testnet/demo only | Still present in mainnet example and task definition. Application gates `GENESIS_UTXOS` to empty on `NETWORK=Mainnet`; mainnet deployment omits these secrets from production scope. |
| Faucet wallet | `FAUCET_SEED_PHRASE` / `faucet-seed-phrase` | Testnet faucet payouts. | Testnet/demo only | Config prevents faucet genesis funding on mainnet; mainnet Terraform keeps faucet disabled and omits faucet secrets. |
| Faucet API key | `FAUCET_API_KEY` / `faucet-api-key` | Server-to-server faucet claim authentication. | Testnet/demo only | Demo/testnet only. |
| L1 provider credentials | `L1_BLOCKFROST_KEY`, `L1_OGMIOS_KEY`, `L1_KUPO_KEY` | Chain read/submission provider access. | Sensitive non-signing secret | Stored via Secrets Manager in AWS; rotation and failover policy still required. |
| Database/admin secrets | `POSTGRES_PASSWORD`, `GRAFANA_ADMIN_PASSWORD` | Data plane and observability access. | Sensitive non-signing secret | Stored via Secrets Manager in AWS; access logging and rotation policy required. |

Mainnet policy determination: real mainnet operator signing material is
handled through the approved custody path, not entered into `.env.deploy.mainnet`,
committed to a local file, pasted into CI, or exposed to public/API role
containers.

## Target Mainnet Key-Custody Architecture

The target architecture is the intended mainnet operating model.

### Custody Model

Mainnet signing moves from application-held seed phrases to one of the
following approved patterns:

| Pattern | When acceptable | Required properties |
| --- | --- | --- |
| External threshold signer / MPC provider | Preferred for institutional operation and material-value wallets. | No complete private key or seed phrase is available to the node. Signing policy enforces role, destination, amount, network, and transaction type. Provider is assessed as a service provider under CCSS aspect 2.03. |
| HSM-backed signer service | Acceptable where Cardano signing support and transaction-policy checks are available. | Private key remains non-exportable. Node receives only a signer reference and authorization credential. Signing operations are logged. |
| Cardano multi-signer / script-controlled wallet | Acceptable where the protocol flow supports the required script and latency profile. | No one signer can unilaterally move material funds. Key shards or signer devices are distributed across distinct operators. |
| Temporary hot-key exception | Only for a time-boxed, low-value pilot approved in writing. | Dedicated limited-funds key, explicit expiry, documented risk acceptance, no institutional custody claim, and migration plan to threshold/HSM/multisig custody. |

The default mainnet design is threshold/MPC or HSM-backed signing for
the three operator key classes. If protocol constraints require hot signing for
commitment liveness, the hot signer must hold only the minimum value required
for operational fees and must be isolated from public ingress.

### Runtime Shape

Mainnet runtime is split by role:

| Runtime role | Signing material allowed? | Required controls |
| --- | --- | --- |
| `api` | No | Public ingress only, no operator seed/signing capability, no write access to signer credentials. |
| `tx-processor` | No | Queue processing only, no operator seed/signing capability, least-privilege database access where feasible. |
| `sequencer` | Yes, through signer reference only | Private network placement, signer authorization, transaction-policy checks, audit logging, rate limits, alerting. |
| Break-glass operator workflow | Controlled | Requires approved communication channel, dual approval for material actions, incident ticket, and post-event review. |

The application does not receive `L1_OPERATOR_SEED_PHRASE*` values in
mainnet. It receives signer configuration such as signer endpoint,
key identifier, policy identifier, and short-lived authorization credentials.

### Transaction Policy

The signer or custody layer must reject transactions that violate approved
policy before signing:

- network must be Cardano mainnet only for mainnet keys;
- destination/script addresses must match the approved protocol deployment;
- transaction purpose must match the key role, such as commitment, merge, or
  operator administration;
- amount and fee bounds must be enforced;
- high-risk actions require dual approval through the approved communication
  channel;
- signed transaction hash, policy decision, requester identity, and approval
  evidence must be logged.

## CCSS Aspect Readiness Matrix

Status values:

- `Ready`: current controls and evidence appear sufficient for the stated
  target, subject to audit.
- `Partial`: useful controls exist, but material gaps remain.
- `Open`: current testnet controls need additional mainnet hardening or
  operating evidence.
- `N/A`: not applicable to this scoped system.

Status values reflect the evidence currently recorded for the testnet system
and the additional controls expected for mainnet.

| CCSS aspect | Current status | Current evidence | Mainnet target | Required remediation |
| --- | --- | --- | --- | --- |
| 1.01 Key Material Generation | Open | Current testnet path expects signing configuration to be supplied to the node. Mainnet requires a documented key-generation ceremony or custody-provider process. | Documented generation ceremony using approved hardware/software, sufficient entropy, actor-generated key material, and signed runbook. | Create ceremony runbook; assign roles; validate generation tooling; record entropy and environment checks; prohibit one actor generating another actor's key except for approved automated signer setup. |
| 1.02 Wallet Generation | Open | Current model uses separate signing roles for operator, block commitment, and merge keys. Mainnet wallet policy, custody pattern, redundancy, and signer assignment need to be finalized. | Wallet generation policy with explicit single-signer vs threshold/multisig rationale per wallet. Material-value wallets use threshold/multisig where protocol-compatible. | Define wallet classes; choose custody pattern; record approved addresses/scripts; document recovery redundancy and geographic distribution for threshold keys. |
| 1.03 Key Material Storage | Partial | AWS injects secrets through Secrets Manager for deployment. Mainnet should further reduce seed exposure and document backup controls. | No seed phrases in app env; keys non-exportable or threshold-held; encrypted storage and documented backups with environmental, access-control, and tamper-evidence protections. | Replace seed env vars with signer references; define backup media/location/access process; define tamper-evidence log; remove testnet genesis secrets from mainnet scope. |
| 1.04 Key Material Access | Partial | AWS IAM can scope Secrets Manager access, but there is no key-holder onboarding/offboarding checklist, approved channel, or grant/revoke audit trail. | Least-privilege access with named roles, MFA, approved channels, and auditable grant/revoke workflows. | Write key-holder access checklist; define primary/secondary approvers; require MFA; log all access grants/revocations; review access before mainnet and at least quarterly. |
| 1.05 Key Material Usage | Open | Testnet signing is application-managed. Mainnet should use stronger signer isolation and transaction approval policy. | Key material used only inside the trusted custody environment. Node requests signatures without seeing seed phrases or private keys. | Implement signer abstraction; enforce transaction policy; split runtime roles; remove signing capability from API and tx-processor; add spend verification for high-risk transactions. |
| 1.06 Data Sanitization Documentation | Open | Media/key sanitization policy for key-bearing systems needs to be finalized for mainnet. | NIST SP 800-88-aligned sanitization/destruction policy for any media or system that handled key material. | Create sanitization runbook; identify key-bearing media; define decommissioning evidence; maintain audit trail for sanitized media. |
| 2.01 Security Tests / Audits | Partial | CI/security docs mention gitleaks, Trivy, Semgrep, audit tooling, and local CodeQL. No independent CCSS, penetration, infrastructure, or production smart-contract audit evidence is recorded here. | Independent security assessment before mainnet claim; smart-contract audit evidence for deployed contracts; vulnerabilities risk-accepted or remediated before launch. | Schedule third-party review; attach reports; track remediation; ensure deployed validator version matches audited version. |
| 2.02 Log and Monitor | Partial | CloudWatch, Prometheus, Loki, Alloy, Grafana, readiness checks, metrics, and dead-letter records exist. Authenticated principal logging and one-year retention are not established. | Audit trails for signing, privileged operations, config changes, key access, chain submission, and anomalous blockchain state, with defined retention and alerting. | Define log retention; enable signer/audit logs; alert on key access, signer denials, abnormal submit failures, provider lag, and unexpected wallet movement. |
| 2.03 Governance and Risk | Partial | STRIDE threat model and mainnet readiness review exist. Executive owner, formal risk-management process, and service-provider due diligence should be finalized for mainnet. | Named executive owner, risk register, risk acceptance process, service-provider review for AWS/provider/custody vendor. | Assign owners; create risk register; review MPC/HSM/provider vendors; document annual review cadence. |
| 2.04 Key Compromise Documentation | Open | Mainnet readiness identifies the break-glass workstream; key inventory review, Key Compromise Policy, approved communication channel, and rehearsal should be completed before launch. | Key Compromise Policy covering each key class, roles, secondary actors, communication channels, containment, rotation/migration, and rehearsal. | Write KCP; link inventory; define suspected compromise triggers; rehearse before mainnet; schedule annual test. |

Because CCSS levels are cumulative and the overall score is limited by the
weakest applicable aspect, the items above should be treated as preconditions
for any mainnet CCSS level claim.

## Mainnet Policy Requirements

The following policy requirements become mandatory before `demo/midgard-node`
is presented as mainnet CCSS-ready.

### 1. Key Generation

1. Each mainnet operator signing key must be generated through an approved
   ceremony or custody-provider process.
2. The ceremony must identify participating roles, physical/logical location,
   equipment used, software or hardware versions, entropy source, and
   verification steps.
3. The actor or signer component that will use key material must generate it,
   except where an automated signing agent is initialized through a secure KMS
   process that includes secure transfer and removal from the generation
   environment.
4. Ceremony participants must sign the runbook after each completed step.
5. Testnet genesis, faucet, or development keys must never be reused for
   mainnet operation.

### 2. Wallet Generation And Assignment

1. The security owner must approve a wallet register before mainnet bootstrap.
2. Each wallet entry must include purpose, network, signer/custody provider,
   approved addresses or scripts, spending policy, backup/recovery model, and
   retirement process.
3. Material-value wallets should use threshold/MPC, HSM-backed non-exportable
   keys, or protocol-compatible multisig.
4. Any single-signer wallet must have written risk acceptance, spend limits,
   funding limits, and a migration plan.

### 3. Storage And Backup

1. Mainnet signing seed phrases must not be stored in `.env` files, CI
   variables, issue trackers, chat systems, or general-purpose password
   managers.
2. The production node must not receive full seed phrases for mainnet signing.
3. If exported backup material exists, it must be encrypted, access controlled,
   protected against environmental risk, and tamper evident.
4. Backup inventory must record medium, location, custodian role, creation
   date, tamper-evidence identifier, and last verification date.
5. Backup verification must never expose enough material for one person to
   reconstruct a signing key.

### 4. Access Control

1. Access to production signing capability requires named user identity and
   MFA.
2. Key-holder onboarding and offboarding must follow a checklist approved by a
   security reviewer.
3. Grant, revoke, and emergency access requests must occur over approved
   communication channels.
4. Human access to signer administration, AWS Secrets Manager, Terraform state,
   production CI deploy roles, and production logs must be reviewed at least
   quarterly.
5. API and tx-processor runtime roles must not have access to signer
   credentials or seed material.

### 5. Key Usage

1. Signing must occur only through the approved signer/custody environment.
2. The signer must enforce transaction policy before producing a signature.
3. The node must log signing requests by role, key identifier, transaction hash,
   decision, and request correlation ID.
4. High-risk operations, including key rotation, signer-policy change, recovery
   action, or movement of material value, require dual approval.
5. Failed, denied, timed out, or policy-exception signing attempts must alert
   operators.

### 6. Data Sanitization

1. Any machine, disk, removable media, backup device, or CI runner that handled
   key material must be treated as key-bearing media.
2. Key-bearing media must be sanitized or destroyed according to a documented
   NIST SP 800-88-aligned procedure.
3. Sanitization records must identify the media, method, operator, reviewer,
   date, and result.

### 7. Monitoring And Evidence

1. Mainnet signer events, key access, deployment changes, privileged API calls,
   provider failures, chain submission outcomes, and suspicious wallet movement
   must be logged.
2. Production audit logs must have an approved retention period. For Level 2
   readiness, target at least one year unless legal or architecture review sets
   a stricter requirement.
3. Alerts must have named responders, escalation rules, and response-time
   objectives.
4. Evidence for each control must be stored in the compliance evidence folder
   or an approved GRC/evidence system with immutable history.

## Key Compromise Policy Outline

This section is the current policy framework for a mainnet Key Compromise
Policy. A fully operational runbook remains a launch-gating artifact.

### Compromise Triggers

Invoke the Key Compromise Policy when any of the following occurs:

- confirmed or suspected exposure of operator seed material, signer admin
  credentials, signer API credentials, or custody-provider credentials;
- unexpected signer request, signer denial, or signed transaction;
- tamper-evident backup seal mismatch or missing backup;
- unexplained change in approved signer policy;
- compromise of ECS task, CI deploy role, AWS IAM principal, Terraform state,
  production Secrets Manager path, or custody-provider account;
- disappearance or unavailability of a key holder where that affects recovery
  or spend authority;
- provider or chain-state anomaly that may have caused an unauthorized or
  incorrect signing decision.

### Severity Classes

| Severity | Condition | Immediate action |
| --- | --- | --- |
| Critical | Signing key or threshold quorum may be compromised; unauthorized transaction may have been signed. | Freeze signing if possible, revoke signer credentials, convene incident roles, verify chain state, begin wallet migration/rotation. |
| High | Signer admin or production IAM access may be compromised, but no signature evidence yet. | Revoke suspect access, rotate auth credentials, review signer logs, increase monitoring, prepare wallet migration. |
| Medium | Provider key, logging access, or non-signing secret compromised. | Rotate affected secret, review access logs, validate no signer policy change occurred. |
| Low | Process deviation without evidence of exposure. | Record exception, review cause, remediate control gap. |

### Incident Roles

| Role | Responsibility | Secondary |
| --- | --- | --- |
| Incident Commander | Owns incident timeline, approvals, and communication discipline. | Security Owner |
| Security Owner | Classifies compromise, authorizes containment and rotation path. | Executive Sponsor |
| Key Custody Operator | Executes signer freeze, policy update, key rotation, or wallet migration steps. | Backup Key Custody Operator |
| Chain Operations Lead | Verifies L1 state, pending submissions, commitments, merges, and wallet balances. | Protocol Engineer |
| Infrastructure Lead | Revokes IAM/Secrets/CI access and preserves logs. | DevOps Engineer |
| Communications Owner | Sends approved internal/external notices when required. | Executive Sponsor |

### Required Steps

1. Open an incident record with time, reporter, suspected key class, and known
   blast radius.
2. Switch all incident coordination to the approved communication channel.
3. Freeze or restrict signer policy where technically possible.
4. Preserve signer logs, CloudWatch/Loki logs, ALB access logs, CI audit logs,
   Terraform state history, and custody-provider audit records.
5. Determine whether any unauthorized transaction was signed, submitted, or
   confirmed.
6. Rotate non-signing credentials immediately if they are in scope.
7. For signing-key compromise, migrate authority to fresh approved keys or
   signer policy according to the protocol-specific recovery plan.
8. Reconcile `BlocksDB`, state queue, commitments, merges, and wallet balances
   before returning to normal service.
9. Record root cause, customer/user impact, funds impact, remediation, and
   follow-up owners.
10. Conduct a post-incident review and update this policy, the STRIDE threat
    model, and operational runbooks.

## Remediation Plan

| Priority | Remediation | Owner | Evidence required | Gate |
| --- | --- | --- | --- | --- |
| P0 | Decide and document mainnet custody pattern: MPC, HSM-backed signer, protocol multisig, or approved temporary hot-key exception. | Security Owner / Protocol Lead | Approved custody design record | Before mainnet bootstrap |
| P0 | Remove `TESTNET_GENESIS_WALLET_SEED_PHRASE_A/B/C` from mainnet examples, mainnet Terraform secret requirements, and any mainnet secret path. | Node Owner | PR diff and config test evidence | Before mainnet bootstrap |
| P0 | Replace mainnet `L1_OPERATOR_SEED_PHRASE*` env contract with signer references, or document a time-boxed exception with funding limits. | Node Owner / Security Owner | PR diff, deployment plan, risk acceptance if exception | Before mainnet bootstrap |
| P0 | Split mainnet runtime so public/API roles do not receive signer access. | Infrastructure Owner | Terraform/task definitions, IAM policy review, deployed task inspection | Before public mainnet endpoint |
| P0 | Create mainnet key-generation and wallet-generation runbooks. | Security Owner | Signed ceremony template and wallet register | Before key creation |
| P0 | Create Key Compromise Policy and approved communication channel list. | Security Owner | Approved KCP, rehearsal date | Before funding operator wallets |
| P1 | Implement signer transaction-policy checks for network, purpose, address/script, fee/amount bounds, and approval requirements. | Node Owner / Custody Owner | Unit/integration tests and signer audit logs | Before material value |
| P1 | Define key backups, tamper evidence, access control, and verification process. | Security Owner | Backup inventory template and completed dry run | Before material value |
| P1 | Define signer and key-access logging retention, alerting, and reviewer cadence. | Infrastructure Owner | Log retention config, alert rules, runbook | Before material value |
| P1 | Schedule independent security assessment and smart-contract audit evidence review. | Executive Sponsor | Assessment scope, report, remediation tracker | Before compliance claim |
| P2 | Build formal risk register and service-provider due-diligence record for AWS, L1 provider, and custody provider. | Security Owner | Risk register and vendor review records | Before institutional review |
| P2 | Run key-compromise tabletop and backup-recovery rehearsal. | Security Owner / Ops Lead | Exercise report, findings, remediation | Before institutional review |
| P2 | Define annual reassessment cadence for CCSS controls. | Executive Sponsor | Compliance calendar | Before certification audit |

## Evidence Checklist For Future Audit

Collect these artifacts before asking an assessor or institutional reviewer to
treat the system as CCSS-ready:

- Approved scope statement for the `demo/midgard-node` CCSS trusted
  environment.
- Final key inventory with key IDs, roles, wallet addresses/scripts, custody
  provider, and criticality.
- Key-generation ceremony records and participant sign-offs.
- Wallet-generation policy and approved wallet register.
- Signer/custody architecture diagram and threat model update.
- Terraform/task definition evidence showing API and tx-processor roles do not
  receive signer capability.
- IAM and custody-provider access review evidence.
- Key-holder onboarding/offboarding checklists.
- Backup inventory, tamper-evidence records, and backup verification records.
- NIST SP 800-88-aligned sanitization procedure and any completed sanitization
  records.
- Signer transaction-policy configuration and test evidence.
- Signed transaction audit logs with retention configuration.
- Alerts for signer access, signer denial, suspicious wallet movement, provider
  lag, and failed chain submission.
- Key Compromise Policy and tabletop/rehearsal report.
- Independent security assessment, smart-contract audit report, and remediation
  tracker.
- Service-provider due-diligence records for custody provider, AWS, and L1
  provider.
- Executive security owner acknowledgement and risk acceptance records.

## CCSS Determination

Sundial has a documented CCSS-mapped key-management framework for
`demo/midgard-node`.

Sundial is not CCSS certified, and `demo/midgard-node` is not currently in a
CCSS-claimable mainnet custody state.

The current testnet deployment uses AWS Secrets Manager and environment-backed
signing configuration. The documented mainnet control state requires approved
signer isolation, role separation, backup control, access governance, and a
fully defined key-compromise process before any CCSS level representation is
made.
