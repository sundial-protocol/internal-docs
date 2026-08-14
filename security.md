# Security Best Practices

This document is for developers integrating with [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)'s HTTP
API or building directly against [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk) — what to expect, what
isn't provided for you, and what to check on your own side.

It is scoped differently from
[reports/M4.2-Best-Practices.md](./reports/M4.2-Best-Practices.md), which
covers how Sundial secures its own infrastructure (Terraform, AWS Secrets
Manager, network isolation, CI scanning). Read that one if you're operating
or deploying the node; read this one if you're calling it.

## The Trust Model, First

Before anything else: read
[smart-contracts.md § The Demo Runs Against Placeholder Validators](./smart-contracts.md#the-demo-runs-against-placeholder-validators).
In this demo deployment, on-chain script validation enforces none of
Midgard's protocol rules — all correctness is enforced by
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)'s and [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk)'s own off-chain code. Don't
point real value at demo contract addresses, and don't treat "the demo
accepted my transaction" as evidence that the real protocol would too.

## No Built-In Authentication Or Rate-Limiting

With one exception, [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)'s HTTP surface has no
application-level authentication or rate-limiting. This includes the
state-mutating operator endpoints — `/init`, `/commit`, `/merge`, `/reset` —
which have zero access control in the default (`NODE_ROLE=all`)
configuration. Splitting a deployment into `NODE_ROLE=api` /
`tx-processor` / `sequencer` processes removes most of those routes from the
public-facing process (see [api.md § Router Variants](./api.md#router-variants)),
but that's a deployment-topology mitigation, not an authorization control —
anyone who *can* reach an `api`-role node can still call `POST /submit`
without restriction.

The one exception is `POST /faucet/claims`: a static bearer token compared
with plain `!==` (not constant-time) against `FAUCET_API_KEY`, plus
per-address cooldown and per-IP daily limits. Even there, the IP used for
rate-limiting (`ipHash` in the request body) is **client-supplied**, not
derived from the connection server-side — its effectiveness depends entirely
on whatever sits in front of the node actually computing it correctly.

What this means in practice:

- **If you're operating a publicly reachable instance**, put a reverse
  proxy / API gateway / WAF in front of it and do your own auth,
  rate-limiting, and request-size limiting there. Nothing in the application
  does this for you today — including a body-size cap on `POST /submit`.
- **If you're integrating as a client**, don't assume the server is
  throttling other callers on your behalf, and don't assume your own
  request rate is invisible to abuse from other clients sharing the same
  instance.

## Input Validation Patterns To Mirror Client-Side

Validating on your own side before sending a request saves a round trip and
makes failures easier to attribute. The node's own rules (so you can match
them, not just guess at them):

- Transaction hashes: lowercase-or-uppercase hex, exactly 64 characters.
- Block header hashes: hex, exactly 56 characters.
- Addresses: must parse via Lucid's `getAddressDetails` and carry a payment
  credential — a syntactically valid bech32 string with no payment part
  (e.g. a pure stake address) is rejected.
- `POST /submit` bodies: the entire body must be a valid hex string (and,
  separately, must actually deserialize as a Cardano transaction once it
  reaches the queue processor — hex-validity is checked synchronously at
  submit time, transaction-shape validity is not).
- `/txs` pagination: `limit`/`offset` must be non-negative integers;
  `limit` is clamped to 500 server-side regardless of what you request — see
  [api.md § 5](./api.md#5-query-address-transaction-history).

See the handlers in [`sundial-node/src/commands/listen.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/commands/listen.ts) directly if
these rules ever seem to have drifted from this document.

## Error Handling

- A generic, unattributed failure comes back as `500 {"error":"Something
  went wrong"}`. Don't parse this string for anything meaningful — check the
  status code and, where present, a more specific `error` message or `code`
  field (the faucet endpoint's error responses carry a stable `code`; see
  [api.md § 7](./api.md#7-claim-testnet-ada-from-the-faucet)).
- Codec/deserialization errors carry a path to the specific field that
  failed to decode, rather than a bare "decode failed" — useful for
  pinpointing a malformed value in a large transaction or datum without
  bisecting it by hand.
- Calls out to the configured L1 provider (Blockfrost / Kupo+Ogmios) are
  bounded by explicit timeouts in the node's own code (readiness checks,
  commitment-wallet balance queries) rather than left to hang on a slow or
  unreachable provider — but your own client-side calls to [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)
  itself should still set their own timeouts; nothing stops a handler from
  taking a while under load.

## Idempotency And Retries

- `POST /submit` is fire-and-forget onto a durable queue: a `200` means the
  hex was well-formed and accepted onto the stream, with a stream entry `id`
  in the response — it is **not** confirmation that the transaction was
  parsed, validated, or included in a block. Use `GET /tx?tx_hash=...` to
  check final status, and expect to poll.
- The queue is at-least-once, not exactly-once: a transaction that fails
  processing is retried up to a bounded number of attempts before landing on
  a dead-letter stream. Build your own submissions to be safe to process
  more than once (Cardano transactions are naturally idempotent by tx hash —
  resubmitting identical CBOR doesn't double-spend).
- On-chain flows that need a fresh, unused input as an anti-replay nonce
  (deposit, tx-order, and the state-queue's own initialization) follow a
  nonce-UTxO pattern — see `genesis.ts`'s use of `nonceUTxO` for a worked
  example. If you're building these yourself via the SDK, don't reuse a
  nonce UTxO across retries after it's already been spent.
- `POST /faucet/claims` explicitly supports retry-safety via its required
  `idempotencyKey` field — reuse the same key when retrying the same logical
  claim rather than generating a new one.

## Secrets And Config Hygiene

Locally, operator seed phrases and L1 provider keys are plain values in a
`.env` file — see [environment.md](./environment.md) for the full variable
list. That's appropriate for local development but not for anything
resembling a real deployment:

- The AWS deployment path ([`sundial-node/docs/aws.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docs/aws.md)) injects every
  sensitive value — RDS password, operator/genesis/faucet seed phrases,
  provider API keys, Grafana admin password — via AWS Secrets Manager into
  the ECS task, never as a plain task environment variable. See
  [reports/M4.2-Best-Practices.md](./reports/M4.2-Best-Practices.md) for the
  full infra picture.
- If you're standing up your own instance from the `.env.deploy.*.example`
  templates, keep your populated copy out of version control (the repo's
  root `.gitignore` already excludes `.env.deploy.*` — don't narrow that
  pattern) and treat it with the same care as the Secrets Manager values it
  otherwise mirrors.
- Never reuse an operator or genesis seed phrase as a faucet seed phrase (or
  vice versa) — `services/config.ts` explicitly guards against this at
  config-load time and will refuse to start if it detects it.

## Security Tooling Available To Contributors

CI runs gitleaks (secret scanning), Trivy (dependency/container CVEs), and
`pnpm audit` on PRs touching `demo/**`. Semgrep runs conditionally. **CodeQL
does not run in CI** — `demo/scripts/test/security/codeql.sh` exits
immediately under `GITHUB_ACTIONS=true` by design (license constraints), so
it's local-only tooling. Run it yourself before or alongside a PR:

```bash
cd demo
pnpm run security:codeql:check:node
```

Equivalent scripts exist for the other packages
(`security:codeql:check:{ts,sdk,manager}`).

## Hardening Themes

The following are current, ongoing properties of the node and SDK — not a
changelog, just what to expect today:

- **Bounded resource consumption**: list responses (`/txs`) are paginated
  with a server-enforced maximum page size; L1 provider calls and readiness
  checks are timeout-bounded; retry loops use capped, backing-off schedules
  rather than tight unbounded loops.
- **Resilience to malformed input**: a single malformed transaction in the
  submission queue is rejected and retried/dead-lettered without destabilizing
  the worker pool that processes other transactions.
- **Diagnosability**: decode/validation errors are structured enough to
  locate the specific offending field or event type, rather than surfacing
  as an opaque failure.
- **Correct financial checks**: fee-sufficiency and minimum-fee validation
  are computed against a transaction's actual size and content, and
  time-based validity-interval checks compare consistent units throughout.
- **Storage recovery**: the on-disk ledger/mempool Merkle-Patricia-Trie
  store is checked and recovered after an unclean shutdown rather than
  requiring manual intervention.

None of this substitutes for the [trust-model caveat](#the-trust-model-first)
above — it describes the off-chain safety net's current shape, not an
on-chain guarantee.

## Related Docs

- [API](./api.md)
- [Smart Contract Interactions](./smart-contracts.md)
- [Environment](./environment.md)
- [Architecture](./architecture.md)
- [Sundial's Security Practices (infra/ops)](./reports/M4.2-Best-Practices.md)
