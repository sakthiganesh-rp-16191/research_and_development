# Module: Export, Integration & Privacy

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** The export layer is the interface between Tetragon's kernel telemetry and the rest of the security stack (SIEM, SOAR, detection pipeline). This module documents the gRPC/JSON event APIs, filtering mechanisms (field, CEL, redaction), and privacy controls for managing sensitive data in the event stream.

---

## Table of Contents

1. [gRPC Event Stream](#1-grpc-event-stream)
2. [JSON File Export](#2-json-file-export)
3. [Field Filters](#3-field-filters)
4. [Redaction Filters](#4-redaction-filters)
5. [Capability Export Filter](#5-capability-export-filter)
6. [in_init_tree & Container ID Export Filters](#6-in_init_tree--container-id-export-filters)
7. [CEL Export Filters](#7-cel-export-filters)
8. [Event Type Filtering](#8-event-type-filtering)
9. [Event Log Service](#9-event-log-service)
10. [Unix Domain Socket Default](#10-unix-domain-socket-default)
11. [Cluster Name & Node Identifiers](#11-cluster-name--node-identifiers)
12. [Privacy & Data Minimization](#12-privacy--data-minimization)
13. [SIEM / EDR Pipeline Considerations](#13-siem--edr-pipeline-considerations)
14. [Deprecated & Removed Export APIs](#14-deprecated--removed-export-apis)

---

## 1. gRPC Event Stream

### `GetEvents` gRPC API — **[NEW v0.8.0]**

The primary event consumption interface. Clients connect to the Tetragon agent's gRPC endpoint and receive a stream of `GetEventsResponse` messages containing:
- `process_exec` events
- `process_exit` events
- `process_kprobe` events (from TracingPolicy kprobe rules)
- `process_tracepoint` events
- `process_uprobe` events
- `process_lsm` events
- `process_loader` events (binary loading)
- `test` events (diagnostics)

**Event filter options on `GetEvents` request:**
- Event types to include/exclude
- Pod name / namespace filter
- Policy name filter ([ENH v1.1.0]: filter by tracing policy name [#1867](https://github.com/cilium/tetragon/pull/1867))
- Container name regex filter ([ENH v1.6.0]: [#4051](https://github.com/cilium/tetragon/pull/4051))

**Reconnect option — [ENH v1.4.0]:** `--reconnect` flag for `tetra getevents` to automatically reconnect on connection loss ([#3438](https://github.com/cilium/tetragon/pull/3438)).

**gRPC server address change — [BREAK v1.7.0]:** Default changed from `localhost:54321` (TCP) to `/var/run/tetragon/tetragon.sock` (Unix socket). See §10.

---

## 2. JSON File Export

### JSON export to file — **[NEW v0.8.0]**

Tetragon can write JSON-encoded events to a file for log-file-based ingestion.

**Security fix — [BREAK v1.0.0]:** Export file permissions default changed to `0600` ([#1575](https://github.com/cilium/tetragon/pull/1575)) to prevent unauthorized reads of the event log.

**Permission rotation fix — [FIX v1.3.0]:** If the export file exists with different permissions than specified, Tetragon now corrects permissions on next log rotation. Before v1.3.0, permissions of the existing file were preserved on rotation ([upgrade note](https://github.com/cilium/tetragon/releases/tag/v1.3.0)).

**stdout export with env injection — [ENH v1.6.0]:** Support for injecting environment variables from Kubernetes Secrets into the export stdout process ([#4025](https://github.com/cilium/tetragon/pull/4025)).

---

## 3. Field Filters

### Field filters — **[NEW v0.8.0; hardened v1.1.0]**

Field filters allow selective inclusion or exclusion of fields from exported events, enabling:
- Removal of high-cardinality fields that are unnecessary for a specific use case
- Privacy-compliant event minimization (exclude PII-carrying fields)
- Reduction of event payload size for bandwidth-constrained pipelines

**Reliability fixes in v1.1.0 (critical hardening):**
- [FIX v1.1.0] Segfault when filtering PID information ([#1700](https://github.com/cilium/tetragon/pull/1700))
- [FIX v1.1.0] Multiple segfaults in field filter configuration ([#1712](https://github.com/cilium/tetragon/pull/1712))
- [FIX v1.1.0] Performance improvements for field filters ([#1763](https://github.com/cilium/tetragon/pull/1763), [#1762](https://github.com/cilium/tetragon/pull/1762))
- [FIX v1.1.0] Top-level information missing from events due to regression ([#1882](https://github.com/cilium/tetragon/pull/1882))
- Use a message copy when applying fieldFilters in exec events to avoid data corruption ([#1432](https://github.com/cilium/tetragon/pull/1432))

**Incorrect event types in field filter examples — [FIX v1.4.0]:** Docs fix for field filter configuration ([#3489](https://github.com/cilium/tetragon/pull/3489)).

---

## 4. Redaction Filters

### Redaction Filters — **[NEW v1.1.0]**

**Evidence:** [#2243](https://github.com/cilium/tetragon/pull/2243)

Redaction filters censor sensitive string data in process events before export. Use cases:
- Redact passwords from command-line arguments (e.g., `--****** `)
- Remove API keys or tokens from environment variables
- Mask credentials from process argument strings

**How it works:** Regex-based patterns match against string fields in events. Matching portions are replaced with a redaction placeholder.

**EDR privacy value:**
- Prevents plaintext credentials from flowing into SIEM/log storage
- Enables logging of security-relevant exec events without storing sensitive argument values
- Supports compliance requirements (GDPR, SOC2, PCI-DSS) around sensitive data in logs

---

## 5. Capability Export Filter

### Capability-based export filter — **[NEW v1.1.0]**

**Evidence:** [#2107](https://github.com/cilium/tetragon/pull/2107)

Filter events by the Linux capabilities held by the triggering process. Enables:
- Scoping export to only privilege-escalation-relevant events
- Dashboards focused on `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, etc.
- Reducing SIEM ingest volume for capability-unrelated events

---

## 6. in_init_tree & Container ID Export Filters

### `in_init_tree` export filter — **[NEW v1.3.0]**

**Evidence:** [#3209](https://github.com/cilium/tetragon/pull/3209)

Filter the event stream to include only events where `in_init_tree` is `true` (container-init-managed processes) or `false` (processes that bypassed the container init tree). Useful for:
- Scoping container-escape detection streams to non-init-tree events
- Reducing noise from expected container processes

### `container_id` export filter — **[NEW v1.3.0]**

**Evidence:** [#3209](https://github.com/cilium/tetragon/pull/3209)

Filter the event stream to a specific container by container ID.

---

## 7. CEL Export Filters

### CEL (Common Expression Language) filter — **[NEW v1.3.0]**

**Evidence:** [#3098](https://github.com/cilium/tetragon/pull/3098)

CEL filters evaluate arbitrary expressions against events at export time. All event fields are accessible, and the filter drops events that evaluate to `false`.

**IP/CIDR helpers — [ENH v1.3.0]:** ([#3211](https://github.com/cilium/tetragon/pull/3211)) IP address and CIDR range operations available in CEL expressions:
```
destination.ip.isInRange("10.0.0.0/8")
source.ip == "192.168.1.100"
```

**CEL CLI filter — [ENH v1.4.0]:** ([#3124](https://github.com/cilium/tetragon/pull/3124)) CEL filter available in `tetra getevents --cel-filter` CLI flag.

**CEL in BPF — [NEW v1.7.0]:** In-kernel CEL evaluation (see [detection-policy-engine.md](detection-policy-engine.md#10-cel-filters--in-bpf-cel-evaluation)).

---

## 8. Event Type Filtering

### Event type filter in CLI — **[ENH v1.0.0]**

**Evidence:** [#1549](https://github.com/cilium/tetragon/pull/1549)

Filter events by type in the `tetra getevents` command (e.g., only show `process_exec`, only show `process_kprobe` events).

### PROCESS_TRACEPOINT in exported events — **[ENH v1.0.0]**

**Evidence:** [#1684](https://github.com/cilium/tetragon/pull/1684)

`PROCESS_TRACEPOINT` event type added to the default exported events list in Helm configuration.

### Policy name filter — **[ENH v1.1.0]**

Filter events by the TracingPolicy that generated them (`--policy-names` flag) ([#1867](https://github.com/cilium/tetragon/pull/1867)).

**[FIX v1.3.0]** `--policy-names` now correctly applies to all event types ([#3044](https://github.com/cilium/tetragon/pull/3044)).

---

## 9. Event Log Service

### Event Log Service — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Tetragon v1.7.0 introduces an event log service that provides:
- Structured event logging to files or external log sinks
- Integration point for SIEM/log forwarding pipelines
- Consistent event formatting across deployment modes

---

## 10. Unix Domain Socket Default

### Unix socket default for agent API — **[BREAK v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

The default Tetragon agent server address changed from:
- **Before v1.7.0:** `localhost:54321` (TCP loopback)
- **v1.7.0+:** `/var/run/tetragon/tetragon.sock` (Unix domain socket)

**Security improvement:** Unix sockets are accessible only to processes with appropriate filesystem permissions, unlike TCP loopback which is accessible to any local process regardless of user. This reduces the attack surface for unauthorized access to the Tetragon API.

**Migration:** Tools connecting to `localhost:54321` must be updated to connect to the Unix socket path. The `tetra` CLI handles this transparently ([#967](https://github.com/cilium/tetragon/pull/967)).

---

## 11. Cluster Name & Node Identifiers

### `cluster_name` field — **[NEW v1.3.0]**

**Evidence:** [#3025](https://github.com/cilium/tetragon/pull/3025)

An optional `cluster_name` field is now included in `GetEventsResponse` messages. This enables:
- Multi-cluster SIEM correlation without requiring separate agent configurations
- Disambiguation of events when multiple clusters forward to the same SIEM instance

**Node name in standalone mode — [ENH v1.1.0]:** ([#2123](https://github.com/cilium/tetragon/pull/2123)) Node name set to hostname when running outside Kubernetes.

**Centralized node name logic — [ENH v1.3.0]:** ([#3024](https://github.com/cilium/tetragon/pull/3024)) Logic to set the node name centralized for consistency.

---

## 12. Privacy & Data Minimization

### Privacy controls summary

| Control | Available from | Purpose |
|---------|---------------|---------|
| Field filters | v0.8.0 (hardened v1.1.0) | Remove unnecessary fields |
| Redaction filters | v1.1.0 | Mask sensitive string values |
| Export file permissions (0600) | v1.0.0 | Protect export file from reads |
| Unix socket API | v1.7.0 | Restrict API access by filesystem permissions |
| Metrics label filter | v1.0.0 ([#1444](https://github.com/cilium/tetragon/pull/1444)) | Exclude sensitive labels from metrics |

**`--expose-kernel-addresses` removed — [REM v1.3.0]:** ([#3042](https://github.com/cilium/tetragon/pull/3042)) This flag, which could leak kernel addresses into event logs, was permanently removed.

---

## 13. SIEM / EDR Pipeline Considerations

### Integration Patterns

**Pattern 1: gRPC consumer (recommended for real-time)**
```
Tetragon Agent (gRPC server) → gRPC consumer → SIEM/EDR pipeline
```
- Low latency
- Supports filtering at the client (`GetEvents` request filters)
- Requires gRPC-capable consumer

**Pattern 2: JSON file export + log forwarder**
```
Tetragon Agent → /var/log/tetragon.json → Filebeat/Fluentd/Vector → SIEM
```
- Works with any log forwarder
- Higher latency
- Simpler integration with existing log pipelines

**Pattern 3: Unix socket consumer**
```
Tetragon Agent (Unix socket) → local consumer → SIEM
```
- Secure local access
- Preferred in v1.7.0+ environments

### SIEM Field Mapping

Key fields for SIEM enrichment:

| Tetragon Field | SIEM Mapping |
|----------------|-------------|
| `process.binary` | `process.executable` (ECS) |
| `process.arguments` | `process.args` (ECS) |
| `process.uid` | `user.id` (ECS) |
| `process.username` | `user.name` (ECS) |
| `pod.namespace` | `kubernetes.namespace` |
| `pod.name` | `kubernetes.pod.name` |
| `pod.workload` | `kubernetes.deployment` |
| `policy_name` | Rule/detection name |
| `cluster_name` | `cloud.instance.name` or custom |

### Data Volume Management

For production deployments:
1. Apply rate limiting in TracingPolicy rules
2. Use CEL export filters to drop low-value events before transmission
3. Use field filters to reduce payload size
4. Use redaction filters for compliance
5. Monitor `tetragon_export_ratelimit_events_dropped_total` for tuning

---

## 14. Deprecated & Removed Export APIs

| Item | Removed | Notes |
|------|---------|-------|
| `--expose-kernel-addresses` flag | v1.3.0 | Security: prevented kernel address leakage |
| `--pprof-addr` flag | v1.3.0 | Use standard Go pprof tooling |
| `pod.labels` event field | v1.1.0 | Replaced by `pod.pod_labels` |
| TCP default server address | v1.7.0 (changed) | Now Unix socket `/var/run/tetragon/tetragon.sock` |

---

*Related: [Detection Policy Engine](detection-policy-engine.md) · [Process & Lineage](process-and-lineage.md) · [Kubernetes & Container Context](kubernetes-and-container-context.md)*
