# Reliability Reporting

How to produce a **retrospective reliability report** for
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node):
a dated, point-in-time SLO-compliance report over a closed calendar window
(for example "August 2026 testnet operation"), rendered to an industry-standard
format and reproducible from a frozen evidence bundle.

This is **not** a live status page. It is a signed artifact: given the same
evidence bundle, anyone can re-render byte-identical numbers offline.

## Relationship to other documents

| Topic | Source of truth | This doc adds |
| --- | --- | --- |
| Metric names, labels, retention, SLO rules | [`telemetry.md`](telemetry.md) | Nothing — reference only |
| Local/cloud observability stack, Prometheus snapshot | [`observability.md`](observability.md) | Nothing — reference only |
| Load-campaign benchmark reporting, public panel allowlist, conclusion format | [`scalability-stress-test-report.md`](scalability-stress-test-report.md) §14, §19 | The calendar-window (not load-run) reporting mode |
| Log-based stability & security evidence | [`system-log-assurance-report.public.md`](system-log-assurance-report.public.md) | This report is the metrics/SLO companion the log report cross-checks against |
| SLI / SLO definitions | [`sundial-node/slo/slo.json`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/slo/slo.json) | Nothing — reference only |

## What it reports

Framing: Google SRE SLI/SLO + error budget, applied retrospectively over the
closed window; four golden signals / RED for the request path.

- **SLO compliance** — the seven SLIs from `slo/slo.json` (submit-ingress
  availability and latency, end-to-end tx success, L1 commitment success, merge
  success, inclusion latency, node scrape availability), each as a single
  whole-window aggregate evaluated at the window end, against its objective.
- **Transaction success rate** — ingress / end-to-end / L1 / merge ratios with
  the failure-counter breakdown.
- **Network stability** — `up`-based availability and downtime, restart count
  (from `midgard_node_start_time_seconds` / `up` gaps), block-cadence mean and
  coefficient of variation, stream backlog/lag envelopes, container CPU/memory
  headroom, L1 commitment fees.
- **Incident log** — auto-detected `up==0` gaps, restarts, failure-counter step
  increases and rolling-window SLO breaches, with start/end/duration/severity
  and (internal report) Loki error/warn excerpts.
- **Error-budget accounting** — budget, consumed and remaining per ratio SLI.
- **Data provenance** — the evidence bundle manifest and the exact PromQL used.

The report ends with a **disposition**, matching
[`scalability-stress-test-report.md`](scalability-stress-test-report.md) §19:

| Disposition | Meaning |
| --- | --- |
| `Passed` | Every SLI met its objective with data. |
| `Passed with Observations` | All SLIs with data met their objective; at least one had no data in the window. |
| `Failed` | At least one SLI missed its objective. Drives a non-zero CLI exit code. |

## Window semantics

- **All times are UTC.** The window is a **half-open interval** `[from, to)`.
- `--month 2026-08` expands to `2026-08-01T00:00:00Z` (inclusive) →
  `2026-09-01T00:00:00Z` (exclusive).
- `--from` / `--to` accept `YYYY-MM-DD` (midnight UTC) or a full ISO-8601
  timestamp. `--to` must be strictly after `--from`.
- Report **only closed windows** — run after the window has fully elapsed, so
  the last buckets are complete. Do not report a partial current month.
- The window must sit inside the retention horizon (currently 120d for
  Prometheus and Loki — see [`telemetry.md`](telemetry.md#retention--persistence)).
  In the cloud, history only exists from the `tofu apply` that moved Prometheus
  and Loki onto EFS; windows before that point have no data.

## Cadence

- **Monthly**, once the calendar month has closed — the default cadence, aligned
  with the `--month` mode.
- **Ad hoc** for a specific window on request (an incident post-mortem, a
  partner review, a milestone).
- A first *interim* report can be produced from existing metrics as soon as
  retention is in place, before a full month of the newer SLIs has accumulated;
  note the shortened baseline in the report's limitations.

## How to run

The generator is the `reliability-report` sub-command of the scalability
harness. SLIs and objectives come from `demo/midgard-node/slo/slo.json` — the
same source as the Prometheus recording rules — so the report cannot drift from
the rules or the alerts.

```bash
cd demo/midgard-manager/packages/scalability-harness
npm run build

# A full calendar month against the testnet Prometheus (+ Loki for the log excerpt)
npm run start -- reliability-report --month 2026-08 --env testnet \
  --prometheus https://<prometheus-host> \
  --loki https://<loki-host> \
  --grafana https://<grafana-host>

# An explicit window
npm run start -- reliability-report \
  --from 2026-08-01 --to 2026-08-15T00:00:00Z \
  --env testnet --prometheus http://localhost:9090
```

Useful options: `--slo <path>` (defaults to `demo/midgard-node/slo/slo.json`),
`--output-dir <dir>` (defaults to `./reliability-reports`), `--rolling-window`
(breach-detection window, default `1h`), `--step` (range-query step in seconds,
default `60`).

Output goes to `<output-dir>/<env>-<window>/`.

### Re-rendering from a frozen bundle

```bash
npm run start -- regen-reliability-report ./reliability-reports/testnet-2026-08
```

Re-runs analyze → incidents → charts → render from `collected.json` with **no
network calls**, and refreshes `MANIFEST.sha256`. Use this to regenerate the
document after a template change, or to verify a bundle reproduces.

## Evidence bundle: retention & integrity

Each run writes a self-contained bundle:

| File | Contents |
| --- | --- |
| `reliability-report.md` / `.html` | Internal report (full detail, endpoints, PromQL, Loki excerpt). |
| `reliability-report.public.md` | Redacted public-safe variant (see below). |
| `collected.json` | Raw Prometheus range/instant query results + Loki excerpt — the inputs everything else is derived from. |
| `analysis.json`, `incidents.json` | Computed model: SLO results, availability, cadence, incidents. |
| `slo.json` | Copy of the SLO source as it was at run time. |
| `run-manifest.json` | Harness version, git SHA, window, endpoints, step, rolling window, disposition, `generatedAt`. |
| `charts/*.svg` | Vega-rendered charts embedded in the report. |
| `MANIFEST.sha256` | SHA-256 of every other file in the bundle. |

Integrity and retention:

- Verify a bundle with `sha256sum -c MANIFEST.sha256` (or
  `shasum -a 256 -c`) from inside the bundle directory.
- Store the **whole bundle** — not just the rendered report — in the internal
  evidence store, keyed by `<env>/<window>`. The rendered report alone is not
  evidence; `collected.json` + `MANIFEST.sha256` are.
- Retain bundles indefinitely (they are small — range-query JSON and SVGs).
  They outlive the Prometheus retention horizon, so an old window stays
  reproducible via `regen-reliability-report` after the raw TSDB data has aged
  out.
- Optionally attach a Prometheus TSDB snapshot taken at the window end (see
  [`observability.md`](observability.md#prometheus-snapshot)) for windows that
  warrant a frozen datastore copy.

## Internal vs public-safe

Both variants are produced on every run.

**Internal** (`reliability-report.md` / `.html`) — full detail: Prometheus
endpoint, the exact aggregate PromQL per SLI, the incident table with Loki
error/warn excerpts, node build/commit table.

**Public-safe** (`reliability-report.public.md`) — for a community-facing
summary. It:

- drops the endpoints, the raw log excerpt and the per-SLI PromQL;
- reduces the incident log to counts by severity;
- runs every line through the redactor
  ([`reliability/redact.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-manager/packages/scalability-harness/src/reliability/redact.ts)),
  which scrubs wallet / stake / pool addresses, tx / block / policy hashes,
  UTxO refs, raw CBOR, URLs and internal host:port names, and normalises
  absolute paths.

This matches the "do not expose" list and the community-summary fields in
[`scalability-stress-test-report.md`](scalability-stress-test-report.md) §14.
The node's own metrics are already low-cardinality with bounded labels (the
[`telemetry.md`](telemetry.md#label-policy) label policy forbids hashes /
addresses / CBOR / raw errors), so the SLO numbers themselves carry nothing
sensitive — the redaction is defence-in-depth on free-text sections.

Before publishing a public report, grep the file for `addr`, `stake`, `pool`,
a 56/64-hex run, and `http` to confirm the redactor left nothing behind.

## Sign-off checklist

Before a reliability report is treated as an official record:

- [ ] Window is a **closed** UTC interval, fully inside the retention horizon
      and after the cloud EFS cutover.
- [ ] `slo.json` is committed and `pnpm run slo:check` passes (rules match the
      objectives the report used).
- [ ] Report generated from the **cloud/testnet** Prometheus (the deployment of
      record), not a local stack.
- [ ] `sha256sum -c MANIFEST.sha256` passes on the bundle.
- [ ] `regen-reliability-report` reproduces the report from the bundle with no
      network calls.
- [ ] SLO table numbers spot-checked against hand-run PromQL for the window for
      at least one SLI.
- [ ] Incident log reviewed — each auto-detected incident is real, and no known
      incident is missing.
- [ ] Disposition and error-budget accounting reviewed and agreed.
- [ ] Public-safe variant grepped for identifiers; approved for the intended
      audience.
- [ ] Whole bundle archived in the internal evidence store under `<env>/<window>`.

## Traceability

The reliability-report generator lives in
[`demo/midgard-manager/packages/scalability-harness/src/reliability/`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-manager/packages/scalability-harness/src/reliability)
and reuses the harness's existing Prometheus, Loki, chart and manifest plumbing.
It covers the "Prometheus collection", "Cost collection" and "Report rendering"
rows of
[`scalability-stress-test-report.md`](scalability-stress-test-report.md)
Appendix A for the calendar-window case.
