# ubelix.vector

Ansible role that installs and configures [Vector](https://vector.dev/) for centralised log shipping across the UBELIX HPC cluster.

## Architecture

```
  Cluster nodes (sender role)          Log server (aggregator role)
  ┌─────────────────────────┐          ┌─────────────────────────────────────┐
  │  journald               │          │  Vector source: vector (gRPC/mTLS)  │
  │     │                   │          │     │                               │
  │  Vector sink: vector ───┼──mTLS───►│     ├── Vector source: journald     │
  │  (gRPC, port 9000)      │          │     │   (local log01 journal)        │
  └─────────────────────────┘          │     │                               │
                                       │  Transform: normalize_fields (VRL)  │
                                       │     │  • @timestamp from journal ts  │
                                       │     │  • ECS field mapping           │
                                       │     │  • strip raw journald fields   │
                                       │     │                               │
                                       │  Vector sink: elasticsearch         │
                                       │     • disk buffer (256 MiB)         │
                                       │     • data stream, op_type: create  │
                                       └─────────────────────────────────────┘
                                                        │
                                                        ▼
                                               Elasticsearch
                                          (hpc_*_logs-journald)
```

All nodes ship their systemd journal logs to the central log server (`log01`) via Vector's native gRPC protocol, secured with mutual TLS (mTLS). Only the log server holds Elasticsearch credentials. The log server also reads its own local journal and ships everything to Elasticsearch.

## Roles

The role is deployed in two modes controlled by `vector_role`:

| Mode | Hosts | Description |
|---|---|---|
| `sender` | All cluster nodes | Reads local journald, forwards to aggregator via mTLS |
| `aggregator` | Log server (`log01`) | Receives from all senders, reads own journal, ships to Elasticsearch |

## Variables

### Common (all hosts)

| Variable | Default | Description |
|---|---|---|
| `vector_role` | `sender` | Role mode: `sender` or `aggregator` |
| `vector_ca_file` | `/etc/ssl/certs/ubelix-ca.pem` | Path to the CA certificate for mTLS |
| `vector_cert_file` | `/etc/ssl/certs/{{ ansible_fqdn }}.pem` | Path to the host TLS certificate |
| `vector_key_file` | `/etc/ssl/certs/{{ ansible_fqdn }}.key` | Path to the host TLS private key |

> **Note**: In production the TLS files are stored under `/etc/vector/tls/` to avoid SELinux
> label conflicts (`cert_t`) that prevent the `vector` process from reading `/etc/ssl/certs/`.

### Sender only

| Variable | Default | Description |
|---|---|---|
| `vector_aggregator_address` | `""` | `host:port` of the aggregator to forward logs to |

### Aggregator only

| Variable | Default | Description |
|---|---|---|
| `vector_listen_port` | `9000` | TCP port to receive gRPC connections from senders |
| `vector_es_endpoint` | `""` | Elasticsearch HTTPS endpoint URL |
| `vector_es_index` | `""` | Target data stream / index name |
| `vector_es_username` | `""` | Elasticsearch username |
| `vector_es_password` | `""` | Elasticsearch password (use Ansible Vault) |
| `vector_es_buffer_max_size` | `268435488` | Disk buffer size in bytes (default: 256 MiB) |

## TLS / mTLS

Every connection between senders and the aggregator is mutually authenticated:

- The aggregator presents its certificate and verifies the sender's certificate against the CA.
- Senders present their certificate and verify the aggregator's certificate against the CA.
- Both sides must hold a certificate signed by the UBELIX internal CA.

Certificate files are provisioned by the `log_shipping.yml` playbook in `pre_tasks` before the
role runs. They are written to `/etc/vector/tls/` on each host.

## Elasticsearch sink behaviour

The aggregator uses a **disk-backed buffer** in front of the Elasticsearch sink:

| Property | Value |
|---|---|
| Buffer type | Disk (persistent across restarts) |
| Default buffer size | 256 MiB |
| On buffer full | `block` — back-pressure to sources |
| Write operation | `op_type: create` (required for data streams) |

## Resilience: what happens when the aggregator is unavailable?

**On sender nodes**: Vector's `vector` sink uses `when_full: block` (the default). When the
aggregator is unreachable the sink retries with exponential back-off and back-pressure propagates
to the journald source, which **pauses reading**. The journald cursor position is not advanced
until events are successfully sent, so all unshipped events remain safely in the local journald
journal. When the aggregator comes back online Vector resumes from where it left off. No events
are lost as long as journald hasn't rotated the relevant journal files (governed by
`SystemMaxUse` / `SystemKeepFree` in `journald.conf` — typically a rolling window of days).

If the Vector process on a sender is restarted, the cursor file (`/var/lib/vector/`) persists
across restarts, so Vector resumes from the last confirmed position — again without data loss.

**On the aggregator**: If Elasticsearch becomes unavailable, the disk buffer absorbs up to
`vector_es_buffer_max_size` (default 256 MiB) of incoming events. Once full, back-pressure
propagates to all senders via the mechanism above. The disk buffer survives Vector restarts, so
events buffered before a restart are re-sent when Elasticsearch recovers.

## Single points of failure

| Component | SPOF? | Mitigation |
|---|---|---|
| `log01` (aggregator) | **Yes** | Senders queue in journald until log01 returns. No events lost within journald retention window. A second aggregator + VIP/DNS failover would eliminate this. |
| Elasticsearch cluster | **Yes** | 256 MiB disk buffer on aggregator absorbs events during short outages. |
| CA server | Soft | Only needed for certificate renewal (yearly). Daily operation does not depend on it. |

> **Note on TLS hostname binding**: The aggregator's TLS certificate uses `ansible_fqdn` as CN
> and SAN. The sender's `vector_aggregator_address` must match this FQDN. If the two diverge
> (e.g. after a DNS rename), `verify_hostname: true` on senders will reject the connection.
> The certificate SANs also include `log.ubelix.unibe.ch` for the log server, allowing a
> future DNS alias to be used as the aggregator address without recertification.

## Log retention on senders

Vector does **not** store logs locally on sender nodes beyond the in-memory queue. The source of
truth for local logs remains **systemd-journal** itself. Journald's own retention policy
(`SystemMaxUse`, `SystemKeepFree`, etc. in `/etc/systemd/journald.conf`) governs how long logs
stay on disk on each node — typically a rolling window of a few days to a few weeks depending on
write rate and disk space.

## ECS field mapping

The aggregator normalises raw journald fields into
[Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html) before
indexing. All other raw journald fields (uppercase, e.g. `_PID`, `_COMM`, `SYSLOG_FACILITY`) are
discarded to avoid field explosion.

| Journald field | ECS field | Type |
|---|---|---|
| event timestamp | `@timestamp` | date |
| `MESSAGE` | `message` | text |
| `host` (Vector metadata) | `host.name` | keyword |
| `_SYSTEMD_UNIT` | `systemd.unit` | keyword |
| `SYSLOG_IDENTIFIER` / `_COMM` | `process.name` | keyword |
| `_PID` | `process.pid` | integer |
| `PRIORITY` | `log.syslog.severity.code` | integer |

## Files managed on target hosts

| Path | Description |
|---|---|
| `/etc/vector/vector.yaml` | Vector configuration (owner `root:vector`, mode `0640`) |
| `/etc/vector/tls/` | TLS certificates and CA cert for mTLS |
| `/var/lib/vector/` | Vector data directory: journal cursor, disk buffer |
| `/var/lib/vector/buffer/` | Elasticsearch disk buffer (aggregator only) |
