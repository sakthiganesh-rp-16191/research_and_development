# Module: Operations, Performance & Hardening

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** Production EDR deployments require low overhead, predictable resource usage, observable health metrics, and safe deployment patterns. This module documents Tetragon's performance mechanisms, observability/diagnostics tooling, memory management, and deployment hardening.

---

## Table of Contents

1. [Ring Buffer & Perf Buffer](#1-ring-buffer--perf-buffer)
2. [BPF Ring Buffer Default (kernel ≥5.11)](#2-bpf-ring-buffer-default-kernel-511)
3. [Event Processing Queue](#3-event-processing-queue)
4. [Process & Event Cache](#4-process--event-cache)
5. [Missed Event Metrics](#5-missed-event-metrics)
6. [BPF Overhead Metrics](#6-bpf-overhead-metrics)
7. [BPF Map Memory Metrics](#7-bpf-map-memory-metrics)
8. [Policy & Action Metrics](#8-policy--action-metrics)
9. [Metrics Observability Infrastructure](#9-metrics-observability-infrastructure)
10. [Memory Footprint Optimization](#10-memory-footprint-optimization)
11. [BPF Error Metrics](#11-bpf-error-metrics)
12. [Diagnostics & Bugtool](#12-diagnostics--bugtool)
13. [Deployment Hardening](#13-deployment-hardening)
14. [Operator HA & Non-Root](#14-operator-ha--non-root)
15. [EDR Production Readiness Checklist](#15-edr-production-readiness-checklist)

---

## 1. Ring Buffer & Perf Buffer

### Perf ring buffer tuning — **[NEW v0.9.0]**

**Evidence:** [#480](https://github.com/cilium/tetragon/pull/480)

Tetragon v0.9.0 adds `--rb-size` and `--rb-size-total` options to configure the perf ring buffer size, enabling:
- Higher-throughput nodes to use larger buffers to reduce event drops
- Memory-constrained environments to use smaller buffers

**Buffer between perf reader and event processor — [ENH v1.0.0]:** ([#593](https://github.com/cilium/tetragon/pull/593)) A decoupling buffer between the BPF ring buffer reader and the event processing pipeline prevents back-pressure from slow processing from causing BPF-side drops.

**Size suffix support — [ENH v1.0.0]:** ([#1593](https://github.com/cilium/tetragon/pull/1593)) Ring buffer size options now accept size suffixes (e.g., `10MB`, `1GB`).

---

## 2. BPF Ring Buffer Default (kernel ≥5.11)

### BPF ring buffer default — **[NEW v1.6.0]**

**Evidence:** [#4075](https://github.com/cilium/tetragon/pull/4075)

Tetragon v1.6.0 adopts the BPF ring buffer as the default event delivery mechanism for kernels ≥5.11, replacing the older perf event buffer.

**Advantages of BPF ring buffer over perf buffer:**
- Simpler single-producer single-consumer semantics
- Lower overhead for high-frequency events
- Better memory efficiency (contiguous allocation)
- More reliable ordering guarantees

**Fallback:** On kernels <5.11, the perf event buffer is used automatically.

---

## 3. Event Processing Queue

### Configurable event cache retry — **[ENH v1.3.0]**

**Evidence:** [#2928](https://github.com/cilium/tetragon/pull/2928)

The event cache (which holds events waiting for pod metadata enrichment) now has tunable:
- Number of retries for enrichment
- Delay between retries

This reduces the risk of events being emitted without pod metadata in environments where the K8s watcher is slow to populate.

### EventCache retries metric — **[NEW v1.1.0]**

`tetragon_event_cache_retries_total` and `tetragon_event_cache_parent_info_errors_total` metrics added ([#1923](https://github.com/cilium/tetragon/pull/1923)).

---

## 4. Process & Event Cache

### Process cache — **[Core from v0.8.0; progressively hardened]**

Tetragon maintains an in-memory LRU cache of process state (execve map) to enrich events with process context. Key improvements:

| Release | Change |
|---------|--------|
| v1.0.0 | TID/PID handling improvements ([#1256](https://github.com/cilium/tetragon/pull/1256)) |
| v1.1.0 | Process cache size metric (`tetragon_process_cache_size`) |
| v1.1.0 | Process cache capacity metric (`tetragon_process_cache_capacity`) |
| v1.2.0 | Memory reduction for unused features |
| v1.3.0 | Process cache dump capability (`tetra dump processcache`) ([#2246](https://github.com/cilium/tetragon/pull/2246)) |
| v1.3.0 | GC interval configurable ([#3130](https://github.com/cilium/tetragon/pull/3130)) |
| v1.4.0 | `GetExecveEntries` function ([#3390](https://github.com/cilium/tetragon/pull/3390)) |
| v1.6.1 | Memory leak fix in process and event caches ([#4257](https://github.com/cilium/tetragon/pull/4257)) |

### Deleted pod cache — **[NEW v1.3.0]**

**Evidence:** [#2930](https://github.com/cilium/tetragon/pull/2930)

A cache of recently deleted pods ensures that events arriving after pod deletion still carry pod metadata (rather than being emitted without context).

### kthreads during /proc scan — **[FIX v1.1.0]**

**Evidence:** [#2089](https://github.com/cilium/tetragon/pull/2089)

Kernel threads are now correctly parsed during the `/proc` scan at startup, preventing false "missing parent" events for kernel thread-spawned processes.

---

## 5. Missed Event Metrics

### Missed events per type metric — **[NEW v1.1.0]**

**Evidence:** [#1674](https://github.com/cilium/tetragon/pull/1674)

`tetragon_missed_events_total{type="..."}` tracks how many events were dropped (not delivered to userspace) per event type. Key for:
- Detecting ring buffer overflow (too many events, insufficient buffer size)
- Identifying which event types are most affected by drops
- Tuning ring buffer size and rate limits

**Enforcer missed notifications metric — [NEW v1.3.0]:**
`tetragon_enforcer_missed_notifications_total` ([#2994](https://github.com/cilium/tetragon/pull/2994)) — tracks enforcement-relevant events that the enforcer failed to process (critical for enforcement reliability monitoring).

---

## 6. BPF Overhead Metrics

### BPF overhead metrics — **[NEW v1.3.0]**

**Evidence:** [#3040](https://github.com/cilium/tetragon/pull/3040)

Per-BPF-program execution overhead metrics track the CPU time consumed by each eBPF program:
- Identifies which TracingPolicy rules are most CPU-intensive
- Enables cost-benefit analysis of individual detection rules
- Aggregated in userspace before reporting to reduce metric cardinality ([#3217](https://github.com/cilium/tetragon/pull/3217))

**Fix for return probes — [FIX v1.3.0]:** ([#3074](https://github.com/cilium/tetragon/pull/3074)) Overhead metrics for kretprobe programs were not correctly reported; fixed in v1.3.0.

---

## 7. BPF Map Memory Metrics

### BPF map memory usage metric — **[NEW v1.3.0]**

**Evidence:** [#2984](https://github.com/cilium/tetragon/pull/2984)

`tetragon_tracingpolicy_kernel_memory_bytes` tracks BPF map memory usage broken down by TracingPolicy. Visible via:
- `tetra tp list` command
- Prometheus metrics scrape

**Improvements in v1.1.0:**
- `tetragon_map_entries` (renamed from `tetragon_map_in_use_gauge`)
- `tetragon_map_capacity` — new metric for BPF map capacity
- Metrics collected directly from BPF maps at scrape time

**LRU data cache metrics — [NEW v1.3.0]:** ([#2908](https://github.com/cilium/tetragon/pull/2908))

---

## 8. Policy & Action Metrics

### Policy metrics — **[Progressive from v1.0.0]**

Policy status exposed through metrics and CLI ([#2090](https://github.com/cilium/tetragon/pull/2090)):
- Policy load state (enabled, disabled, errored)
- Policy sensor load status

**Action counters per policy — [NEW v1.6.0]:** ([#4074](https://github.com/cilium/tetragon/pull/4074)) Per-policy action counts (how many times each action fired per policy).

**Duplicate policy name metrics fix — [FIX v1.3.0]:** ([#3006](https://github.com/cilium/tetragon/pull/3006)) Fixed metrics collection when two policies have the same name.

**Rate limit events dropped metric — [RENAMED v1.3.0]:** `tetragon_export_ratelimit_events_dropped_total` (was `tetragon_ratelimit_dropped_total`).

**`map_errors_update_total` / `map_errors_delete_total` — [NEW v1.4.0]:** ([#3346](https://github.com/cilium/tetragon/pull/3346)) Replaced `tetragon_map_errors_total`.

---

## 9. Metrics Observability Infrastructure

### Metrics initialization — **[ENH v1.1.0]**

Metrics with known label values are initialized to 0 at startup ([#2162](https://github.com/cilium/tetragon/pull/2162)). This:
- Ensures stable resource usage (avoids sudden memory growth when new labels appear)
- Enables Prometheus queries that compare current vs. zero (not "no data")
- Enables alerting on first occurrence of a label combination

### Metrics label filter configuration — **[NEW v1.0.0]**

**Evidence:** [#1444](https://github.com/cilium/tetragon/pull/1444)

Configurable label filter for metrics — exclude high-cardinality labels from being reported.

### Build information in metrics — **[ENH v1.3.0]**

Version included in build info metric ([#3035](https://github.com/cilium/tetragon/pull/3035)).

### ServiceMonitor for Prometheus operator — **[ENH v1.0.0+]**

Helm chart includes ServiceMonitor resources for Tetragon agent and operator metrics. Default scrape interval changed from 10s to 60s in v1.5.0.

---

## 10. Memory Footprint Optimization

### Large memory footprint reductions — **[NEW v1.2.0]**

Significant memory reductions for the Tetragon agent when certain features are not used:
- Conditional feature allocation
- Reduced baseline idle memory

### BTF and kallsyms cache removal — **[ENH v1.3.0]**

**Evidence:** [#2937](https://github.com/cilium/tetragon/pull/2937)

BTF type data and kallsyms symbol caches were removed as persistent in-memory structures. Data is resolved on-demand instead, reducing baseline memory footprint significantly.

### Pod delete queue memory fix — **[FIX v1.1.0]**

Pod metrics queued for deletion are now removed after deletion ([#2287](https://github.com/cilium/tetragon/pull/2287)), preventing unbounded memory growth in high-churn environments.

### Memory leak fix in caches — **[FIX v1.6.1]**

**Evidence:** [#4257](https://github.com/cilium/tetragon/pull/4257) (backported from v1.6.0 development)

Memory leaks in the process and event caches fixed in v1.6.1.

---

## 11. BPF Error Metrics

### BPF error metrics — **[NEW v1.3.0]**

**Evidence:** [#3205](https://github.com/cilium/tetragon/pull/3205)

BPF map operation errors (failed reads/writes) now exposed as metrics. Helps detect:
- BPF map exhaustion (LRU eviction under load)
- Map update failures that could cause missed policy evaluations

---

## 12. Diagnostics & Bugtool

### bugtool — **[Available from v0.9.0]**

The `tetra bugtool` command collects diagnostic information for incident response and support, including:
- BPF map dumps
- Process cache dumps
- Agent configuration
- Kernel configuration checks

**Enhancements:**

| Release | Enhancement |
|---------|------------|
| v1.1.0 | pprof heap dump collection ([#2007](https://github.com/cilium/tetragon/pull/2007)) |
| v1.3.0 | Memory-related info ([#2880](https://github.com/cilium/tetragon/pull/2880)) |
| v1.3.0 | BPF map JSON dump added to bugtool ([#2963](https://github.com/cilium/tetragon/pull/2963)) |

### Debug programs command — **[NEW v1.3.0]**

**Evidence:** [#2967](https://github.com/cilium/tetragon/pull/2967)

`tetra debug progs` lists loaded BPF programs with their details, useful for:
- Verifying that expected policies are loaded
- Identifying BPF program load failures
- Checking BPF program metadata

### Debug maps command — **[NEW v1.3.0]**

**Evidence:** [#2959](https://github.com/cilium/tetragon/pull/2959)

`tetra debug maps` provides inspection of BPF maps — their contents, sizes, and usage.

### Process cache dump — **[NEW v1.3.0]**

`tetra dump processcache` ([#2246](https://github.com/cilium/tetragon/pull/2246)) — dumps the in-memory process execution cache for analysis.

### `tetra probe config` — **[NEW v1.6.0]**

**Evidence:** [#4020](https://github.com/cilium/tetragon/pull/4020)

Command to probe and check the kernel configuration for Tetragon feature compatibility. Useful for:
- Pre-deployment compatibility checks
- Validating kernel support for BPF ring buffer, LSM, fentry, etc.

---

## 13. Deployment Hardening

### Userspace / BPF struct alignment check — **[NEW v1.0.0]**

**Evidence:** [#1650](https://github.com/cilium/tetragon/pull/1650)

On startup, Tetragon checks for alignment mismatches between userspace and BPF struct definitions. If a mismatch is detected, the agent exits with a clear error message rather than silently producing incorrect events.

### BPF verification fixes — **[Progressive]**

Regular BPF verifier fixes to ensure programs pass kernel verification on various kernel versions and configurations:
- [FIX v1.3.0] BPF verifier fix for `bpf_execve_event` ([#1454](https://github.com/cilium/tetragon/pull/1454))
- [FIX v1.4.0] BPF complexity issues in selectors ([#3523](https://github.com/cilium/tetragon/pull/3523))
- [FIX v1.4.0] Verification fixes ([#3460](https://github.com/cilium/tetragon/pull/3460))

### Secure export file permissions — **[BREAK v1.0.0]**

Export JSON file now defaults to `0600` permissions ([#1575](https://github.com/cilium/tetragon/pull/1575)).

### Unix socket API (reduced attack surface) — **[BREAK v1.7.0]**

Default gRPC address changed from TCP loopback to Unix socket (see [export-integration-and-privacy.md](export-integration-and-privacy.md#10-unix-domain-socket-default)).

### `--expose-kernel-addresses` removed — **[REM v1.3.0]**

This flag that could leak kernel address layout information was permanently removed.

---

## 14. Operator HA & Non-Root

### Multiple operator replicas — **[NEW v1.4.0]**

See [kubernetes-and-container-context.md](kubernetes-and-container-context.md#13-operator-high-availability--hardening).

### Operator non-root default — **[NEW v1.6.0]**

Tetragon operator runs as UID 65532 by default ([#3909](https://github.com/cilium/tetragon/pull/3909)).

### Helm chart CRI configuration — **[ENH v1.4.0]**

`cri.enabled`, `cri.socketHostPath`, and `cgidmap.enabled` Helm variables added for explicit CRI socket configuration ([#3382](https://github.com/cilium/tetragon/pull/3382)).

---

## 15. EDR Production Readiness Checklist

Use this as a pre-deployment checklist for production EDR deployments:

### Performance
- [ ] Ring buffer size configured for expected event volume (`--rb-size`)
- [ ] BPF ring buffer used on ≥5.11 kernels (automatic from v1.6.0)
- [ ] Rate limits configured in TracingPolicy rules for high-frequency hooks
- [ ] `tetragon_missed_events_total` monitored and alerted on
- [ ] BPF overhead metrics reviewed per policy

### Reliability
- [ ] Memory leak patch applied (v1.6.1+)
- [ ] `tetragon_event_cache_retries_total` and `tetragon_event_cache_parent_info_errors_total` monitored
- [ ] Deleted pod cache enabled (automatic from v1.3.0)
- [ ] Event cache retry count and delay tuned for K8s API latency

### Security
- [ ] Export file permissions set to `0600` (default from v1.0.0)
- [ ] Agent API using Unix socket (default from v1.7.0)
- [ ] `--expose-kernel-addresses` absent (removed in v1.3.0)
- [ ] Operator running non-root (default from v1.6.0)
- [ ] Redaction filters configured for sensitive process arguments

### Observability
- [ ] Prometheus ServiceMonitor configured
- [ ] `tetragon_enforcer_missed_notifications_total` alerted on
- [ ] Per-policy action counters monitored (from v1.6.0)
- [ ] bugtool runbook documented for incident response

### Kernel Compatibility
- [ ] `tetra probe config` run pre-deployment to verify feature support
- [ ] Kernel version ≥5.7 for LSM enforcement
- [ ] BTF available for CO-RE (fentry, attribute resolution)
- [ ] cgroupv1/v2 configuration correct for kernel version

---

*Related: [Kernel & Userspace Instrumentation](kernel-and-userspace-instrumentation.md) · [Detection Policy Engine](detection-policy-engine.md) · [Kubernetes & Container Context](kubernetes-and-container-context.md)*
