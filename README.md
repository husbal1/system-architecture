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

**NTMAP** is a distributed, cloud-native observability platform designed to ingest multi-terabit mirrored traffic from 4G/5G core networks. Its primary differentiator is the **decoupling of the data-plane (Frontend) from the analytical-plane (NBE)**, using a sophisticated event-driven backbone to facilitate real-time anomaly detection, session correlation, and on-demand packet retrieval.

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

<img width="1606" height="979" alt="3bc3f797-9a7a-4643-8069-9fa5855191b3" src="https://github.com/user-attachments/assets/f33e93e0-1b2f-423e-97c9-416d30bb3168" />

### Core Architecture Principles

* **Geo-Federation:** Instead of massive data duplication, we use **Elasticsearch Cross Cluster Search (CCS)** to query across regional pods.
* **Intelligent Capture:** We achieve regulatory-grade packet capture using **Live Session Filtering**, ensuring expensive raw PCAP storage is only consumed for high-value anomalies.

---

## 3. The Three-Tiered Topology

### Tier 1: Frontend (FE) - The Acquisition Layer

* **vTap & CP ECS:** Acting as the ingestion gateway, this tier performs load-balanced packet stripping.
* **Processing Pipelines:** These pods decode complex telecom protocols (GTP, NGAP, Diameter) and produce high-fidelity metadata.
* **Live Session Filter (Type 2):** This performs the "heavy lifting" of traffic inspection. By pinning these to specific hardware resources (or utilizing high-performance DPDK-like paths where possible), we minimize packet drops at the edge.

### Tier 2: Routing/Backend Environment (RBE) - The Nervous System

* **Kafka Backbone:** Partitioning is the key to our scalability. By keying topics with `murmur2(IMSI-hash)`, we guarantee that every event for a subscriber lands on the same partition.
* **MirrorMaker 2:** This facilitates the asynchronous cross-DC replication essential for our active-active HA model.
* **Storage Tiers:**
* **Hot/Warm:** ElasticSearch Data Teiring
* **Search:** Elasticsearch (distributed clusters) for signaling logs.
* **Cold:** S3-compatible object store for long-term PCAP archive.

### Tier 3: National Backend Environment (NBE) - The Analytics Intelligence

* **ES Cross Cluster Search:** The "Smart Indexer" allows the APIs to aggregate data from local and GR-site Elasticsearch clusters without the overhead of bulk data replication.
* **Dynamic Config:** The brain of the platform. It translates user-requested troubleshooting sessions into filter rules, which are propagated down to the FE/RBE in real-time via the Message Bus.

---

## 4. High-Value Engineering Details (Beyond the Diagram)

While the diagram shows the flow, the following architectural choices are critical for production stability:

### 4.1 Streaming Semantics (Apache Flink)

We utilize Flink with **checkpointing to S3** every 30 seconds. By employing a `bounded-out-of-orderness` watermark strategy, we handle the inherent latency jitters of telecom signaling. This is essential for maintaining accurate **session-setup-time KPIs** when packets arrive out of order from different taps.

### 4.2 Tiered Storage & Cost Optimization

We do not store all data on expensive NVMe. Our Elasticsearch clusters utilize **TTL-based data movement**.

* **Hot (0-7 days):** Local NVMe for instantaneous troubleshooting.
* **Warm (7-30 days):** SATA SSDs for trend analysis.
* **Cold (30-365 days):** S3-backed MergeTree tables.
This allows us to maintain 13 months of data with a storage cost structure that is 80% lower than a monolithic approach.

### 4.3 Resilient Deployment (GitOps)

We utilize **ArgoCD with an App-of-Apps pattern**. Each pod (FE, RBE, NBE) is managed as a standalone helm release.

* **Canary Analysis:** Argo Rollouts performs automated canary analysis based on Prometheus metrics. If the "error-budget" for a canary version burns too fast, the traffic is automatically shifted back to the stable version without manual intervention.

---

## 5. Disaster Recovery (DR) Strategy

Our DR strategy is **Active-Active** by design:

* **Capture Layer:** Since raw packets are ephemeral, regional probes capture locally and persist to regional storage.
* **Event Layer:** Metadata topics are replicated via Kafka MirrorMaker to the GR site, meaning that even if the primary site disappears, the historical event metadata is queryable from the GR site via the federated Elasticsearch cluster.

---

---

## 6. The Capture / Probe Tier (The Most Critical Layer)

This tier is **bare-metal** or **virtualized** with SRIOV & DPDK to support high throughputs.

### 6.1 Software Stack on Probe

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

## 7. Environments: Dev / Test / Staging / Production

We run **four distinct environments**, each isolated by network, IAM, and cloud account/datacenter.

### 7.1 Environment Matrix

```
┌─────────────┬──────────────┬─────────────┬──────────────┬───────────────┐
│  Aspect     │     DEV      │    TEST/QA  │   STAGING    │  PRODUCTION   │
├─────────────┼──────────────┼─────────────┼──────────────┼───────────────┤
│ Location    │ Cloud (k8s)  │ Cloud (k8s) │ On-prem DC2  │ On-prem DC1+DC2│
│ Traffic     │ Synthetic    │ Replayed    │ 5% live mirror│ 100% live     │
│             │ generators   │ PCAP corpus │ (shadow tap) │  mirror       │
│ Scale       │ 1 probe (VM) │ 2 probes(VM)│ 4 probes(HW) │ 40+ probes(HW)│
│ Kafka       │ 3 brokers    │ 3 brokers   │ 6 brokers    │ 24 brokers    │
│ Data        │ Fully synthetic│ Anonymized │ Anonymized   │ Real (encrypted)│
│ Refresh     │ On commit    │ Nightly     │ Per release  │ Controlled    │
│ Uptime SLA  │ None         │ None        │ 99%          │ 99.999%       │
└─────────────┴──────────────┴─────────────┴──────────────┴───────────────┘
```

### 7.2 Why Staging Uses a "Shadow Tap"

Staging receives a **5% statistically-sampled mirror** of live production traffic (IMSI-anonymized at the probe). This lets us validate new decoders and detection rules against *real* protocol behaviors and malformed packets that synthetic generators can't reproduce — without risking production.

### 7.3 Traffic Generation for Dev/Test

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

## 8. Production Deployment Strategy

- **Applications (k8s tier):** Argo Rollouts **canary** — 5% → 25% → 50% → 100%, with automatic rollback if SLO error budget burns (Prometheus-based analysis).
- **Probes (bare-metal):** **Rolling, NPB-coordinated.** Because probes are stateless-ish capture nodes behind the packet broker, we drain one probe at a time. The NPB redistributes its flow-hash buckets to peers.

---

## 9. Streaming / Message Bus Tier (Apache Kafka)

### 9.1 Kafka Topology

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

### 9.2 Why partition by IMSI-hash?

Subscriber session correlation requires that the control-plane event (e.g., GTP-C `Create Session`) and user-plane stats (GTP-U byte counters) for the *same subscriber* be processed by the *same* Flink task. Keying by IMSI-hash guarantees co-location, enabling **exactly-once stateful joins** without cross-task shuffles.

---

## 10. Monitoring, Observability & Alerting

### 10.1 The Observability Stack ("monitoring the monitor")

```
┌───────────────────────────────────────────────────────────┐
│  METRICS:  Prometheus (federated) + Thanos (long-term)   │
│    • Probe drop counters, mbuf usage, NUMA balance        │
│    • Kafka lag (Burrow)      │
│    • Node exporter, DCGM (GPU), NIC stats                 │
│                                                          │
│  LOGS:     Loki + Promtail (app logs)                    │
│            Elasticsearch (signaling/audit logs)          │
│                                                          │
│  DASHBOARDS: Grafana (200+ dashboards)                   │
│                                                          │
│  ALERTING: Alertmanager → PagerDuty / Opsgenie          │
│            + Slack + auto-ticket (Jira/ServiceNow)       │
│                                                          │
│  SLO TRACKING: Sloth-generated SLO rules + error budgets │
└───────────────────────────────────────────────────────────┘
```

### 10.2 Key SLOs & Golden Signals

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

## 11. Auto-Scaling Strategy

### 11.1 What scales how

```
┌─────────────────┬──────────────────────────────────────────┐
│ Component       │ Scaling Approach                          │
├─────────────────┼──────────────────────────────────────────┤
│ Probe tier      │ MANUAL/PLANNED — bare metal, scale by     │
│                 │ adding NPB capacity + new probe nodes     │
│                 │ (capacity planning quarterly)             │
│                 │                                           │
│                 │                                           │
│ App/API tier    │ HPA (Horizontal Pod Autoscaler) on CPU + │
│                 │ custom metrics (req/s, p95 latency);     │
│                 │ KEDA for Kafka-lag-driven scaling        │
│                 │                                           │
│                 │                                           │
│ k8s nodes       │ Cluster Autoscaler (cloud) / fixed       │
│                 │ capacity pools (on-prem)                 │
└─────────────────┴──────────────────────────────────────────┘
```

## 12. Security & Compliance

Telecom data is *extremely* sensitive (subscriber PII, lawful intercept, GDPR).

```
┌──────────────────────────────────────────────────────────┐
│  DATA PROTECTION                                         │
│   • IMSI/MSISDN hashed (HMAC-SHA256, keyed) at PROBE     │
│     before leaving capture tier — pseudonymization       │
│   • Encryption at rest: LUKS (disks), etc.TDE            │
│   • Encryption in transit: mTLS everywhere               │
│                                                          │
│  ACCESS CONTROL                                          │
│   • RBAC + ABAC; just-in-time access via Vault           │
│   • All DB access with audit log                         │
│   • session recording for prod access                    │
│                                                          │
│  COMPLIANCE                                              │
│   • GDPR: right-to-erasure workflows, data minimization  │
│   • Lawful Intercept (ETSI/3GPP TS 33.108) module        │
│     air-gapped, separate access domain                   │
│   • SOC2 / ISO27001 audit logging (immutable, WORM S3)   │
│   • Data residency: regional data never leaves region    │
└──────────────────────────────────────────────────────────┘
```

---

