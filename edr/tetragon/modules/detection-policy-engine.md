# Module: Detection & Policy Engine

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** The policy engine is Tetragon's "detection logic" layer — it defines what is instrumented, what conditions trigger events or actions, and how policies are scoped to specific workloads. A well-tuned policy engine is the difference between a useful EDR signal and an unmanageable alert flood.

---

## Table of Contents

1. [TracingPolicy Overview](#1-tracingpolicy-overview)
2. [matchBinaries & followChildren](#2-matchbinaries--followchildren)
3. [matchArgs Selectors](#3-matchargs-selectors)
4. [matchCapabilities & Capability Filters](#4-matchcapabilities--capability-filters)
5. [matchNamespaces](#5-matchnamespaces)
6. [Pod Label / Container / Host Selectors](#6-pod-label--container--host-selectors)
7. [matchParentBinaries](#7-matchparentbinaries)
8. [Policy Lists](#8-policy-lists)
9. [Rate Limiting](#9-rate-limiting)
10. [CEL Filters & In-BPF CEL Evaluation](#10-cel-filters--in-bpf-cel-evaluation)
11. [Monitoring Mode](#11-monitoring-mode)
12. [Policy Tagging](#12-policy-tagging)
13. [Policy Validation & Error Handling](#13-policy-validation--error-handling)
14. [Deprecated & Removed Selectors / Actions](#14-deprecated--removed-selectors--actions)
15. [EDR Design Notes](#15-edr-design-notes)

---

## 1. TracingPolicy Overview

### TracingPolicy CRD — **[NEW v0.8.0]**

`TracingPolicy` is a Kubernetes CRD (or standalone YAML for non-K8s deployments) that defines:

1. **What to instrument** — kprobes, tracepoints, LSM hooks, uprobes, USDT, fentry
2. **What conditions to filter on** — selectors (binary, argument, capability, namespace, pod, parent)
3. **What actions to take** — `Post` (emit event), `Signal`, `Override`, `NotifyEnforcer`

**Namespaced policies** — `TracingPolicyNamespaced` enables namespace-scoped policies. Enabled by default from v1.0.0 ([v1.0.0 upgrade note](https://github.com/cilium/tetragon/releases/tag/v1.0.0)).

**Policy name in events** — `policy_name` field added to kprobe, tracepoint, and uprobe events in v1.0.0 ([#1574](https://github.com/cilium/tetragon/pull/1574)).

**Standalone (non-K8s) use:** Tetragon can run without Kubernetes API access since v1.1.0 ([#1931](https://github.com/cilium/tetragon/pull/1931)).

---

## 2. matchBinaries & followChildren

### `matchBinaries` selector — **[NEW v0.8.0; major rework v1.1.0]**

Allows a policy to target specific executable binaries by path.

**Evolution:**

| Release | Change |
|---------|--------|
| v0.8.0 | Basic binary path matching |
| v1.1.0 | **Rework** of matchBinaries to read the exe at execve time ([#1731](https://github.com/cilium/tetragon/pull/1731), [#1926](https://github.com/cilium/tetragon/pull/1926)) |
| v1.1.0 | `Prefix` and `NotPrefix` operators ([#1732](https://github.com/cilium/tetragon/pull/1732)) |
| v1.3.0 | matchBinary with args (binary name matching with argument filtering) ([#3210](https://github.com/cilium/tetragon/pull/3210)) |
| v1.6.0 | Empty matchBinaries correctly ignored ([#4022](https://github.com/cilium/tetragon/pull/4022)) |

**`followChildren`** — when set, all child processes of the matched binary are also tracked.
- **[FIX v1.3.0]** — Fixed tracking of matchBinary children ([#3186](https://github.com/cilium/tetragon/pull/3186))

**`matchBinaries` moved to earlier execution — [ENH v0.10.0+]:**
The matchBinaries filter was moved earlier in the processing pipeline ([#833](https://github.com/cilium/tetragon/pull/833)) for better performance.

---

## 3. matchArgs Selectors

### `matchArgs` — **[NEW v0.8.0; progressively enhanced]**

`matchArgs` conditions filter events based on the values of kernel function arguments. Operators include:

| Operator | Available from |
|---------|---------------|
| `Equal`, `NotEqual` | v0.8.0 |
| `Prefix`, `NotPrefix`, `Postfix` | v0.8.0 / v1.1.0 |
| `Contains` (substring) | v0.8.0 |
| `GT` / `LT` (greater/less than) | v1.1.0 ([#1863](https://github.com/cilium/tetragon/pull/1863)) |
| `Mask` (bitmask) | v0.8.0 |
| Regex | v1.3.0 (parent args) |
| `Mask` for syscall64 | v1.3.0 ([#2948](https://github.com/cilium/tetragon/pull/2948)) |
| Range filter | v1.6.0 ([#4109](https://github.com/cilium/tetragon/pull/4109)) |

**String length extended to 4096 chars — [ENH v1.1.0]:** ([#2069](https://github.com/cilium/tetragon/pull/2069))

**Prefix/postfix matching fix for long strings — [FIX v1.1.0]:** ([#1806](https://github.com/cilium/tetragon/pull/1806))

**Hash lookup optimization for string/char_buf matches — [ENH v1.0.0]:** ([#1408](https://github.com/cilium/tetragon/pull/1408))

**BPF complexity fix in selectors — [FIX v1.4.0]:** ([#3523](https://github.com/cilium/tetragon/pull/3523))

**Multiple inactive selector fix — [FIX v1.5.0]:** ([#3947](https://github.com/cilium/tetragon/pull/3947))

**Off-by-one bounds check fix — [FIX v1.6.0]:** ([#4170](https://github.com/cilium/tetragon/pull/4170))

---

## 4. matchCapabilities & Capability Filters

### `matchCapabilities` — **[NEW v1.1.0]**

Filters events based on the capabilities held by the triggering process. Enables:
- Detecting processes with unexpected capabilities (e.g., `CAP_SYS_ADMIN`)
- Tracking capability acquisition events

**`op_capabilities_gained` consistency fix — [FIX v1.6.1]:** ([v1.6.1](https://github.com/cilium/tetragon/releases/tag/v1.6.1)) Fixed inconsistency in the `op_capabilities_gained` filter operator.

### Capability export filter — **[NEW v1.1.0]**

**Evidence:** [#2107](https://github.com/cilium/tetragon/pull/2107)

A capability-based export filter allows filtering the event stream to events involving specific capabilities. Useful for scoping dashboards to privilege-escalation relevant events.

---

## 5. matchNamespaces

### `matchNamespaces` — **[NEW v0.9.0]**

Filters events based on Linux namespace membership (PID namespace, user namespace, network namespace, mount namespace). Useful for:
- Policies that only apply to processes outside the host namespace
- Detecting namespace escape attempts (process that enters a different namespace)

---

## 6. Pod Label / Container / Host Selectors

### Pod label filters — **[NEW v0.9.0; default on v1.0.0]**

**Evidence:** [v0.9.0](https://github.com/cilium/tetragon/releases/tag/v0.9.0), [#1647](https://github.com/cilium/tetragon/pull/1647)

Policies can be scoped to pods matching specific Kubernetes labels, enabling:
- Per-workload detection rules
- Applying stricter rules to sensitive workloads (databases, payment systems)
- Multi-tenant policy isolation

**Namespace label filtering — [ENH v1.1.0]:** ([#1952](https://github.com/cilium/tetragon/pull/1952)) — Filter using `k8s:io.kubernetes.pod.namespace` in pod label selectors.

### Container name selector (`containerSelector`) — **[NEW v1.1.0]**

**Evidence:** [#2231](https://github.com/cilium/tetragon/pull/2231)

Target policies to specific container names within a pod.

### `hostSelector` — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Target policies to host-level processes (processes running directly on the node, not in containers).

**EDR value:** Critical for deploying different detection rules for host-level vs. container-level processes, especially on mixed Kubernetes nodes.

---

## 7. matchParentBinaries

### `matchParentBinaries` selector — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Matches events only when the parent process binary matches the specified pattern. Enables:
- Detecting `curl` or `wget` when spawned from `bash` (but not from a CI pipeline)
- Spawn-chain-based detection without post-event correlation in SIEM
- Complement to `matchBinaries` for bidirectional lineage checks

**Example:** Alert only when `/usr/bin/python3` is spawned by `/bin/bash` (interactive shell → scripting = LOLBin risk).

---

## 8. Policy Lists

### Named Lists — **[NEW v1.0.0]**

**Evidence:** [#1401](https://github.com/cilium/tetragon/pull/1401)

Lists allow defining reusable sets of values (binary paths, IP addresses, strings) that can be referenced across multiple selector rules. Reduces policy duplication.

**Syscall attribute in lists — [FIX v1.5.0]:** ([#3895](https://github.com/cilium/tetragon/pull/3895)) Fixed respecting the syscall attribute in lists.

---

## 9. Rate Limiting

### `rateLimit` — **[NEW v1.0.0]**

Limits the rate of events emitted per policy rule. Essential for high-frequency hooks (network, file) where the event rate would otherwise overwhelm the pipeline.

### `rateLimitScope` — **[NEW v1.1.0]**

**Evidence:** [#1962](https://github.com/cilium/tetragon/pull/1962)

Controls whether rate limiting applies per-process or globally per-policy. Options:
- `global` — rate limit is shared across all processes
- `process` — each process has its own rate limit counter (allows one event per process per interval)

**[FIX v1.5.0]:** Fixed issue where mixing rate-limited and non-rate-limited kprobes in the same sensor caused load failure ([#3903](https://github.com/cilium/tetragon/pull/3903)).

**`tetragon_export_ratelimit_events_dropped_total` metric — [RENAMED v1.3.0]:** (was `tetragon_ratelimit_dropped_total`)

---

## 10. CEL Filters & In-BPF CEL Evaluation

### CEL Export Filter — **[NEW v1.3.0]**

**Evidence:** [#3098](https://github.com/cilium/tetragon/pull/3098)

Common Expression Language (CEL) filter enables arbitrary expression-based event filtering at export time. Supports:
- Boolean operators
- String operations
- IP address helpers (v1.3.0, [#3211](https://github.com/cilium/tetragon/pull/3211))
- Field access on all event types

**CEL CLI filter — [ENH v1.4.0]:** ([#3124](https://github.com/cilium/tetragon/pull/3124)) CEL filter exposed in the `tetra` CLI `getevents` command.

### In-BPF CEL Evaluation — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Tetragon v1.7.0 introduces CEL evaluation **inside the BPF program** (kernel-side), rather than only at export time. This is a significant leap:

| Aspect | CEL Export Filter (v1.3.0) | In-BPF CEL (v1.7.0) |
|--------|---------------------------|---------------------|
| Evaluation point | Userspace, at export time | Kernel BPF program |
| Events dropped before userspace | No | Yes (reduces kernel→user overhead) |
| Can trigger enforcement | No | Yes (can gate actions) |
| Overhead | Userspace CPU | BPF verifier complexity |

**EDR value:** In-BPF CEL enables high-performance policy evaluation without sending every matching event to userspace first, reducing both latency and userspace CPU load.

---

## 11. Monitoring Mode

### Monitoring Mode in TracingPolicy — **[NEW v1.4.0]**

**Evidence:** [#3393](https://github.com/cilium/tetragon/pull/3393)

Policies can be set to "monitoring mode" — they generate events but do not execute enforcement actions (Override/Signal/NotifyEnforcer).

**EDR value for safe rollout:**
1. Deploy policy in monitoring mode
2. Observe events and validate detection logic
3. Check for false positives
4. Switch to enforcement mode

This is equivalent to an IDS/IPS "detection only" mode and is essential for production-safe policy deployment.

---

## 12. Policy Tagging

### Policy Tags — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Policies can be annotated with tags for categorization, routing, and filtering. Tags can be used to:
- Group policies by threat category (MITRE ATT&CK technique, compliance framework)
- Filter event streams by policy tag in the export pipeline
- Enable dashboard views grouped by tag

---

## 13. Policy Validation & Error Handling

### Policy validation hardening — **[Progressive from v0.8.0]**

Tetragon validates TracingPolicy specs at load time:
- **[FIX v1.0.0]** — Crash fix in kprobe validation ([#1551](https://github.com/cilium/tetragon/pull/1551))
- **[ENH v1.1.0]** — Improved error message when failing to load a TracingPolicy ([#2031](https://github.com/cilium/tetragon/pull/2031))
- **[FIX v1.3.0]** — ArgSelector field validation fixes ([#4143](https://github.com/cilium/tetragon/pull/4143))
- **[ENH v1.3.0]** — CRD short names and category added ([#3065](https://github.com/cilium/tetragon/pull/3065))
- **[FIX v1.6.0]** — `NotifyEnforcer` action rejected if no Enforcer sensor is loaded ([#4008](https://github.com/cilium/tetragon/pull/4008))

### Policy metrics — **[ENH v1.3.0]**

BPF map memory usage by tracing policy exposed via gRPC API and `tetragon_tracingpolicy_kernel_memory_bytes` metric ([#2984](https://github.com/cilium/tetragon/pull/2984)).

### policyfilter stability — **[BETA → GA in v1.3.0]**

The policyfilter subsystem (cgroup-based policy scoping) was marked beta through v1.2.0 and promoted to stable in v1.3.0 ([#3056](https://github.com/cilium/tetragon/pull/3056)).

---

## 14. Deprecated & Removed Selectors / Actions

| Item | Deprecated | Removed | Migration |
|------|-----------|---------|-----------|
| `FollowFD` action | v1.4.0 | v1.5.0 | No direct replacement |
| `UnfollowFD` action | v1.4.0 | v1.5.0 | No direct replacement |
| `CopyFD` action | v1.4.0 | v1.5.0 | No direct replacement |
| gRPC sensor API | v1.3.0 | v1.4.0 ([#3437](https://github.com/cilium/tetragon/pull/3437)) | Use TracingPolicy API |
| `--enable-compatibility-syscall64-size-type` flag | — | v1.4.0 | Update to use `SyscallId` type |
| `tetragonOperator.skipCRDCreation` | v1.1.0 | v1.3.0 | Use `crds.installMethod=none` |
| Legacy stacktrace-tree format | v1.7.0 | v1.7.0 | See [Stack Traces docs](https://tetragon.io/docs/concepts/tracing-policy/selectors/#stack-traces) |

---

## 15. EDR Design Notes

### Policy Hierarchy for EDR

A recommended policy layering approach:

```
Layer 1: Built-in sensors (always-on process exec/exit/network baseline)
Layer 2: Organization-wide TracingPolicies (applied to all pods)
Layer 3: Workload-specific TracingPolicies (applied via pod label selectors)
Layer 4: Incident-response TracingPolicies (time-limited deep-dive rules)
```

### Selector Composition

Multiple selectors within a single rule are **AND-composed** (all must match). Multiple rules in a policy are **OR-composed** (any can match). This allows:
- Narrowly scoped rules: `matchBinaries[python3] AND matchArgs[suspicious_path]`
- Broad catch-all rules: multiple separate rules in one policy

### Noise Reduction Checklist

1. Use `matchBinaries` to scope rules to relevant binaries
2. Use `matchArgs` to narrow on specific paths, addresses, or patterns
3. Use `rateLimit` with appropriate `rateLimitScope`
4. Use `containerSelector` / pod label filters for workload scoping
5. Use `hostSelector` to separate host vs. container policies
6. Use monitoring mode before enforcement

---

*Related: [Process & Lineage](process-and-lineage.md) · [Enforcement & Response](enforcement-and-response.md) · [Kubernetes & Container Context](kubernetes-and-container-context.md) · [Export, Integration & Privacy](export-integration-and-privacy.md)*
