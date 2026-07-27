# EDR Adoption Guide — Tetragon

> **Navigation:** [README](README.md) | [Matrix](edr-capability-matrix.md) | [Release Timeline](release-timeline.md) | [Sources](sources.md)
>
> **Purpose:** A practical, phased adoption guide for teams deploying Tetragon as the telemetry and enforcement foundation for an EDR program. Tetragon is an **EDR building block**, not a complete EDR product — this guide explains what Tetragon provides, what you must build around it, and how to progress safely from observation to full enforcement.

---

## Table of Contents

1. [What Tetragon Is (and Is Not)](#1-what-tetragon-is-and-is-not)
2. [Prerequisites](#2-prerequisites)
3. [Phase 1: Observe (Baseline Telemetry)](#3-phase-1-observe-baseline-telemetry)
4. [Phase 2: Tune (Reduce Noise)](#4-phase-2-tune-reduce-noise)
5. [Phase 3: Detect (Targeted Detection Rules)](#5-phase-3-detect-targeted-detection-rules)
6. [Phase 4: Validate (Telemetry Health & Coverage)](#6-phase-4-validate-telemetry-health--coverage)
7. [Phase 5: Enforce (Prevention & Response)](#7-phase-5-enforce-prevention--response)
8. [Phase 6: Integrate (SIEM / SOC Pipeline)](#8-phase-6-integrate-siem--soc-pipeline)
9. [Telemetry Health Checks](#9-telemetry-health-checks)
10. [What to Build Around Tetragon](#10-what-to-build-around-tetragon)
11. [Kernel Version Planning](#11-kernel-version-planning)

---

## 1. What Tetragon Is (and Is Not)

### Tetragon IS:

- An **eBPF-based security telemetry and enforcement agent** for Linux/Kubernetes
- A **detection substrate** — it provides the raw events that drive detection rules
- An **enforcement engine** — it can kill processes, override syscalls, and enforce at LSM hooks
- A **Kubernetes-native** observability tool with pod/namespace/workload context
- An **EDR building block** that provides the visibility layer of an EDR architecture

### Tetragon IS NOT:

| Capability | Reality |
|-----------|---------|
| Complete EDR product | ❌ Requires integration with detection logic, alerting, case management |
| SIEM | ❌ Events must be forwarded to an external SIEM for correlation |
| Malware scanner | ❌ No signature-based malware detection; detection is behavior-based via TracingPolicy |
| Network firewall | ❌ Can detect network connections but does not natively modify iptables/nftables |
| File quarantine | ❌ Can block file writes via Override/Signal but cannot isolate/quarantine files |
| Rollback engine | ❌ Cannot roll back file changes, process effects, or network connections |
| Case management | ❌ Alert management requires external tooling |
| Remote remediation | ❌ No built-in remediation workflows; enforcement is local kill/override |
| Windows EDR | ⚠️ Windows support (v1.5.0+) limited to process create/exit events |

---

## 2. Prerequisites

### Kernel Requirements

| Tetragon Feature | Minimum Kernel |
|-----------------|---------------|
| Core process/file/network telemetry | ≥4.19 |
| kprobe multi (multi-link) | ≥5.10 |
| LSM enforcement | ≥5.7 (`CONFIG_BPF_LSM=y`) |
| fentry hooks | ≥5.5 (with BTF) |
| BPF ring buffer | ≥5.11 (recommended) |
| USDT probes | ≥3.5 + instrumented binaries |
| BTF / CO-RE | ≥5.2 (`CONFIG_DEBUG_INFO_BTF=y`) |

**Run `tetra probe config`** (v1.6.0+) to verify kernel compatibility before deployment.

### Infrastructure Requirements

- Kubernetes ≥1.20 (for full CRD-based policy management) or standalone Linux host
- CRI-O ≥1.26 or containerd ≥1.7 for NRI hook support
- Prometheus/Grafana for metrics observability (optional but strongly recommended)
- SIEM or log aggregation for event consumption

### Tetragon Version

**Recommendation:** Deploy ≥v1.6.1 (latest patch of v1.6.x) or v1.7.0 for:
- Memory leak fix (v1.6.1)
- BPF ring buffer default (v1.6.0)
- USDT support (v1.6.0)
- Full feature set including fentry, CEL-in-BPF, matchParentBinaries, env vars (v1.7.0)

---

## 3. Phase 1: Observe (Baseline Telemetry)

**Goal:** Deploy Tetragon and establish a baseline of process, file, and network events without any detection rules or enforcement.

### Steps

1. **Deploy Tetragon** with default configuration (DaemonSet on all nodes)
2. **Enable baseline events:**
   - Process exec/exit (always-on)
   - Network connections (via built-in or example policies)
3. **Forward events** to SIEM/log storage (gRPC consumer or JSON file + log forwarder)
4. **Do not enable enforcement** at this stage

### Baseline TracingPolicy (read-only, observe mode)

Use the policy examples from the Tetragon repository:
- [`examples/tracingpolicy/`](https://github.com/cilium/tetragon/tree/main/examples/tracingpolicy) — official example policies
- TCP listen/connect examples
- File access examples

```yaml
# Minimal observe-only process exec policy (no action = events only)
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: observe-exec
spec:
  kprobes:
    - call: "security_bprm_check"
      syscall: false
      return: false
      args:
        - index: 0
          type: "linux_binprm"
      selectors:
        - matchActions:
            - action: Post  # emit event only, no enforcement
```

### What you get in Phase 1

- Full process tree visibility (exec, exit, fork)
- Network connection telemetry with process context
- Kubernetes pod/namespace/workload attribution
- File access events (from any TracingPolicy file rules)

---

## 4. Phase 2: Tune (Reduce Noise)

**Goal:** Reduce event volume to a manageable level without losing security-relevant signals.

### Noise Reduction Techniques

| Technique | Implementation |
|-----------|---------------|
| Rate limiting | Add `rateLimit` to high-frequency TracingPolicy rules |
| Binary scoping | Use `matchBinaries` to focus on security-relevant binaries |
| Argument filtering | Use `matchArgs` to narrow on specific paths or patterns |
| CEL export filter | Drop low-value events before SIEM ingestion |
| Field filters | Remove unnecessary fields to reduce payload size |
| Redaction filters | Mask credentials before storage |

### Rate Limiting Example

```yaml
selectors:
  - matchBinaries:
      - operator: In
        values:
          - "/bin/bash"
          - "/bin/sh"
    matchActions:
      - action: Post
        rateLimit: "5m"           # max 1 event per 5 minutes
        rateLimitScope: "process" # per-process limit
```

### Redaction Filter Example

```yaml
# In tetragon configuration / Helm values
exportFilters:
  redactionFilters:
    - match:
        - regex: "--****** ]+"
          redact: "--******"
```

---

## 5. Phase 3: Detect (Targeted Detection Rules)

**Goal:** Deploy purpose-built TracingPolicy rules targeting known attack techniques.

### Priority Detection Rules for EDR

Start with the highest-value detections:

#### Tier 1: Critical — Deploy First

| Detection | Policy Focus | Tetragon Feature |
|-----------|-------------|-----------------|
| Container escape via Docker socket | `connect` to `/var/run/docker.sock` | AF_UNIX path (v1.7.0) |
| Fileless execution (memfd) | `process_exec` with anonymous binary | Anonymous binary detection (v1.1.0) |
| Privilege escalation | `process_exec` with UID/capability changes | Credentials (v1.0.0), matchCapabilities (v1.1.0) |
| Reverse shell | Shell binary with inbound `accept()` | Process + network events |
| Kubernetes credential file access | Read of `/etc/kubernetes/*.conf`, `~/.kube/config` | File path matching |

#### Tier 2: High Value — Deploy Second

| Detection | Policy Focus | Tetragon Feature |
|-----------|-------------|-----------------|
| Suspicious child processes | `matchParentBinaries` for web → shell spawns | matchParentBinaries (v1.7.0) |
| Kernel module loading | `init_module` / `finit_module` kprobes | Module tracking (v1.0.0) |
| Sensitive file writes | Write to `/etc/passwd`, `/etc/sudoers`, cron | File monitoring |
| BPF rootkit detection | `bpf()` syscall from unexpected processes | BPF prog argument type (v1.6.0) |
| Credential theft via env vars | `AWS_*`, `KUBECONFIG` in child process env | Env vars (v1.7.0) |

#### Tier 3: Good Coverage — Deploy Third

| Detection | Policy Focus | Tetragon Feature |
|-----------|-------------|-----------------|
| Binary integrity (hash) | LSM events with IMA hash | IMA hashes (v1.3.0) |
| Unexpected network outbound | Shell/interpreter processes making TCP connections | TCP + process context |
| SUID execution | Execution of SUID binaries by non-root users | Credentials (v1.0.0) |
| Process injection indicators | Unexpected user-space call stacks | User stack traces (v1.1.0) |

### Using Monitoring Mode for Safe Deployment

**Always deploy new detection rules in monitoring mode first:**

```yaml
spec:
  options:
    - name: "disable-enforcement"
      value: "true"    # monitoring mode — events only, no enforcement
```

Validate for 1–2 weeks, review for false positives, then switch to enforcement mode.

---

## 6. Phase 4: Validate (Telemetry Health & Coverage)

**Goal:** Verify that Tetragon is producing reliable, complete telemetry and identify any blind spots.

### Telemetry Health Checks

#### Check 1: Event Delivery Rate

```
Metric: tetragon_missed_events_total{type="..."}
Alert: > 0 for extended periods = ring buffer overflow or BPF program issue
Action: Increase --rb-size or add rate limiting to high-volume policies
```

#### Check 2: Process Cache Health

```
Metric: tetragon_event_cache_retries_total
Metric: tetragon_event_cache_parent_info_errors_total
Alert: Sustained high values = slow K8s API or CRI socket issues
Action: Tune event cache retry count/delay; check CRI socket config
```

#### Check 3: Enforcement Reliability

```
Metric: tetragon_enforcer_missed_notifications_total
Alert: Any non-zero value when enforcement policies are active
Action: Check enforcer sensor load status; review BPF verifier logs
```

#### Check 4: Policy Load Status

```
Command: tetra tp list
Check: All expected policies show "enabled" status
Alert: Policies in "error" or "disabled" state
```

#### Check 5: Pod Context Coverage

```
Query: Count of events with missing pod metadata
Alert: High rate of events without pod.namespace = cgroup mapping failure
Action: Check CRI socket access; verify cgroupv2 configuration
```

### Coverage Gap Analysis

| Gap Area | Detection | Mitigation |
|----------|-----------|-----------|
| Pre-Tetragon processes | Events without full lineage | `/proc` scan at startup; event cache enrichment |
| Short-lived processes | Missed exec events | Tune ring buffer; add process start tracepoint |
| Kernel threads | May appear as `kthread` parent | Expected behavior; filter with matchBinaries |
| Encrypted traffic payload | Only connection metadata | Use uprobes on TLS libraries for plaintext |
| Windows hosts | Limited to process events | Use Windows EDR for Windows coverage |

---

## 7. Phase 5: Enforce (Prevention & Response)

**Goal:** Enable enforcement actions for the highest-confidence detections with low false-positive rates.

### Enforcement Decision Criteria

Before enabling enforcement on a rule:

| Question | Required Answer |
|----------|----------------|
| Is the false-positive rate < 0.1% in monitoring mode? | Yes |
| Has the rule been validated in staging for ≥1 week? | Yes |
| Is the enforcement impact bounded (single process kill vs. service disruption)? | Yes |
| Is there a runbook for handling enforcement-related incidents? | Yes |
| Are action counters monitored and alerting configured? | Yes |

### Enforcement Action Selection

| Scenario | Recommended Action | Rationale |
|----------|-------------------|-----------|
| Container escape via socket | Signal (SIGKILL) + event | Async acceptable; socket connect already happened |
| Fileless execution | Override on `execve` / NotifyEnforcer on `security_bprm_check` | Synchronous block before execution |
| Kernel module load | Override on `init_module` / `security_kernel_module_request` | Block before module is loaded |
| Suspicious inbound shell | Signal (SIGKILL) | Process-level termination |
| File write to critical path | Override on `vfs_write` / `security_file_permission` | Block write synchronously |

### Persistent Enforcement Configuration

For production environments where Tetragon may restart during upgrades:

```yaml
# Helm values
tetragon:
  persistentEnforcement: true
```

This ensures enforcement policies remain active during Tetragon restart cycles.

---

## 8. Phase 6: Integrate (SIEM / SOC Pipeline)

**Goal:** Connect Tetragon telemetry to the broader security operations workflow.

### Integration Architecture

```
Tetragon Agent
    │
    ├── gRPC stream ──→ EDR Agent / SIEM collector
    │                       │
    │                       ├── Alert correlation rules
    │                       ├── Threat intel enrichment
    │                       ├── Alert → ticket creation
    │                       └── SOAR playbook trigger
    │
    └── JSON file ──→ Log forwarder (Filebeat/Fluent Bit) ──→ SIEM
```

### Key Integration Points

1. **Alert enrichment:** Add threat intelligence to IP/hash fields from Tetragon events
2. **Kubernetes context join:** Join with K8s audit logs using pod UID (v1.6.0+) and pod name
3. **Multi-cluster correlation:** Use `cluster_name` field (v1.3.0+) for multi-cluster SIEM rules
4. **Policy attribution:** `policy_name` in events enables rule-level alert attribution
5. **Incident response:** Tetragon events provide the "who, what, when" but not the "why" — combine with application logs and K8s audit for full context

### What Must Be Built Outside Tetragon

| Capability | External Tool Required |
|-----------|----------------------|
| Alert correlation & deduplication | SIEM (Splunk, Elastic, Chronicle) |
| Threat intelligence matching | SIEM + TI feed |
| Alert prioritization & scoring | SOAR / detection platform |
| Incident case management | ITSM / SOAR |
| Forensic artifact collection | Separate forensics tooling |
| Remediation workflows | SOAR + human response |
| Asset inventory | CMDB / K8s asset management |
| Vulnerability correlation | Vulnerability scanner integration |

---

## 9. Telemetry Health Checks

Quick reference checklist for ongoing telemetry health monitoring:

### Daily Checks

- [ ] `tetragon_missed_events_total` within acceptable threshold
- [ ] All TracingPolicy resources showing "enabled" status (`tetra tp list`)
- [ ] No new errors in Tetragon agent logs

### Weekly Checks

- [ ] Review `tetragon_event_cache_retries_total` trend
- [ ] Review `tetragon_enforcer_missed_notifications_total` (should be 0)
- [ ] Review per-policy action counters for unexpected spikes
- [ ] Validate that test detection events are triggering (canary processes)

### Monthly Checks

- [ ] Review BPF overhead metrics per policy — identify expensive rules
- [ ] Review BPF map memory metrics — identify policies with large memory footprints
- [ ] Audit TracingPolicy configurations for stale or unused rules
- [ ] Review Tetragon version — apply patch releases for security/reliability fixes

---

## 10. What to Build Around Tetragon

The following diagram shows Tetragon's position in an EDR architecture and what must be built:

```
┌──────────────────────────────────────────────────────────────────┐
│                        EDR Architecture                          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              TETRAGON (EDR Foundation Layer)             │    │
│  │  • Process telemetry (exec/exit/clone/credentials)       │    │
│  │  • File & integrity events (kprobe/LSM/IMA)              │    │
│  │  • Network telemetry (TCP/UDP/Unix socket)               │    │
│  │  • Detection policy engine (TracingPolicy)               │    │
│  │  • Enforcement (Signal/Override/Enforcer)                │    │
│  │  • Kubernetes context enrichment                         │    │
│  │  • gRPC/JSON event export                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         ↓ Events                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Detection    │  │ SIEM /       │  │ Incident Response    │  │
│  │ Logic Layer  │  │ Log Store    │  │ (SOAR / Manual)      │  │
│  │ (rules,      │  │ (Splunk,     │  │ (ticket, forensics,  │  │
│  │  scoring,    │  │  Elastic,    │  │  remediation)        │  │
│  │  correlation)│  │  Chronicle)  │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              External Enrichment Sources                  │   │
│  │  • Threat intelligence feeds                             │   │
│  │  • Vulnerability scanner                                 │   │
│  │  • Asset inventory / CMDB                               │   │
│  │  • Kubernetes audit logs                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 11. Kernel Version Planning

| Scenario | Recommended Kernel | Key Tetragon Features Unlocked |
|----------|-------------------|-------------------------------|
| Minimum viable | ≥4.19 | Core process/network/file telemetry |
| Recommended baseline | ≥5.10 | kprobe multi, improved selector performance |
| Security-hardened | ≥5.7 | LSM enforcement, IMA hashes |
| Full feature set | ≥5.11 | BPF ring buffer default, fentry, full BTF |
| Latest features | ≥6.x | Best cgroup support, latest BPF capabilities |

**Note:** Check `tetra probe config` on each node before deploying new features.

---

*Related: [EDR Capability Matrix](edr-capability-matrix.md) · [Operations, Performance & Hardening](modules/operations-performance-and-hardening.md) · [Enforcement & Response](modules/enforcement-and-response.md)*
