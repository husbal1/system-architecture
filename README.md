# system-architecture


After a few production glitches, you have decided to no longer simply be reactive to problems, but proactive. Your goal is to monitor and collect key metrics for the software systems (web applications and web servers) developed by the company.

You have decided to offer a common API that will be used to standardize the collection of this data and that will be used by all development teams.

Produce a plan that details a proposal for architecture, design and specifications for this monitoring system.


# Monitoring System: Architecture, Design & Specification Plan

## 1. Objectives & Scope

### Primary Goals
- **Standardize** metric collection across all teams via a common API
- **Proactively** detect issues before they impact users
- **Centralize** observability data for web applications and web servers
- Enable **alerting**, **dashboards**, and **historical analysis**

### Scope
| In Scope | Out of Scope (Phase 1) |
|----------|------------------------|
| Web applications | Network hardware monitoring |
| Web servers | Mobile app instrumentation |
| Application metrics | Business intelligence/analytics |
| System/host metrics | Full APM tracing (later phase) |
| Alerting & dashboards | |

---

## 2. Key Metrics to Collect

### Application Metrics (The "Four Golden Signals" + RED)
- **Latency** – request response times (p50, p95, p99)
- **Traffic** – requests per second, throughput
- **Errors** – error rate, error count by type (4xx, 5xx)
- **Saturation** – queue depth, connection pool usage

### Web Server / Host Metrics
- CPU usage (%)
- Memory usage (used/available)
- Disk I/O and disk space
- Network I/O (bandwidth, packet errors)
- Active connections / open file descriptors
- Process uptime

### Application-Specific
- Custom business metrics (e.g., logins, checkout completions)
- Cache hit/miss ratios
- Database query times
- Dependency/external service health

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Teams                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                  │
│  │  App A   │   │  App B   │   │  App C   │  (instrumented    │
│  │ + Agent  │   │ + Agent  │   │ + Agent  │   via Client SDK) │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                  │
└───────┼──────────────┼──────────────┼───────────────────────┘
        │              │              │
        │   Metrics (push) / scrape (pull)
        ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│              COLLECTION API (Common Standard)                │
│   ┌─────────────────┐      ┌──────────────────────┐          │
│   │ Ingestion Gateway│─────▶│  Validation/Auth      │         │
│   └─────────────────┘      └──────────────────────┘          │
└───────────────────────┬──────────────────────────────────────┘
                        │
                        ▼
            ┌────────────────────────┐
            │   Message Queue/Buffer  │  (Kafka / Redis)
            └───────────┬─────────────┘
                        ▼
            ┌────────────────────────┐
            │  Time-Series Database   │  (Prometheus / InfluxDB)
            └───────────┬─────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌─────────────┐ ┌──────────────┐ ┌──────────────┐
│ Dashboards  │ │ Alerting     │ │  Query/API   │
│ (Grafana)   │ │ (Alertmgr)   │ │  for reports │
└─────────────┘ └──────┬───────┘ └──────────────┘
                       ▼
              ┌─────────────────┐
              │ Notifications    │
              │ Slack/Email/Page │
              └─────────────────┘
```

### Collection Pattern: Push vs Pull
| Approach | Pros | Cons | Recommendation |
|----------|------|------|----------------|
| **Pull (scrape)** | Service discovery, easy health detection | Harder behind firewalls/ephemeral jobs | **Default** for long-running services |
| **Push** | Works for short-lived jobs, batch | Requires gateway, harder to detect down hosts | For batch/serverless via Pushgateway |

**Decision:** Hybrid — pull as default (Prometheus model), push gateway available for ephemeral workloads.

---

## 4. The Common API Specification

### Design Principles
- **Language-agnostic** — provide SDKs but expose open standard
- **Low overhead** — minimal performance impact on apps
- **Self-documenting** — consistent naming conventions
- **Backward compatible** — versioned API

### Recommended Standard
Adopt **OpenTelemetry** + **Prometheus exposition format** as the foundation rather than reinventing the wheel.

### 4.1 Metric Naming Convention
```
<namespace>_<subsystem>_<metric>_<unit>

Examples:
  http_server_request_duration_seconds
  http_server_requests_total
  app_db_query_duration_seconds
  system_cpu_usage_ratio
```

### 4.2 Metric Types
| Type | Use Case | Example |
|------|----------|---------|
| **Counter** | Monotonic increasing values | total requests, errors |
| **Gauge** | Values that go up/down | memory usage, queue size |
| **Histogram** | Distributions (latency) | request duration buckets |
| **Summary** | Client-side quantiles | p99 latency |

### 4.3 Mandatory Labels (for consistency)
```yaml
service:      name of the service        # e.g., "payment-api"
environment:  prod | staging | dev
team:         owning team                # e.g., "checkout-team"
version:      app version                # e.g., "1.4.2"
instance:     host/pod identifier
region:       deployment region
```

### 4.4 Sample Endpoint Contract

**Pull mode (scrape endpoint exposed by app):**
```
GET /metrics
Content-Type: text/plain; version=0.0.4

# HELP http_server_requests_total Total HTTP requests
# TYPE http_server_requests_total counter
http_server_requests_total{method="GET",status="200",service="payment-api"} 1027
http_server_request_duration_seconds_bucket{le="0.1",...} 950
```

**Push mode (REST ingestion API):**
```http
POST /api/v1/metrics
Authorization: Bearer <api-token>
Content-Type: application/json

{
  "service": "payment-api",
  "environment": "prod",
  "timestamp": "2024-01-15T10:30:00Z",
  "metrics": [
    {
      "name": "http_server_requests_total",
      "type": "counter",
      "value": 1027,
      "labels": {"method": "GET", "status": "200"}
    }
  ]
}

Response: 202 Accepted
```

### 4.5 Client SDK Responsibilities
Provide thin libraries (Java, Python, Node.js, Go) that:
- Auto-instrument common frameworks (Express, Spring, Flask)
- Enforce naming conventions
- Handle batching, buffering, retries
- Manage authentication
- Expose simple API:
```python
metrics.counter("orders_processed").inc()
metrics.histogram("checkout_duration").observe(0.234)
with metrics.timer("db_query"):
    run_query()
```

---

## 5. Technology Stack Proposal

| Layer | Recommendation | Alternatives |
|-------|----------------|--------------|
| Instrumentation | OpenTelemetry SDK | Micrometer, StatsD |
| Collection | Prometheus / OTel Collector | Telegraf |
| Buffering | Kafka | Redis Streams |
| Storage (TSDB) | Prometheus + Thanos/Mimir (long-term) | InfluxDB, VictoriaMetrics |
| Visualization | Grafana | Kibana |
| Alerting | Alertmanager | Grafana Alerting |
| Notifications | Slack, PagerDuty, Email | Opsgenie |

**Rationale:** Open-source, CNCF-backed, industry-standard, avoids vendor lock-in, large community.

---

## 6. Alerting Strategy

### Alert Tiers
| Severity | Response | Example |
|----------|----------|---------|
| **P1 – Critical** | Page immediately | Service down, 5xx > 10% |
| **P2 – Warning** | Notify channel | p99 latency > 1s |
| **P3 – Info** | Log/dashboard | Disk > 70% |

### Best Practices
- Alert on **symptoms**, not causes (user-facing impact)
- Define **SLOs/SLIs** and alert on error budget burn
- Avoid alert fatigue — tune thresholds, use inhibition rules
- Every alert must be **actionable** and have a **runbook link**

### Example Alert Rule (Prometheus)
```yaml
- alert: HighErrorRate
  expr: |
    sum(rate(http_server_requests_total{status=~"5.."}[5m])) by (service)
    / sum(rate(http_server_requests_total[5m])) by (service) > 0.05
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High error rate on {{ $labels.service }}"
    runbook: "https://wiki/runbooks/high-error-rate"
```

---

## 7. Data Retention & Storage Policy

| Resolution | Retention | Purpose |
|-----------|-----------|---------|
| Raw (high-res) | 15 days | Debugging, incident analysis |
| 5-min downsampled | 90 days | Trend analysis |
| 1-hour downsampled | 1+ year | Capacity planning |

Use downsampling/rollups to manage storage costs.

---

## 8. Security & Governance

- **Authentication:** API tokens / mTLS per service
- **Authorization:** Team-scoped write access, role-based dashboard access
- **Encryption:** TLS in transit, encryption at rest
- **PII:** Forbid sensitive data in labels (enforce in SDK)
- **Multi-tenancy:** Isolate teams' data and quotas
- **Cardinality limits:** Prevent label explosion (cost/performance)

---

## 9. Implementation Roadmap

### Phase 1 – Foundation (Weeks 1–6)
- Stand up core stack (Prometheus, Grafana, Alertmanager)
- Define API spec & naming conventions
- Build first SDK (most-used language)
- Pilot with 1–2 willing teams

### Phase 2 – Standardization (Weeks 7–14)
- Release SDKs for all major languages
- Onboard host/server metrics (node exporters)
- Build standard dashboard templates
- Establish alerting baselines & runbooks

### Phase 3 – Scale & Adopt (Weeks 15–24)
- Roll out to all teams (with support)
- Add long-term storage (Thanos/Mimir)
- Implement SLO tracking
- Documentation & training

### Phase 4 – Maturity (Ongoing)
- Add distributed tracing (full observability)
- Anomaly detection / ML-based alerting
- Cost optimization & cardinality governance

---

## 10. Adoption & Governance

- **Documentation portal** with quickstarts and examples
- **"Golden path" templates** — make the right way the easy way
- **Onboarding support** — office hours, dedicated channel
- **Standards enforcement** — CI checks for metric naming, linting
- **Center of Excellence** — small team owning the platform
- **Feedback loop** — regular reviews with dev teams

---

## 11. Success Metrics (Meta-Monitoring)

How we measure if this initiative succeeds:
- % of services onboarded
- Mean Time To Detect (MTTD) ↓
- Mean Time To Resolve (MTTR) ↓
- Number of incidents caught proactively
- Developer satisfaction / ease of adoption

---

DevOps Infrastructure:

We are interested in your skill level and general experience with Devops infrastructure. We would like you to describe, to the extent of your knowledge, experience, contractual obligations (e.g., confidentiality agreements in place), and available time, the most complex Devops and architecture you have worked with.  

Include as many details as you wish: development, testing and production environments; operational parameters; database systems, their backup and replication processes, DR-HA 
configurations; monitoring and testing; infrastructure management and scaling; any other interesting features, etc. 

Consider this exercise as the first part of a whiteboard interview question - as we wi l use it to move the discussion forward during the technical interview. As much detail as possible is appreciated (diagrams and text explanations). 



# Telecom Network Traffic Monitoring & Analytics Platform (NTMAP)
## A Complete DevOps & Architecture Deep-Dive

---

## 1. System Overview & Business Context

**NTMAP** (Network Traffic Monitoring & Analytics Platform) is a passive monitoring system deployed in a Tier-1 telecom operator's network. It receives **mirrored (SPAN/TAP) traffic** from the control and user planes of the 4G (EPC) and 5G (5GC) core network functions, decodes telecom protocols in real time, correlates sessions across network functions, detects anomalies/fraud/SLA violations, and feeds dashboards + alerting + long-term analytics.

### 1.1 What it monitors

| Network Generation | Interfaces Monitored | Protocols Decoded |
|---|---|---|
| **4G EPC** | S1-MME, S1-U, S11, S6a, S5/S8, Gx, Gy, SGi | S1AP, GTP-C, GTP-U, Diameter, RADIUS |
| **5G SA Core** | N1/N2, N3, N4, N6, SBI (N7,N8,N10,N11,N12,N15) | NGAP, PFCP, GTP-U, HTTP/2 (SBI), JSON |
| **IMS/VoLTE** | Gm, Mw, ISC, Cx/Dx | SIP, Diameter, RTP/RTCP |

### 1.2 Scale Targets (Production)

```
Aggregate mirrored throughput........ 1.2 Tbps (peak), 600–800 Gbps (avg)
Packets per second................... ~180 Mpps (peak)
Active subscriber sessions tracked... 45–60 million concurrent
Control-plane transactions/sec....... 2.5 million TPS
CDR-like records generated/day....... ~85 billion
Hot-data retention................... 30 days (queryable)
Warm-data retention.................. 13 months (aggregated)
Cold/archive retention............... 7 years (regulatory/lawful intercept)
Alert end-to-end latency............. < 2 seconds (p95)
```

---

## 2. High-Level Architecture

```

<img width="1729" height="837" alt="image" src="https://github.com/user-attachments/assets/1950b64c-8a89-45f3-a0f9-66408cce6c54" />

---

## 3. The Capture / Probe Tier (The Most Critical Layer)

This tier is **bare-metal** or **virtualized** with SRIOV & DPDK to support high throughputs.

### 3.1 Probe Server Hardware Spec (per node)

```
CPU.............. 2× AMD EPYC 9554 (64-core each) — 128 physical cores
RAM.............. 1 TB DDR5 (NUMA-aware, hugepages reserved: 256 GB)
NICs............. 4× NVIDIA/Mellanox ConnectX-7 (100GbE), DPDK-bound
                  + 1× 25GbE management NIC (kernel-bound)
Storage.........  2× 1.92TB NVMe (OS, RAID1)
                  8× 7.68TB NVMe (ring-buffer PCAP, RAID0)
Boot............. PXE + iPXE, immutable OS image
```

### 3.2 Software Stack on Probe

```
┌──────────────────────────────────────────────────────────┐
│  Probe Application (C++/Rust, DPDK 23.x)                  │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ RX queues  │→ │ Flow Hash &  │→ │ Protocol Decoders│  │
│  │ (RSS, 64q) │  │ Reassembly   │  │ (GTP/Diameter/   │  │
│  └────────────┘  └──────────────┘  │  SIP/NGAP/HTTP2) │  │
│                                     └────────┬─────────┘  │
│  ┌──────────────────────┐          ┌─────────▼─────────┐  │
│  │ Rolling PCAP buffer  │          │ Record Builder    │  │
│  │ (on-demand export)   │          │ (Protobuf)        │  │
│  └──────────────────────┘          └─────────┬─────────┘  │
│                                              Kafka Producer │
└──────────────────────────────────────────────────────────┘
   Pinned cores: 0-3 OS, 4-63 RX/decode (NUMA0),
                 64-67 OS, 68-127 RX/decode (NUMA1)
   Hugepages: 1GB pages, mbuf pools per NUMA node
```

### 3.3 Key Operational Parameters

| Parameter | Value | Why |
|---|---|---|
| Packet drop threshold (alert) | > 0.001% | Lawful-intercept compliance needs near-zero loss |
| Ring buffer size | 60 GB per NIC | ~5 sec of full-rate PCAP for retro-analysis |
| Flow table size | 80M entries | Covers concurrent sessions + headroom |
| Flow idle timeout | 300 s (UDP), 3600 s (TCP) | Matches GTP-U session lifetimes |
| Hugepage allocation | 256 GB | mbuf pools must never starve |
| Decoder worker threads | 56 per NUMA node | One per pinned core |

---

## 4. Environments: Dev / Test / Staging / Production

We run **four distinct environments**, each isolated by network, IAM, and cloud account/datacenter.

### 4.1 Environment Matrix

```
┌─────────────┬──────────────┬─────────────┬──────────────┬───────────────┐
│  Aspect     │     DEV      │    TEST/QA  │   STAGING    │  PRODUCTION   │
├─────────────┼──────────────┼─────────────┼──────────────┼───────────────┤
│ Location    │ Cloud (k8s)  │ Cloud (k8s) │ On-prem DC2  │ On-prem DC1+DC2│
│ Traffic     │ Synthetic    │ Replayed    │ 5% live mirror│ 100% live     │
│             │ generators   │ PCAP corpus │ (shadow tap) │  mirror       │
│ Scale       │ 1 probe (VM) │ 2 probes(VM)│ 4 probes(HW) │ 40+ probes(HW)│
│ Kafka       │ 3 brokers    │ 3 brokers   │ 6 brokers    │ 24 brokers    │
│ ClickHouse  │ 1 node       │ 2 nodes     │ 6 nodes      │ 48 nodes      │
│ Data        │ Fully synthetic│ Anonymized │ Anonymized   │ Real (encrypted)│
│ Refresh     │ On commit    │ Nightly     │ Per release  │ Controlled    │
│ Uptime SLA  │ None         │ None        │ 99%          │ 99.999%       │
└─────────────┴──────────────┴─────────────┴──────────────┴───────────────┘
```

### 4.2 Why Staging Uses a "Shadow Tap"

Staging receives a **5% statistically-sampled mirror** of live production traffic (IMSI-anonymized at the probe). This lets us validate new decoders and detection rules against *real* protocol behaviors and malformed packets that synthetic generators can't reproduce — without risking production.

### 4.3 Traffic Generation for Dev/Test

```
DEV:  TRex (Cisco) + custom scenario scripts generating:
      - GTP-C attach/detach storms
      - Diameter CCR/CCA credit-control flows
      - SIP INVITE/BYE VoLTE call flows
      - 5G NGAP registration + PDU session establishment

TEST: Curated PCAP corpus (12 TB) of:
      - Real anonymized captures (replayed via tcpreplay-edit at scale)
      - Known-bad packets (fuzzing corpus, malformed headers)
      - Regression cases (every past production bug → a PCAP)
```

---

## 5. CI/CD Pipeline

```
Developer
   │ git push (feature branch)
   ▼
┌──────────────────────────────────────────────────────────────┐
│  GitLab CI / ArgoCD (GitOps)                                  │
│                                                               │
│  STAGE 1: Build & Static Analysis                            │
│    • Compile probe (C++/Rust) with -Werror                   │
│    • clang-tidy, cppcheck, cargo clippy                      │
│    • SAST (Semgrep, Coverity), SBOM generation (Syft)        │
│    • Container image build (multi-stage, distroless)         │
│    • Sign images (cosign), scan (Trivy/Grype)               │
│                                                               │
│  STAGE 2: Unit + Component Tests                             │
│    • 8,000+ unit tests, decoder golden-file tests           │
│    • Protocol decoder fuzzing (AFL++, 30 min budget)        │
│    • Coverage gate: ≥ 80% line, ≥ 70% branch                │
│                                                               │
│  STAGE 3: Integration Tests (ephemeral k8s namespace)       │
│    • Spin full stack via Helm in throwaway namespace        │
│    • Replay 500 GB PCAP, assert record counts & KPIs        │
│    • Contract tests (Pact) for API consumers                │
│                                                               │
│  STAGE 4: Performance / Soak (nightly, dedicated HW)        │
│    • TRex line-rate test: assert 0 drops @ 100GbE/probe     │
│    • 8-hour soak: memory-leak detection, latency p99        │
│                                                               │
│  STAGE 5: Deploy                                            │
│    • DEV: auto on merge to develop                          │
│    • STAGING: auto on merge to main + manual approval       │
│    • PROD: manual approval + change-ticket + canary         │
└──────────────────────────────────────────────────────────────┘
```

### 5.1 Production Deployment Strategy

- **Applications (k8s tier):** Argo Rollouts **canary** — 5% → 25% → 50% → 100%, with automatic rollback if SLO error budget burns (Prometheus-based analysis).
- **Probes (bare-metal):** **Rolling, NPB-coordinated.** Because probes are stateless-ish capture nodes behind the packet broker, we drain one probe at a time. The NPB redistributes its flow-hash buckets to peers, we reflash the OS image (immutable), reboot, validate zero-drop, then return it to the hash pool. ~6 probes upgraded per maintenance window.
- **Database schema changes:** Expand-contract migrations (gh-ost / ClickHouse `ON CLUSTER` DDL), never destructive in a single release.

---

## 6. Streaming / Message Bus Tier (Apache Kafka)

### 6.1 Kafka Topology

```
Production Kafka Cluster (24 brokers, 3 racks × 8)
┌─────────────────────────────────────────────────────────┐
│  Topic Design                                           │
│  ┌──────────────────────┬───────────┬────────┬────────┐ │
│  │ Topic                │ Partitions│ Repl.  │ Retain │ │
│  ├──────────────────────┼───────────┼────────┼────────┤ │
│  │ gtpc.records         │   480     │   3    │  24h   │ │
│  │ gtpu.flow.records    │   960     │   3    │  12h   │ │
│  │ diameter.records     │   240     │   3    │  24h   │ │
│  │ sip.records          │   240     │   3    │  24h   │ │
│  │ ngap.records         │   240     │   3    │  24h   │ │
│  │ pfcp.records         │   240     │   3    │  24h   │ │
│  │ enriched.sessions    │   480     │   3    │  48h   │ │
│  │ alerts               │    48     │   3    │   7d   │ │
│  │ dlq.*  (dead-letter) │    24     │   3    │  14d   │ │
│  └──────────────────────┴───────────┴────────┴────────┘ │
│                                                          │
│  Partition key: murmur2(IMSI-hash) → guarantees all     │
│  records for one subscriber land on same partition →    │
│  enables stateful per-subscriber correlation in Flink   │
└─────────────────────────────────────────────────────────┘

  min.insync.replicas = 2     acks = all (for alerts/CDR)
  acks = 1 (for high-volume gtpu flow records — perf tradeoff)
  Compression: zstd            Tiered storage → S3 for older segments
```

### 6.2 Why partition by IMSI-hash?

Subscriber session correlation requires that the control-plane event (e.g., GTP-C `Create Session`) and user-plane stats (GTP-U byte counters) for the *same subscriber* be processed by the *same* Flink task. Keying by IMSI-hash guarantees co-location, enabling **exactly-once stateful joins** without cross-task shuffles.

---

## 7. Stream Processing & Correlation (Apache Flink)

```
┌─────────────────────────────────────────────────────────────┐
│  Flink Application Cluster (Kubernetes, 60 TaskManagers)    │
│                                                             │
│  Job 1: Session Correlation                                │
│    Source(gtpc) ─┐                                         │
│    Source(gtpu) ─┼→ KeyBy(IMSI) → Stateful Window Join →   │
│    Source(pfcp) ─┘    (RocksDB state backend)              │
│                       → Enriched Session Record            │
│                                                             │
│  Job 2: KPI Aggregation                                    │
│    → Tumbling 1-min windows: attach success rate,          │
│      session setup time, throughput per cell/APN/slice     │
│                                                             │
│  Job 3: Anomaly / Fraud Detection                          │
│    → CEP patterns: SIM-box detection, signaling storms,    │
│      DDoS on control plane, abnormal roaming patterns      │
│                                                             │
│  State backend: RocksDB on local NVMe                      │
│  Checkpointing: every 30s → S3 (incremental, exactly-once) │
│  Savepoints: before every deploy → S3                      │
└─────────────────────────────────────────────────────────────┘
```

### 7.1 Flink Operational Parameters

| Parameter | Value |
|---|---|
| Parallelism (Job 1) | 480 (matches partition count) |
| Checkpoint interval | 30 s |
| Checkpoint timeout | 5 min |
| State size (steady) | ~4 TB across cluster |
| Restart strategy | exponential-delay, max 10 attempts |
| Watermark strategy | bounded-out-of-orderness, 10 s |
| Allowed lateness | 60 s (then → side-output / DLQ) |

---

## 8. Storage Tier — Databases in Detail

### 8.1 Database-by-Purpose Map

```
┌──────────────────┬────────────────────────────────┬──────────────┐
│  Database        │  Purpose                        │  Data Type   │
├──────────────────┼────────────────────────────────┼──────────────┤
│ ClickHouse       │ Analytical queries, CDR-like    │ Columnar OLAP│
│                  │ records, traffic analytics      │              │
│ TimescaleDB      │ Time-series KPIs / network      │ Time-series  │
│ (PostgreSQL ext) │ metrics for Grafana             │              │
│ PostgreSQL       │ Config, topology, users, rules, │ Relational   │
│                  │ alert definitions, RBAC         │ OLTP         │
│ Elasticsearch    │ Full-text search over signaling │ Search index │
│                  │ logs, troubleshooting           │              │
│ Redis (Cluster)  │ Hot session cache, rate-limit,  │ KV / in-mem  │
│                  │ enrichment lookup (IMSI→profile)│              │
│ Ceph / S3        │ Cold PCAP archive, lawful       │ Object store │
│                  │ intercept, Flink checkpoints    │              │
└──────────────────┴────────────────────────────────┴──────────────┘
```

### 8.2 ClickHouse — The Analytical Workhorse

```
Production ClickHouse Cluster
┌────────────────────────────────────────────────────────────┐
│  48 nodes = 24 shards × 2 replicas                          │
│  Coordination: ClickHouse Keeper (Raft, 5 nodes)           │
│                                                            │
│  Shard 1        Shard 2        ...      Shard 24           │
│  ┌────┬────┐    ┌────┬────┐             ┌────┬────┐        │
│  │ R1 │ R2 │    │ R1 │ R2 │             │ R1 │ R2 │        │
│  └────┴────┘    └────┴────┘             └────┴────┘        │
│   DC1   DC2      DC1   DC2               DC1   DC2          │
│   (cross-DC replica placement for DR)                     │
│                                                            │
│  Engine: ReplicatedMergeTree                              │
│  Distributed table fans out queries across shards         │
│                                                            │
│  Per node: 64 cores, 512 GB RAM, 24× 7.68TB NVMe (RAID10) │
└────────────────────────────────────────────────────────────┘
```

**Table design example (records table):**

```sql
CREATE TABLE session_records ON CLUSTER ntmap
(
    event_time      DateTime64(3) CODEC(DoubleDelta, ZSTD),
    imsi_hash       UInt64,
    msisdn_hash     UInt64,
    cell_id         UInt32,
    apn             LowCardinality(String),
    slice_id        LowCardinality(String),  -- 5G S-NSSAI
    proto           LowCardinality(String),
    bytes_up        UInt64 CODEC(T64, ZSTD),
    bytes_down      UInt64 CODEC(T64, ZSTD),
    setup_time_ms   UInt32,
    cause_code      Int16,
    -- ... 40+ more columns
)
ENGINE = ReplicatedMergeTree('/clickhouse/{shard}/session_records','{replica}')
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (apn, cell_id, imsi_hash, event_time)
TTL event_time + INTERVAL 30 DAY TO VOLUME 'cold',     -- tiered storage
    event_time + INTERVAL 13 MONTH DELETE
SETTINGS storage_policy = 'hot_warm_cold';
```

**Tiered storage policy:**
```
hot   → local NVMe        (0–7 days,  fastest queries)
warm  → local SATA SSD    (7–30 days)
cold  → S3-backed disk    (30 days–13 months, MergeTree on object store)
```

### 8.3 Materialized Views for Real-Time Rollups

```sql
-- Pre-aggregate per-cell KPIs at insert time
CREATE MATERIALIZED VIEW mv_cell_kpi_1min
ENGINE = ReplicatedAggregatingMergeTree(...)
AS SELECT
    toStartOfMinute(event_time) AS minute,
    cell_id,
    countState() AS sessions,
    avgState(setup_time_ms) AS avg_setup,
    sumState(bytes_up + bytes_down) AS total_bytes
FROM session_records
GROUP BY minute, cell_id;
```

---

## 9. Backup & Replication Strategy (Per Datastore)

### 9.1 Summary Table

```
┌──────────────┬──────────────────┬───────────────┬──────────┬────────┐
│ Datastore    │ Replication      │ Backup Method │ RPO      │ RTO    │
├──────────────┼──────────────────┼───────────────┼──────────┼────────┤
│ ClickHouse   │ Native repl ×2   │ clickhouse-   │ ~0       │ <30min │
│              │ (cross-DC)       │ backup → S3   │ (replica)│        │
│              │                  │ incremental/3h│ 3h(backup)│       │
│ PostgreSQL   │ Streaming repl   │ pgBackRest    │ <5s      │ <10min │
│              │ (sync to 1,async │ (WAL archiving│          │        │
│              │  to 2) + Patroni │ + full daily) │          │        │
│ TimescaleDB  │ Same as PG       │ pgBackRest +  │ <5s      │ <10min │
│              │                  │ chunk export  │          │        │
│ Elasticsearch│ 1 primary +      │ Snapshot to   │ 15min    │ <30min │
│              │ 1 replica shard  │ S3 repository │          │        │
│ Redis        │ Cluster + repl   │ RDB+AOF → S3  │ 1s (AOF) │ <5min  │
│              │ (each master+1)  │ hourly RDB    │          │        │
│ Kafka        │ Repl factor 3 +  │ Tiered storage│ ~0       │ <15min │
│              │ MirrorMaker2 →DR │ to S3 + MM2   │          │        │
│ Ceph/S3      │ Erasure-coded +  │ Cross-region  │ async    │ N/A    │
│              │ multi-site repl  │ replication   │ ~15min   │        │
└──────────────┴──────────────────┴───────────────┴──────────┴────────┘
```

### 9.2 PostgreSQL HA in Detail (Patroni + etcd)

```
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL HA Cluster (Patroni)             │
│                                                          │
│   DC1                          DC2                       │
│  ┌──────────┐   sync repl    ┌──────────┐               │
│  │ Primary  │ ─────────────→ │ Sync     │               │
│  │ (Leader) │                │ Standby  │               │
│  └────┬─────┘                └──────────┘               │
│       │ async repl                                       │
│       └──────────────────→  ┌──────────┐                │
│                             │ Async    │ (read replica  │
│                             │ Standby  │  + DR)         │
│                             └──────────┘                │
│                                                         │
│   Patroni agents manage leader election via etcd (5     │
│   nodes). HAProxy/PgBouncer routes writes → leader,     │
│   reads → standbys. Automatic failover < 30s.           │
│                                                         │
│   WAL archived continuously via pgBackRest to S3.       │
│   Full backup: daily. Differential: every 6h.          │
│   PITR (point-in-time-recovery) granularity: per-txn.  │
└─────────────────────────────────────────────────────────┘
```

### 9.3 Backup Verification (Critical — backups you don't test are not backups)

- **Nightly automated restore drills:** A dedicated "restore-validation" namespace restores the *latest* PostgreSQL and ClickHouse backups, runs schema/row-count/checksum assertions, then tears down. Failures page the on-call DBA.
- **Quarterly full DR game-day:** Simulated DC1 total loss → fail over to DC2 → validate full functionality → measure actual RTO/RPO against targets.

---

## 10. Disaster Recovery & High Availability (DR-HA)

### 10.1 Multi-DC Topology

```
┌────────────────────────────┐      ┌────────────────────────────┐
│        DC1 (Primary)       │      │     DC2 (Active-DR)        │
│  ─────────────────────────  │      │  ────────────────────────  │
│  • 24 probes (50% traffic) │      │  • 16 probes (region 2)    │
│  • Kafka brokers 1-12      │◄────►│  • Kafka brokers 13-24     │
│  • ClickHouse shards 1-12  │ Mirr.│  • ClickHouse shards 13-24 │
│  • PG Primary              │ Maker│  • PG Sync Standby         │
│  • Flink active jobs       │  2   │  • Flink standby (savepts) │
│  • Full app tier           │      │  • Full app tier (active)  │
└──────────────┬─────────────┘      └─────────────┬──────────────┘
               │     Dark fiber, 2× 400GbE         │
               │     <2ms latency, BGP anycast     │
               └───────────────┬───────────────────┘
                               ▼
                   ┌────────────────────┐
                   │   DC3 (Cold DR)    │
                   │  S3 cross-region   │
                   │  backup target +   │
                   │  IaC to rebuild    │
                   └────────────────────┘
```

**Design philosophy: Active-Active for capture & ingest (each DC monitors its regional network functions), Active-Passive for the stateful write-primary databases.**

### 10.2 Failure Scenarios & Responses

| Failure | Detection | Automated Response | Manual? |
|---|---|---|---|
| Single probe dies | NPB heartbeat + Prometheus | NPB redistributes flow buckets; alert | No |
| Kafka broker dies | Controller detects | Partitions re-elect leaders (ISR) | No |
| ClickHouse replica dies | Keeper detects | Queries route to surviving replica | No |
| PG primary dies | Patroni/etcd | Auto-failover to sync standby (<30s) | No |
| Entire DC1 lost | Multi-signal (BGP, health) | DC2 promotes PG, Flink restores from savepoint, anycast withdraws DC1 | Approval gate |
| Both DCs lost | — | Rebuild from DC3 via IaC + S3 restore | Yes (full runbook) |

### 10.3 Capture-Layer Resilience (the hard part)

Because mirrored packets are **ephemeral** (you can't "re-request" a packet that already passed), the capture tier must never drop. Mitigations:

1. **NPB-level load balancing** with N+1 probe redundancy per traffic group.
2. **Rolling 60s ring buffers** on each probe — if downstream Kafka stalls, the probe keeps capturing to local NVMe and back-pressures gracefully.
3. **Dual-feed critical interfaces** (S6a/Diameter, N7/N12) tapped to two independent probes for zero-loss compliance interfaces.

---

## 11. Monitoring, Observability & Alerting

### 11.1 The Observability Stack ("monitoring the monitor")

```
┌───────────────────────────────────────────────────────────┐
│  METRICS:  Prometheus (federated) + Thanos (long-term)   │
│    • Probe drop counters, mbuf usage, NUMA balance        │
│    • Kafka lag (Burrow), Flink checkpoint duration        │
│    • ClickHouse merge backlog, query latency p99          │
│    • Node exporter, DCGM (GPU), NIC stats                 │
│                                                          │
│  LOGS:     Loki + Promtail (app logs)                    │
│            Elasticsearch (signaling/audit logs)          │
│                                                          │
│  TRACES:   OpenTelemetry → Tempo / Jaeger                │
│            (API request → Flink → ClickHouse query span) │
│                                                          │
│  DASHBOARDS: Grafana (200+ dashboards)                   │
│                                                          │
│  ALERTING: Alertmanager → PagerDuty / Opsgenie          │
│            + Slack + auto-ticket (Jira/ServiceNow)       │
│                                                          │
│  SLO TRACKING: Sloth-generated SLO rules + error budgets │
└───────────────────────────────────────────────────────────┘
```

### 11.2 Key SLOs & Golden Signals

```
┌─────────────────────────────┬──────────┬────────────────────┐
│  SLO                        │ Target   │ Error Budget        │
├─────────────────────────────┼──────────┼────────────────────┤
│ Packet capture completeness │ 99.999%  │ 26s/month drop      │
│ Ingest pipeline availability│ 99.99%   │ 4.3 min/month       │
│ Query API availability      │ 99.95%   │ 21.9 min/month      │
│ Alert latency (p95)         │ < 2s     │ —                   │
│ Data freshness (hot data)   │ < 60s    │ ingest→queryable    │
└─────────────────────────────┴──────────┴────────────────────┘
```

### 11.3 Critical Alerts (sample)

```yaml
# Probe packet drops - PAGE immediately (compliance risk)
- alert: ProbePacketDropHigh
  expr: rate(probe_rx_dropped_packets_total[1m]) 
        / rate(probe_rx_packets_total[1m]) > 0.00001
  for: 30s
  severity: critical
  annotations:
    runbook: "https://runbooks/probe-drops"

# Kafka consumer lag growing - pipeline falling behind
- alert: KafkaConsumerLagGrowing
  expr: kafka_consumergroup_lag > 5000000 
        and deriv(kafka_consumergroup_lag[5m]) > 0
  for: 5m
  severity: critical

# ClickHouse merge backlog - ingest will stall soon
- alert: ClickHouseMergeBacklog
  expr: clickhouse_async_metric_MaxPartCountForPartition > 300
  for: 10m
  severity: warning
```

---

## 12. Infrastructure Management & Automation (IaC)

### 12.1 Tooling Layers

```
┌──────────────────────────────────────────────────────────┐
│  LAYER 1: Physical / Bare-metal provisioning             │
│    • MAAS (Metal-as-a-Service) — PXE boot, OS deploy     │
│    • Foreman + Ironic for hardware lifecycle             │
│    • Immutable OS images (built via Packer, signed)      │
│                                                          │
│  LAYER 2: Infrastructure provisioning                    │
│    • Terraform (network, storage, k8s clusters, DNS)    │
│    • Terragrunt for env DRY config                      │
│    • State in remote backend (S3 + DynamoDB lock)       │
│                                                          │
│  LAYER 3: Configuration management                       │
│    • Ansible (probe tuning: hugepages, CPU pinning,     │
│      NIC IRQ affinity, DPDK binding, sysctl)            │
│                                                          │
│  LAYER 4: Application orchestration                      │
│    • Kubernetes (app/stream/storage-operator tiers)     │
│    • Helm charts + ArgoCD (GitOps, app-of-apps pattern) │
│    • Operators: Strimzi (Kafka), ClickHouse Operator,   │
│      Flink Operator, Zalando Postgres Operator          │
│                                                          │
│  LAYER 5: Policy & Security                              │
│    • OPA/Gatekeeper (admission policies)                │
│    • Kyverno, Falco (runtime security)                  │
│    • Vault (secrets, PKI, dynamic DB creds)             │
└──────────────────────────────────────────────────────────┘
```

### 12.2 Example: Probe Tuning via Ansible (excerpt)

```yaml
- name: Configure DPDK hugepages
  sysctl:
    name: vm.nr_hugepages
    value: "256"        # 256 × 1GB pages
  
- name: Pin NIC IRQs to NUMA-local cores
  shell: |
    set_irq_affinity.sh -x local mlx5_core
    
- name: Isolate cores from scheduler (kernel cmdline)
  lineinfile:
    path: /etc/default/grub
    regexp: '^GRUB_CMDLINE_LINUX'
    line: 'GRUB_CMDLINE_LINUX="isolcpus=4-63,68-127 
           nohz_full=4-63,68-127 rcu_nocbs=4-63,68-127 
           default_hugepagesz=1G hugepagesz=1G hugepages=256 
           intel_iommu=on iommu=pt"'
  notify: regenerate grub & reboot
```

---

## 13. Auto-Scaling Strategy

### 13.1 What scales how

```
┌─────────────────┬──────────────────────────────────────────┐
│ Component       │ Scaling Approach                          │
├─────────────────┼──────────────────────────────────────────┤
│ Probe tier      │ MANUAL/PLANNED — bare metal, scale by     │
│                 │ adding NPB capacity + new probe nodes     │
│                 │ (capacity planning quarterly)             │
│                 │                                           │
│ Flink           │ Reactive autoscaling (Flink Autoscaler)   │
│                 │ based on Kafka lag + backpressure metrics │
│                 │                                           │
│ App/API tier    │ HPA (Horizontal Pod Autoscaler) on CPU + │
│                 │ custom metrics (req/s, p95 latency);     │
│                 │ KEDA for Kafka-lag-driven scaling        │
│                 │                                           │
│ ClickHouse      │ Vertical first; horizontal by re-sharding │
│                 │ (planned, since rebalancing is heavy)    │
│                 │                                           │
│ Kafka           │ Add brokers + Cruise Control auto-rebal. │
│                 │ partition reassignment                    │
│                 │                                           │
│ k8s nodes       │ Cluster Autoscaler (cloud) / fixed       │
│                 │ capacity pools (on-prem)                 │
└─────────────────┴──────────────────────────────────────────┘
```

### 13.2 KEDA Example — Scale API consumers on Kafka lag

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: enrichment-consumer-scaler
spec:
  scaleTargetRef:
    name: enrichment-service
  minReplicaCount: 6
  maxReplicaCount: 60
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: enrichment-grp
      topic: enriched.sessions
      lagThreshold: "50000"   # scale up when lag > 50k per replica
```

---

## 14. Security & Compliance

Telecom data is *extremely* sensitive (subscriber PII, lawful intercept, GDPR).

```
┌──────────────────────────────────────────────────────────┐
│  DATA PROTECTION                                          │
│   • IMSI/MSISDN hashed (HMAC-SHA256, keyed) at PROBE     │
│     before leaving capture tier — pseudonymization       │
│   • Reversible mapping vault accessible only to lawful-  │
│     intercept module under dual-control (4-eyes)         │
│   • Encryption at rest: LUKS (disks), TDE (ClickHouse), │
│     S3 SSE-KMS                                            │
│   • Encryption in transit: mTLS everywhere (SPIFFE/      │
│     SPIRE workload identity)                              │
│                                                          │
│  ACCESS CONTROL                                          │
│   • RBAC + ABAC; just-in-time access via Vault           │
│   • All DB access via PgBouncer/proxy with audit log    │
│   • Bastion + session recording for prod access         │
│                                                          │
│  COMPLIANCE                                              │
│   • GDPR: right-to-erasure workflows, data minimization │
│   • Lawful Intercept (ETSI/3GPP TS 33.108) module       │
│     air-gapped, separate access domain                   │
│   • SOC2 / ISO27001 audit logging (immutable, WORM S3)  │
│   • Data residency: regional data never leaves region   │
└──────────────────────────────────────────────────────────┘
```

---

## 15. End-to-End Data Flow (Putting it all together)

```
1. Subscriber attaches to 5G network → AMF/SMF signaling
                    │
2. Optical TAP mirrors N1/N2/N4 packets → Network Packet Broker
                    │  (flow-consistent hash by GTP-TEID/IMSI)
3. Packet Broker load-balances to Probe #17
                    │  (DPDK RX, NUMA-local processing)
4. Probe decodes NGAP+PFCP, builds session record,
   hashes IMSI, produces Protobuf → Kafka topic ngap.records
                    │  (partition = murmur2(imsi_hash) % 240)
5. Flink Job 1 consumes ngap + pfcp + gtpu (same partition),
   stateful-joins by IMSI → enriched session record
                    │  (RocksDB state, exactly-once)
6. Enrichment service adds subscriber profile (Redis lookup)
                    │
7. Record written to:
   • ClickHouse (analytics, 30-day hot)  → Grafana dashboards
   • Elasticsearch (searchable signaling) → troubleshooting UI
   • TimescaleDB (KPI rollups)            → SLA reports
                    │
8. Flink Job 3 CEP detects signaling storm on this cell
   → produces to "alerts" topic
                    │
9. Alerting Engine → PagerDuty + auto-ticket + Slack
                    │  (end-to-end < 2 seconds)
10. NOC engineer queries last 5 min of raw PCAP for the cell
    → on-demand export from probe ring buffer → root cause
```

---

## 16. Interesting / Advanced Features

1. **On-demand PCAP retrieval:** Each probe keeps a 60-second rolling full-packet buffer. The UI lets an engineer "rewind" and download the exact packets for a subscriber/cell/time window for deep forensic analysis — bridging metadata analytics with raw packet truth.

2. **Decoder hot-reload via WASM:** New/experimental protocol decoders can be deployed as sandboxed WASM modules to probes *without recompiling/redeploying* the whole probe binary, enabling rapid response to new 3GPP releases.

3. **Shadow-deploy decoders:** New decoder versions run in parallel with the old on staging's 5% live feed; outputs are diffed automatically to catch regressions before promotion.

4. **Chaos engineering:** Scheduled chaos (LitmusChaos) kills probes, Kafka brokers, and PG primaries in staging weekly to continuously validate HA assumptions.

5. **Cost/capacity model:** Storage tiering + ClickHouse compression (ZSTD + specialized codecs like `T64`, `DoubleDelta`, `Gorilla`) achieves ~12:1 compression, making 30-day hot retention of 85B records/day economically feasible.

6. **GitOps everything:** The *entire* platform state — including ClickHouse schemas, Kafka topics, Grafana dashboards, and alert rules — is declared in Git. A full DC rebuild is `terraform apply` + `argocd sync` + data restore.

---

## 17. Summary Architecture Diagram (Consolidated)

```
                    ┌──────────────────────────────────────┐
   TELECOM CORE     │  4G EPC  │  5G Core  │  IMS/VoLTE     │
   (live traffic)   └────┬─────────┬───────────┬───────────┘
                         │ TAP/SPAN│           │
                    ┌────▼─────────▼───────────▼──────┐
                    │  NETWORK PACKET BROKER (NPB)     │  Gigamon/Arista
                    └────────────────┬─────────────────┘
                                     │ balanced 100/400G
        ┌────────────────────────────┼────────────────────────────┐
        │  CAPTURE TIER (bare metal, DPDK, 40+ probes)            │
        │  decode • IMSI-hash • PCAP ring • Protobuf records      │
        └────────────────────────────┬───────────────────────────┘
                                      │
        ┌─────────────────────────────▼──────────────────────────┐
        │  KAFKA (24 brokers, 3x repl, tiered→S3, MM2→DR)        │
        └─────────────────────────────┬──────────────────────────┘
                                      │
        ┌─────────────────────────────▼──────────────────────────┐
        │  FLINK (60 TMs, RocksDB, exactly-once, autoscaled)     │
        │  correlation • KPI • CEP fraud/anomaly                 │
        └─────┬───────────┬───────────┬───────────┬──────────────┘
              │           │           │           │
        ┌─────▼───┐ ┌─────▼────┐ ┌────▼─────┐ ┌───▼──────┐
        │ClickHse │ │Timescale │ │ Elastic  │ │ Redis    │
        │48 nodes │ │ (PG/HA)  │ │ search   │ │ cluster  │
        │OLAP     │ │ KPIs     │ │ search   │ │ session  │
        └─────┬───┘ └──────────┘ └──────────┘ └──────────┘
              │           ┌──────────────┐    ┌──────────┐
              │           │ PostgreSQL   │    │ Ceph/S3  │
              │           │ (Patroni HA) │    │ cold+PCAP│
              │           │ config/RBAC  │    │ archive  │
              │           └──────────────┘    └──────────┘
        ┌─────▼────────────────────────────────────────────┐
        │  APP/API/UI TIER (k8s, ArgoCD, HPA/KEDA)         │
        │  REST/gRPC • React UI • Grafana • Alerting       │
        └─────────────────────┬─────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │  PagerDuty  │ NOC Engineers │ Reports │
              └─────────────────────────────────────┘

   CROSS-CUTTING:  Prometheus+Thanos | Loki | Tempo | Vault |
                   Terraform | Ansible | MAAS | OPA/Falco
   DR:  DC1 (primary) ⇄ DC2 (active-DR) → DC3 (cold/S3)
```

---

This gives you the full picture — from optical taps and DPDK kernel-bypass at the wire, through streaming correlation, into tiered analytical storage, wrapped in GitOps automation, multi-DC DR, and five-nines observability.

Would you like me to **zoom into any specific area** in even more depth — for example: the **exactly-once Flink state recovery mechanics**, the **ClickHouse resharding procedure**, the **detailed DR failover runbook**, or the **lawful-intercept compliance subsystem**?







This final version synthesizes the precise topology of your provided diagrams with the high-impact, industry-standard engineering depth from your original draft. It keeps the "big value" sections—specifically the **streaming semantics, storage efficiency, and deep observability**—which are critical for demonstrating senior-level competence, even if they aren't explicitly drawn as boxes in the architecture.

---

# Telecom Network Traffic Monitoring & Analytics Platform (NTMAP)

## A High-Scale, Federated, Cloud-Native Architecture

---

## 1. System Overview & Strategic Intent

NTMAP is a distributed, cloud-native observability platform designed to ingest multi-terabit mirrored traffic from 4G/5G core networks. Its primary differentiator is the **decoupling of the data-plane (Frontend) from the analytical-plane (NBE)**, using a sophisticated event-driven backbone to facilitate real-time anomaly detection, session correlation, and on-demand packet retrieval.

### Core Architecture Principles

* **Geo-Federation:** Instead of massive data duplication, we use **Elasticsearch Cross Cluster Search (CCS)** to query across regional pods.
* **Intelligent Capture:** We achieve regulatory-grade packet capture using **Live Session Filtering**, ensuring expensive raw PCAP storage is only consumed for high-value anomalies.
* **Exactly-Once Stream Processing:** Using Flink with RocksDB state backends, we perform stateful session stitching keyed by **IMSI-hashes**, ensuring control-plane and user-plane events for a single subscriber are processed contiguously.

---

## 2. The Three-Tiered Topology

### Tier 1: Frontend (FE) - The Acquisition Layer

* **vTap & CP ECS:** Acting as the ingestion gateway, this tier performs load-balanced packet stripping.
* **Processing Pipelines:** These pods decode complex telecom protocols (GTP, NGAP, Diameter) and produce high-fidelity metadata.
* **Live Session Filter (Type 2):** This performs the "heavy lifting" of traffic inspection. By pinning these to specific hardware resources (or utilizing high-performance DPDK-like paths where possible), we minimize packet drops at the edge.

### Tier 2: Routing/Backend Environment (RBE) - The Nervous System

* **Kafka Backbone:** Partitioning is the key to our scalability. By keying topics with `murmur2(IMSI-hash)`, we guarantee that every event for a subscriber lands on the same partition.
* **MirrorMaker 2:** This facilitates the asynchronous cross-DC replication essential for our active-active HA model.
* **Storage Tiers:**
* **Hot/Warm:** ClickHouse (OLAP) for CDR-like analytics and KPI rollups.
* **Search:** Elasticsearch (distributed clusters) for signaling logs.
* **Cold:** S3-compatible object store for long-term PCAP archive.



### Tier 3: Northbound Environment (NBE) - The Analytics Intelligence

* **ES Cross Cluster Search:** The "Smart Indexer" allows our Northbound APIs to aggregate data from local and GR-site Elasticsearch clusters without the overhead of bulk data replication.
* **Dynamic Config:** The brain of the platform. It translates user-requested troubleshooting sessions into filter rules, which are propagated down to the FE/RBE in real-time via the Message Bus.

---

## 3. High-Value Engineering Details (Beyond the Diagram)

While the diagram shows the flow, the following architectural choices are critical for production stability:

### 3.1 Streaming Semantics (Apache Flink)

We utilize Flink with **checkpointing to S3** every 30 seconds. By employing a `bounded-out-of-orderness` watermark strategy, we handle the inherent latency jitters of telecom signaling. This is essential for maintaining accurate **session-setup-time KPIs** when packets arrive out of order from different taps.

### 3.2 Tiered Storage & Cost Optimization

We do not store all data on expensive NVMe. Our ClickHouse and Elasticsearch clusters utilize **TTL-based data movement**.

* **Hot (0-7 days):** Local NVMe for instantaneous troubleshooting.
* **Warm (7-30 days):** SATA SSDs for trend analysis.
* **Cold (30-365 days):** S3-backed MergeTree tables.
This allows us to maintain 13 months of data with a storage cost structure that is 80% lower than a monolithic approach.

### 3.3 Production-Grade Observability

Beyond basic metrics, we implement **SLO-based alerting** using the *Sloth* generator.

* **Critical Path:** If the `Packet capture completeness` drops below 99.999%, the CI/CD pipeline *automatically freezes* all new feature deployments.
* **Distributed Tracing:** We use OpenTelemetry to trace a query from the `NBE API` through the `Cross Cluster Search` coordinating node, down into the `RBE ClickHouse shard`, identifying bottlenecks in the query plan before they hit the user.

### 3.4 Resilient Deployment (GitOps)

We utilize **ArgoCD with an App-of-Apps pattern**. Each pod (FE, RBE, NBE) is managed as a standalone helm release.

* **Schema Evolution:** For databases like ClickHouse and PostgreSQL, we use `expand-contract` migration patterns (e.g., adding columns without breaking existing queries), ensuring zero-downtime releases.
* **Canary Analysis:** Argo Rollouts performs automated canary analysis based on Prometheus metrics. If the "error-budget" for a canary version burns too fast, the traffic is automatically shifted back to the stable version without manual intervention.

---

## 4. Disaster Recovery (DR) Strategy

Our DR strategy is **Active-Active** by design:

* **Capture Layer:** Since raw packets are ephemeral, regional probes capture locally and persist to regional storage.
* **Event Layer:** Metadata topics are replicated via Kafka MirrorMaker to the GR site, meaning that even if the primary site disappears, the historical event metadata is queryable from the GR site via the federated Elasticsearch cluster.
* **Database HA:** Patroni (for PostgreSQL) and ClickHouse Keeper ensure that mastership is maintained across DC boundaries. In a site-wide failover, BGP anycast updates route traffic to the survivor, and the federated ES coordinating node automatically begins querying only the surviving GR cluster.

---

### Summary for the Technical Interview

This architecture is built for **telecom-grade reliability**. The combination of **distributed stream processing (Flink)**, **cross-cluster data federation (Elasticsearch CCS)**, and **dynamic rule propagation** allows us to provide massive visibility without the typical overhead of monolithic packet-capture platforms. I approach this architecture with an SRE mindset: assuming failure is imminent, designing for automatic recovery, and strictly enforcing SLOs at every pipeline stage.

**Would you like me to expand on the specific Flink state-management techniques used for the session-stitching, or perhaps dive deeper into the ClickHouse sharding strategy?**

