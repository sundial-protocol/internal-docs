# Support Runbook — Sundial L2

This is the doc a support responder uses *behind the scenes* when helping a
testnet adopter — as opposed to
[testnet-user-guide.md](./testnet-user-guide.md), which is what gets handed
*to* the adopter. It defines who does support, what they're expected to know,
where they triage, and when they stop and hand off to engineering.

## Purpose

Sundial's adopter-facing surface today is small and specific: a testnet
faucet, a demo dashboard, a CLI, and an HTTP API for
integrators. Most things that go wrong for a new adopter map to a small,
enumerable set of causes — a wallet on the wrong network, a misread error
message, a node that isn't ready yet — and already have a documented
resolution somewhere in this repo. What's been missing is a single place
that tells a support responder *which* doc answers *which* question, what to
say when none of them do, and when a "user problem" is actually an incident.

## Scope

**In scope:** triaging and resolving reports from testnet adopters and early
integrators — faucet claims, wallet/network setup, the demo dashboard, CLI usage, and questions about the HTTP API's documented behavior.

**Out of scope:**

- Infrastructure operations (deploys, rollbacks, cloud incident response) —
  owned by DevOps/Infrastructure per
  [mainnet-readiness.md](./mainnet-readiness.md) and
  [compliance/bcp-dr-rto-rpo.md](./compliance/bcp-dr-rto-rpo.md).
- Protocol design, tokenomics, or roadmap questions beyond FAQ level —
  escalate to Project/Product Stakeholders (MTP §18.6).
- Security vulnerability reports — see
  [Security Reports From Adopters](#security-reports-from-adopters) below;
  these get escalated immediately, not triaged like a normal ticket.
- Anything involving real funds. Sundial's adopter-facing surface today is
  testnet-only (see [testnet-user-guide.md](./testnet-user-guide.md)); if an
  adopter describes a real-value loss, that's a mainnet-support process this
  doc does not yet cover (see [Open Gaps](#open-gaps)).

## Relationship to Other Docs

This runbook adds a triage layer on top of docs that are already the source
of truth for their own subject — it doesn't restate them.

| Topic | Source of truth | This doc adds |
| --- | --- | --- |
| Faucet/wallet/dashboard/CLI walkthrough, exact error text | [testnet-user-guide.md](./testnet-user-guide.md) | What to do when a step in that guide fails for someone |
| HTTP API behavior, endpoint list, faucet error `code`s | [api.md](./api.md) | Mapping API error codes to support responses and escalation |
| Trust model, auth/rate-limit posture, integrator guidance | [security.md](./security.md) | What to tell an integrator who asks about it; what's a real vuln report vs. expected behavior |
| What user/claim data exists and how it's handled | [compliance/data-handling-addendum.md](./compliance/data-handling-addendum.md) | Privacy rules for support staff handling a ticket |
| Node/infra incident response, RTO/RPO | [compliance/bcp-dr-rto-rpo.md](./compliance/bcp-dr-rto-rpo.md) | The point at which a support ticket becomes that workflow |
| Alerting/paging, dashboards | [observability.md](./observability.md), [mainnet-readiness.md](./mainnet-readiness.md#4-observability--operational-readiness) | Read-only tooling access for support to self-serve node status |
| Role vocabulary | [master-test-plan.md](./master-test-plan.md#18-roles-and-responsibilities) | Support-specific tiers layered on top |

## Support Tiers & Roles

Extending the role vocabulary already defined in MTP §18 rather than
introducing a parallel set of titles:

| Tier | Handles | Escalates to |
| --- | --- | --- |
| **L1 — Community Support** | Faucet errors, wallet/network setup, "how do I..." questions answerable from [testnet-user-guide.md](./testnet-user-guide.md) | L2, if the fix isn't in this runbook's [Triage Playbook](#triage-playbook) |
| **L2 — Technical Support** | CLI/API integration questions, reading node status via health endpoints, reproducing a report before filing it | Engineering Team (MTP §18.3) or DevOps/Infrastructure (MTP §18.4) |
| **Engineering / DevOps (on-call)** | Confirmed node/infra faults, anything matching [compliance/bcp-dr-rto-rpo.md](./compliance/bcp-dr-rto-rpo.md#incident-workflow) | Security Reviewer (MTP §18.5), if the fault has a security dimension |

This mirrors a gap called out in
[mainnet-readiness.md §6](./mainnet-readiness.md#6-roles--sign-off): that
table currently has no support-facing row. Adopting this tier structure and
naming an L1/L2 owner closes that gap; see
[Readiness Sign-off](#readiness-sign-off) below.

## Support Channels

Current adopter-facing channels (per Sundial's public community links):

- Discord: `https://discord.gg/9pFR7kp9C`
- X/Twitter: `x.com/sundialprotocol`
- The faucet page itself surfaces most errors inline (see
  [Triage Playbook](#triage-playbook)) — many "tickets" are really
  self-resolving if the adopter rereads the on-page message.

**Open gap:** there is no dedicated support inbox or ticketing tool
(Zendesk/Intercom/a shared mailbox) today — reports arrive ad hoc through
community channels. Until one exists, L1 responders monitoring Discord/X
*are* the intake system, and this runbook's triage tables are what stand in
for a ticketing tool's macros/canned responses.

### Severity & response targets

Proposed starting targets — ratify alongside the sign-off in
[Readiness Sign-off](#readiness-sign-off) rather than treating them as
already committed:

| Severity | Definition | First response | Example |
| --- | --- | --- | --- |
| **P0 — Critical** | Faucet or node fully unavailable for all adopters; a security report | 30 min | `GET /health/ready` failing cluster-wide; a credible vulnerability report |
| **P1 — High** | A whole feature degraded for many adopters | 4 hours | Faucet `DEPLETED` for an extended period; dashboard unreachable |
| **P2 — Normal** | Single-adopter issue with a known cause | 1 business day | Wallet on wrong network, cooldown confusion, a stuck-looking tx |
| **P3 — Low** | Question, doc gap, feature request | Best effort | "Is there a block explorer yet?" |

## New Support Team Member Onboarding Checklist

Before someone is considered "trained and ready" to staff L1/L2 support:

- [ ] Has completed the adopter-side flow themselves end to end: installed a
      wallet, claimed from the faucet, and connected to the demo dashboard,
      following [testnet-user-guide.md](./testnet-user-guide.md) exactly as
      an adopter would.
- [ ] Has built and run the `midgard` CLI locally (`demo/midgard-manager/packages/cli`)
      and successfully created a wallet, funded it, and sent a transaction
      (same doc, [CLI section](./testnet-user-guide.md#sending-your-testnet-sbtc-to-someone-else)).
- [ ] Has read [api.md](./api.md), at minimum the
      [Endpoint Summary](./api.md#endpoint-summary),
      [Client Flow](./api.md#client-flow), and the
      [faucet error `code` table](./api.md#8-claim-testnet-ada-from-the-faucet).
- [ ] Has read [security.md](./security.md) — specifically
      [The Trust Model, First](./security.md#the-trust-model-first) and
      [No Built-In Authentication Or Rate-Limiting](./security.md#no-built-in-authentication-or-rate-limiting)
      — well enough to explain to an integrator why the node has no
      application-level rate limiting without it sounding like news.
- [ ] Has read this runbook's [Triage Playbook](#triage-playbook) and
      [Escalation Matrix](#escalation-matrix) sections.
- [ ] Has read [Data Handling & Privacy](#data-handling--privacy-when-helping-adopters)
      below and knows never to ask an adopter for a seed phrase or private
      key.
- [ ] Knows how to check node status without engineering help: `GET
      /health/live`, `GET /health/ready` (see
      [api.md §1](./api.md#1-check-whether-the-node-is-available)), and, if
      granted access, the Grafana dashboards in
      [observability.md](./observability.md#exposed-local-endpoints).
- [ ] Has shadowed at least one real or simulated ticket of each severity
      with an existing L2 responder before going live solo.

## Triage Playbook

### Step 0 — classify

Before touching a specific issue, decide: is this a single adopter's
mistake (P2/3, work the tables below), or is it affecting multiple
people / matches a [bcp-dr-rto-rpo.md](./compliance/bcp-dr-rto-rpo.md) signal
(P0/1, skip straight to [Escalation Matrix](#escalation-matrix))? A quick
check: `GET /health/ready` against the public testnet node
(`https://sundial-node.testnet.sundialprotocol.com`, per
[testnet-user-guide.md § Testnet network details](./testnet-user-guide.md#testnet-network-details)).
If it's `503`, this is not a one-off — go straight to escalation.

### Faucet errors

Straight from the adopter-visible messages in
[testnet-user-guide.md § Troubleshooting](./testnet-user-guide.md#troubleshooting-faucet-errors),
cross-referenced with the stable `code`s a developer would see calling the
API directly (see [api.md §8](./api.md#8-claim-testnet-ada-from-the-faucet)):

| Adopter sees | API `code` | L1 response | Escalate if |
| --- | --- | --- | --- |
| "Enter a valid Sundial testnet payment address" | `ADDRESS_INVALID` / `ADDRESS_NETWORK_MISMATCH` / `ADDRESS_NO_PAYMENT_CREDENTIAL` / `ADDRESS_SCRIPT` | Confirm their wallet is switched to Testnet/Preprod and they copied a payment (not stake/script) address starting `addr_test1` | Never — this is always adopter-side |
| "This address has already claimed recently" | `COOLDOWN` (429) | Point to the "Next eligible at" time from their last claim, or suggest a different address | Never |
| "This browser has reached the current faucet claim limit" | `IP_LIMIT` (429) | Suggest retrying later or from a different network | If reported by many adopters on unrelated networks around the same time (possible faucet-cap misconfiguration) |
| "The faucet is temporarily out of funds" | `DEPLETED` (503) | Acknowledge, note it's refilled periodically | If it persists beyond a routine refill window → L2/DevOps |
| "The faucet is currently unavailable" | `DISABLED` (404) or unreachable node | Acknowledge, ask them to retry shortly | If it persists more than a few minutes → straight to Escalation Matrix, this is `FAUCET_ENABLED=false` or the node being down, not adopter error |
| A `500` / `VALIDATION_FAILED` / `INTERNAL` shown or reported by an integrator | `VALIDATION_FAILED` / `INTERNAL` (500) | Get the `claimId` if they have one | Always → L2 to reproduce, then Engineering |

### Wallet / network setup

- Address starts `addr1` instead of `addr_test1` → their wallet is still on
  Mainnet; walk them back to
  [Step 1 of the user guide](./testnet-user-guide.md#step-1--switch-your-wallet-to-testnet-and-copy-your-address).
- Balance not updating after a successful claim → confirm they're looking at
  the wallet extension's Testnet/Preprod view, not Mainnet; remind them
  there's no public Sundial L2 block explorer yet during testnet (per the
  user guide), so the wallet balance and the faucet's own confirmation
  screen are the only two places a claim is visible.
- Asking "where's the Send button" → there isn't one in the wallet for
  Sundial L2 sBTC yet; that requires the `midgard` CLI (or a custom
  integration against [api.md §2–3](./api.md#2-build-a-transaction-to-submit)) — point
  them at [testnet-user-guide.md § Sending your testnet sBTC](./testnet-user-guide.md#sending-your-testnet-sbtc-to-someone-else).

### CLI (`midgard`) issues

- `send` fails with a Lucid coin-selection error → the wallet isn't funded
  yet; this is the documented, expected error shape in the user guide's
  [Send funds](./testnet-user-guide.md#send-funds) section, not a bug.
- `send` or `wallet balance` can't connect at all → have them run `midgard
  node node-status` first
  ([reference](./testnet-user-guide.md#if-somethings-not-working)) before
  assuming their own setup is broken. "Live: yes / Ready: no" with a
  `failing` subsystem listed means the *node*, not their CLI, is the
  problem — escalate to L2 with the `failing` subsystem name.
- `tx-lookup` returns "Not found" right after a `send` → expected; `POST
  /submit` only queues the transaction, a background worker processes it
  shortly after (see [api.md §4](./api.md#4-look-up-a-transaction)). Ask
  them to retry in a few seconds before treating it as stuck.

### Dashboard

The demo dashboard is explicitly an early-stage, simulated preview running
against Bitcoin testnet3, separate from the faucet's sBTC balance (see
[testnet-user-guide.md § Explore the testnet dashboard](./testnet-user-guide.md#explore-the-testnet-dashboard)).
Set expectations accordingly — "the numbers look wrong" may just be the
simulation, not a bug worth escalating, unless the dashboard is entirely
unreachable (→ L2).

### Developer / integrator questions

For anyone asking about the HTTP API directly rather than the adopter-facing
web flows:

- Point to [api.md](./api.md) first — it documents the full endpoint set,
  [Router Variants](./api.md#router-variants) (a public `api`-role
  deployment doesn't serve `/tx`, `/txs`, `/utxos`, `/block`), and
  [Error Behavior](./api.md#error-behavior).
- If they ask why there's no auth/rate-limiting on most endpoints, or why
  `always-succeeds.ts` hard-codes `Preprod`, that's documented, expected
  behavior — see [security.md](./security.md#no-built-in-authentication-or-rate-limiting)
  and [api.md § Current Caveats](./api.md#current-caveats). Don't treat it
  as a new finding; if they report it as a *vulnerability* rather than ask
  about it as *documented behavior*, see
  [Security Reports From Adopters](#security-reports-from-adopters).

## Known Issues & Things To Proactively Disclose

Adopters and integrators sometimes report these as bugs. They're documented,
current behavior — acknowledge, don't apologize as if it's a defect, and
link the source doc:

- No public Sundial L2 block explorer yet during testnet
  ([testnet-user-guide.md](./testnet-user-guide.md#step-3--confirm-you-received-your-funds)).
- No wallet-native "Send" for L2 sBTC — the CLI or a custom integration is
  required ([testnet-user-guide.md](./testnet-user-guide.md#sending-your-testnet-sbtc-to-someone-else)).
- On-chain script validation doesn't enforce protocol rules in this
  deployment — off-chain node/SDK code does
  ([security.md § The Trust Model, First](./security.md#the-trust-model-first)).
- No application-level auth or rate-limiting on most endpoints; the faucet
  claim endpoint is the one exception
  ([security.md](./security.md#no-built-in-authentication-or-rate-limiting)).

## Data Handling & Privacy When Helping Adopters

Per [compliance/data-handling-addendum.md § Faucet Claims](./compliance/data-handling-addendum.md#faucet-claims),
a successful faucet claim stores a claim ID, idempotency key, recipient
address, a caller-supplied IP hash, claim amount, tx hash, and claim/cooldown
timestamps.

- **Safe to ask an adopter for, and to share back:** their `addr_test1…`
  address, a `claimId`, or a transaction hash. These are exactly the
  reference IDs the faucet's own success screen shows them
  ([testnet-user-guide.md §2](./testnet-user-guide.md#step-2--request-testnet-sbtc-from-the-faucet)).
- **Never ask for:** a seed phrase or private key, testnet or otherwise. The
  user guide explicitly tells adopters their key material stays local
  ([testnet-user-guide.md](./testnet-user-guide.md#create-a-wallet)); a
  support responder asking for one trains adopters to hand it over to
  whoever asks, which is the opposite of what testnet is supposed to model.
- Testnet assets are valueless by design
  ([testnet-user-guide.md § Safety reminders](./testnet-user-guide.md#safety-reminders)),
  but treat IP hashes and claim data with the same handling discipline
  described in the addendum regardless — it's still adopter-linked data.

## Escalation Matrix

| Signal | Escalate to | Reference |
| --- | --- | --- |
| `GET /health/ready` failing broadly, not just for one adopter | Engineering / DevOps on-call | [compliance/bcp-dr-rto-rpo.md § Incident Workflow](./compliance/bcp-dr-rto-rpo.md#incident-workflow) |
| Faucet `DEPLETED` or `DISABLED` persisting past a routine refill | DevOps/Infrastructure (MTP §18.4) | [api.md §8](./api.md#8-claim-testnet-ada-from-the-faucet) |
| A reproducible `500`/`VALIDATION_FAILED` not explained by adopter error | Engineering Team (MTP §18.3) | [api.md § Error Behavior](./api.md#error-behavior) |
| Anything matching the [BCP/DR Incident Workflow](./compliance/bcp-dr-rto-rpo.md#incident-workflow)'s monitoring signals (stream backlog, blocks stuck, provider errors) | Engineering / DevOps on-call | [compliance/bcp-dr-rto-rpo.md](./compliance/bcp-dr-rto-rpo.md) |
| A security report (see below) | Security Reviewer (MTP §18.5), immediately | [Security Reports From Adopters](#security-reports-from-adopters) |

**Current limitation:** [mainnet-readiness.md §4](./mainnet-readiness.md#4-observability--operational-readiness)
flags that alert rules and paging are not yet defined anywhere in the repo —
dashboards being reachable isn't the same as an on-call engineer being
paged. Until that's resolved, this escalation matrix assumes a support
responder pings the relevant person directly rather than a page firing on
its own; don't assume engineering already knows just because a dashboard
would show it.

### Security Reports From Adopters

There is currently no formal responsible-disclosure process or bug-bounty
program documented in this repo. If an adopter or integrator reports
something that sounds like a genuine vulnerability (not documented,
expected behavior per [Known Issues](#known-issues--things-to-proactively-disclose)
above) — an auth bypass, a way to drain the faucet beyond its documented
limits, unexpected access to another adopter's data:

1. Do not attempt to reproduce it extensively yourself or ask the reporter
   for more detail beyond what they've already volunteered.
2. Escalate directly to the Security Reviewer (MTP §18.5) immediately —
   treat it as P0 regardless of how it presents.
3. Do not discuss it in public community channels until the Security
   Reviewer has triaged it.

## Tooling & Access Checklist

What an L1/L2 responder needs provisioned before going live:

- Read access to this repo (`internal-docs`, at minimum).
- The public faucet page and dashboard, as any adopter would use them.
- A working `midgard` CLI checkout for reproducing CLI-reported issues.
- Read-only Grafana access, if granted, per
  [observability.md § Exposed Local Endpoints](./observability.md#exposed-local-endpoints)
  (or the cloud equivalent) — for self-serving "is the node actually down"
  before escalating.
- Membership in the community channels listed in
  [Support Channels](#support-channels).

## Readiness Sign-off

Mirroring the sign-off pattern in
[mainnet-readiness.md §6](./mainnet-readiness.md#6-roles--sign-off): a
support function shouldn't be declared "trained and ready" while any row
below is unfilled.

| Role | Responsible for | Name | Date | Approved |
| --- | --- | --- | --- | --- |
| L1 — Community Support Lead | Staffing and training L1 responders against this runbook | | | |
| L2 — Technical Support Lead | Staffing L2, owning the Triage Playbook's accuracy | | | |
| Engineering Team | Confirming the Escalation Matrix's technical signals are current | | | |
| Project / Product Stakeholders | Approving severity/response-time targets in [Support Channels](#support-channels) | | | |

## Open Gaps

Carried forward honestly rather than implied as solved:

- No dedicated support inbox/ticketing tool — community channels are the
  intake system today.
- No formal responsible-disclosure process for security reports.
- No mainnet adopter-support process yet — this runbook is scoped to the
  testnet surface described in
  [testnet-user-guide.md](./testnet-user-guide.md); revisit before mainnet
  alongside [mainnet-readiness.md](./mainnet-readiness.md).
- Escalation currently relies on a person, not a page — see the note in
  [Escalation Matrix](#escalation-matrix) tied to
  [mainnet-readiness.md §4](./mainnet-readiness.md#4-observability--operational-readiness).

## Review Cadence

Review this runbook whenever
[testnet-user-guide.md](./testnet-user-guide.md) or
[api.md](./api.md)'s adopter-facing behavior changes, and at minimum each
time [mainnet-readiness.md](./mainnet-readiness.md) is revisited for a
launch decision.

## Related Docs

- [Testnet User Guide](./testnet-user-guide.md)
- [API](./api.md)
- [Security Best Practices](./security.md)
- [Data Handling Addendum](./compliance/data-handling-addendum.md)
- [BCP/DR Plan with RTO/RPO](./compliance/bcp-dr-rto-rpo.md)
- [Observability](./observability.md)
- [Mainnet Readiness Review](./mainnet-readiness.md)
- [Master Test Plan](./master-test-plan.md)
- [Disaster Recovery Plan](./architecture/disaster-plan.md)
