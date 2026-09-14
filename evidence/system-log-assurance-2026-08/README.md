# Evidence bundle — System Log Assurance Report, deployed-node closure (2026-09-14)

Collected to close the §10 pending verification steps in `system-log-assurance-report.md`
for the deployed-node portion of the reporting window (2026-05-03 – 2026-08-31), and to
extend real coverage through the full window instead of stopping at the 2026-06-23 end of
the controlled-load campaign.

## Chain of custody

- **Collected**: 2026-09-14, ~10:30–11:15 local time (session timestamps in each file)
- **AWS account**: 810809345231, region us-west-2 (this account ID is already public elsewhere in this repo, via `demo/midgard-node/infra/aws/terraform/platform/envs/testnet.tfvars`'s ECR image URI — not new exposure)
- **CloudWatch log group**: `/ecs/sundial-node-testnet` (ARN: `arn:aws:logs:us-west-2:810809345231:log-group:/ecs/sundial-node-testnet`)
- **ALB access-log bucket**: `s3://sundial-node-testnet-alb-logs-20260529202411164500000002/`
- **ALB**: `arn:aws:elasticloadbalancing:us-west-2:810809345231:loadbalancer/app/sundial-node-testnet/df76007d21ac06a9`

## Sanitization pass (2026-09-14, post-collection)

This repo is **public**. Before publishing, all files were checked with `gitleaks` (0
findings) and a manual regex sweep for credential/key/token patterns (0 findings — see
`system-log-assurance-report.md` §6, B5/B6). The one thing that *did* need scrubbing: real
client IP addresses in `alb-2026-07-30-deploy-blip-and-2026-08-05-malformed-input.log`,
including the operator's own. These were replaced with stable pseudonyms (`CLIENT-01`,
`CLIENT-02`, ...) so recurring-source patterns (e.g. "the same client hit `/reset` three
times during the incident") stay verifiable without exposing real addresses. Internal/
private addresses (`10.x`, `0.0.0.0`) were left as-is — not sensitive. Real IPs remain
retrievable from the S3 source above by an authorized reviewer if ever needed.

## Key finding: CloudWatch retention is real, not just documented

`/ecs/sundial-node-testnet` has `retention_in_days = 30`. Verified empirically on 2026-09-14:
log-stream metadata (`describe-log-streams`) still lists streams with `firstEventTimestamp`
back to 2026-05-29, but `get-log-events` against those same streams returns **zero events**
for anything older than ~2026-08-17 — the metadata outlives the data. Live CloudWatch queries
for this node cannot currently reach further back than ~30 days before whenever they're run.
This is SLA-01 in the report, now confirmed with a concrete boundary rather than a theoretical risk.

## What closes the July–August gap instead

ALB access logs land in S3 (`alb-daily-status-summary-*.csv` etc. below), which has **no
30-day cap** — it covers the full window back to the node's first request on 2026-05-29.
Used here as the deployed-node evidence source for July–August in place of the expired
CloudWatch data, cross-checked against CloudTrail for deployment causality.

## Files

| File | What it is |
| --- | --- |
| `cloudwatch-crash-disk-scan.json` | Full-scan CloudWatch Logs Insights result: 0 matches for panic/unhandledRejection/FATAL/OOMKilled/heap-out-of-memory/ENOSPC/no-space-left across the whole log group, 2026-08-17–2026-08-31 (8,004,971 records / 1.37GB scanned). Closes A2/A4 for the CloudWatch-retained sub-window. |
| `cloudwatch-secret-sweep-result.json` | Same window, B5/B6 secret-pattern sweep: 0 matches. |
| `cloudwatch-node-errorwarn-2026-08-17_2026-08-31.log` | All 19 ERROR/WARN lines from the node container in the retained window — full population, not a sample. |
| `cloudwatch-node-startup-2026-08-17.log` | Real startup sequence, task that started 2026-08-17T09:03:29Z. |
| `alb-daily-status-summary-2026-05-29_2026-08-31.csv` | Daily request/status-code counts from ALB CloudWatch metrics, full window. Covers *all* ALB traffic, scanner noise included — see next file for the real-usage subset. |
| `alb-genuine-rpc-traffic-summary.txt` | Full-window ALB access-log parse (9,364 objects, ~23MB) scoped to `Host: rpc.testnet.sundialprotocol.com`, split into documented API paths (clean 200s) vs. undefined routes (scanner/exploit-probe traffic, which the app answers with 500 instead of 404 — see report §11 new observation). |
| `alb-2026-07-30-deploy-blip-and-2026-08-05-malformed-input.log` | Raw ALB rows: the 2026-07-30 deploy-correlated unavailability window, and the 2026-08-05 malformed-address 400 rejections. Client IPs redacted to pseudonyms (see Sanitization pass above). |
| `cloudtrail-ecs-updateservice-timeline.txt` | Every `UpdateService` call on the ECS service within CloudTrail's retained history, used to attribute the 2026-07-30 ALB error cluster to two rolling deployments rather than an unexplained fault. |
| `prometheus-p2-gap-diagnosis-2026-09-14.log` | Why §10 P2 (the reliability-report/Prometheus cross-check) stays open: the public Grafana dashboard is real and anonymous by design, but its Prometheus datasource proxy is broken (`connection refused` on `localhost:9090`) even though Prometheus itself is healthy — a live-diagnosed infra bug (SLA-10), not a credentials gap. |
| `MANIFEST.sha256` | Checksums of the files above. |

## Reproducing

CloudWatch Logs Insights queries and their exact query strings are recorded inline in each
file's header comment or in `system-log-assurance-report.md` §9/§10. ALB analysis was done
with a small ad hoc parser (not currently committed to the repo) that groups ALB access-log
rows by Host header and status code; re-run by syncing the S3 prefix above for the desired
date range and parsing with `csv.reader(f, delimiter=' ', quotechar='"')` per row.
