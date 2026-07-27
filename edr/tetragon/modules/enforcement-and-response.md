# Module: Enforcement & Response

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** Enforcement converts Tetragon from a pure detection tool into an active prevention and response platform. This module documents the evolution from the initial Signal/Override actions, through the killer sensor, to the production-grade Enforcer sensor with persistent enforcement, fmod_ret, security hooks, and uprobe overrides.

---

## Table of Contents

1. [Signal Action (SIGKILL)](#1-signal-action-sigkill)
2. [Override Action (Syscall Return Override)](#2-override-action-syscall-return-override)
3. [Killer Sensor Introduction](#3-killer-sensor-introduction)
4. [Enforcer Sensor (Killer Renamed)](#4-enforcer-sensor-killer-renamed)
5. [fmod_ret Support](#5-fmod_ret-support)
6. [security_* Hook Support](#6-security_-hook-support)
7. [Persistent Enforcement](#7-persistent-enforcement)
8. [NotifyEnforcer Action](#8-notifyenforcer-action)
9. [Uprobe Override Action](#9-uprobe-override-action)
10. [Action Telemetry & Stats](#10-action-telemetry--stats)
11. [Monitoring Mode for Safe Rollout](#11-monitoring-mode-for-safe-rollout)
12. [Kernel & Deployment Constraints](#12-kernel--deployment-constraints)
13. [Lifecycle: killer → enforcer Rename](#13-lifecycle-killer--enforcer-rename)
14. [EDR Design Notes](#14-edr-design-notes)

---

## 1. Signal Action (SIGKILL)

### Signal Action — **[NEW v0.8.0]**

The `Signal` action sends a Unix signal to the process that triggered the matched event. Typically used to send `SIGKILL` to terminate a process.

**Policy syntax:**
```yaml
actions:
  - action: Signal
    argSig: 9  # SIGKILL
```

**EDR use case:**
- Terminate processes exhibiting malicious behavior (e.g., process attempting container escape)
- Kill processes that match a specific binary + argument combination

**Limitations:**
- Signal is delivered asynchronously — the kernel operation that triggered the rule may have already completed before the process is killed.
- For synchronous blocking, use Override or the Enforcer.
- `SIGKILL` cannot be caught or ignored, but the process may have already written data or made network connections by the time it is killed.

---

## 2. Override Action (Syscall Return Override)

### Override Action — **[NEW v0.8.0]**

The `Override` action makes the hooked kernel function return a specific error code, effectively blocking the operation at the kernel level.

**Policy syntax:**
```yaml
actions:
  - action: Override
    argError: -1  # -EPERM
```

**EDR use cases:**
- Block specific `open()` / `write()` / `connect()` calls with `EPERM`
- Prevent execution of specific binaries by returning error from `execve`
- Block specific network connections

**How it works:** Uses kprobe override (requires `CONFIG_BPF_KPROBE_OVERRIDE=y` in the kernel) or fmod_ret (via the Enforcer) depending on the hook type.

**Synchronous:** Unlike Signal, Override returns the error before the operation completes — the syscall never executes.

---

## 3. Killer Sensor Introduction

### Killer Sensor — **[NEW v1.0.0]**

**Evidence:** [#1205](https://github.com/cilium/tetragon/pull/1205)

Tetragon v1.0.0 introduces the "killer sensor" as a dedicated enforcement subsystem. The killer sensor:
- Maintains a dedicated BPF program for process termination
- Receives notifications from other sensors via the `NotifyKiller` action
- Provides more reliable kill semantics than the simple `Signal` action

**Relationship to existing actions:** The killer sensor works alongside, not instead of, Signal and Override. It enables more complex enforcement scenarios where the "detect" hook and the "enforce" hook are different kernel functions.

---

## 4. Enforcer Sensor (Killer Renamed)

### Killer → Enforcer Rename — **[BREAK v1.1.0]**

**Evidence:** [#2117](https://github.com/cilium/tetragon/pull/2117)

In v1.1.0, the killer sensor was **renamed** to "enforcer sensor" and `NotifyKiller` action was renamed to `NotifyEnforcer`.

**Migration required (breaking change):**

| Old (v1.0.x) | New (v1.1.0+) |
|-------------|--------------|
| `killers:` in TracingPolicy | `enforcers:` |
| `action: NotifyKiller` | `action: NotifyEnforcer` |

**Why renamed:** The enforcer is not limited to killing processes — it can also override function return values, making "killer" a misleading name.

---

## 5. fmod_ret Support

### fmod_ret in Enforcer — **[NEW v1.1.0]**

**Evidence:** [#1953](https://github.com/cilium/tetragon/pull/1953)

`fmod_ret` (function return modification) is a BPF attachment type that allows a BPF program to modify the return value of a kernel function. The enforcer sensor uses `fmod_ret` to:
- Block operations by returning `EPERM` from security hook functions
- Avoid the need for `CONFIG_BPF_KPROBE_OVERRIDE=y` for enforcement on supported hooks
- Work on kernel ≥5.7 with `BPF_MODIFY_RETURN` support

**Advantage over kprobe override:** fmod_ret hooks are specifically designed for security control, while kprobe override is a more general (and potentially more brittle) mechanism.

---

## 6. security_* Hook Support

### `security_*` Functions in Enforcer — **[NEW v1.1.0]**

**Evidence:** [#2002](https://github.com/cilium/tetragon/pull/2002)

The enforcer can now attach to `security_*` LSM hooks (e.g., `security_file_open`, `security_socket_connect`, `security_bprm_check`). These hooks are:
- Called before the corresponding operation is executed
- Return-value-controllable (the enforcer returns `EPERM` to block)
- More semantically appropriate for security enforcement than raw kprobes

**Combined with fmod_ret:** The enforcer + fmod_ret + `security_*` hooks provide a proper LSM-level enforcement mechanism without requiring a full LSM module.

**Enforcer cleanup fix — [FIX v1.3.0]:** ([#3030](https://github.com/cilium/tetragon/pull/3030))

---

## 7. Persistent Enforcement

### Persistent Enforcement — **[NEW v1.2.0]**

**Evidence:** [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0)

By default, when Tetragon restarts, enforcement policies are also unloaded temporarily during restart. Persistent enforcement keeps the BPF enforcement programs pinned even during Tetragon restart cycles:
- BPF programs remain attached to kernel hooks during restart
- No enforcement gap during upgrade/restart
- Critical for production environments where any enforcement gap could be exploited

**Helm flag — [ENH v1.3.0]:** Dedicated `persistentEnforcement` Helm value added ([#2977](https://github.com/cilium/tetragon/pull/2977)) for explicit configuration.

**Kernel impact:** Persistent enforcement uses BPF program pinning. The pinned programs continue to enforce even when the Tetragon userspace process is not running.

---

## 8. NotifyEnforcer Action

### `NotifyEnforcer` Action — **[NEW v1.1.0]**

The `NotifyEnforcer` action is how other sensors (kprobe, tracepoint, LSM) communicate with the enforcer sensor. When a policy rule fires with `NotifyEnforcer`:
1. The detecting sensor generates the event
2. The enforcer is notified with the process context
3. The enforcer takes the configured enforcement action (kill/override)

**Missed notifications metric — [NEW v1.3.0]:**
`tetragon_enforcer_missed_notifications_total` ([#2994](https://github.com/cilium/tetragon/pull/2994)) — tracks notifications that the enforcer failed to receive (important for blind-spot detection).

**Validation — [ENH v1.6.0]:**
`NotifyEnforcer` action is now rejected at policy load time if no Enforcer sensor is configured ([#4008](https://github.com/cilium/tetragon/pull/4008)), preventing silent misconfiguration.

**Enforcer namespace support — [ENH v1.3.0]:**
Policy namespace is propagated to the enforcer sensor ([#3076](https://github.com/cilium/tetragon/pull/3076)).

---

## 9. Uprobe Override Action

### Uprobe Override — **[NEW v1.6.0]**

**Evidence:** [#4173](https://github.com/cilium/tetragon/pull/4173)

The Override action is now available for uprobe-based policies. This enables:
- Overriding user-space function return values
- Blocking specific user-space operations at the function level
- Prevention without kernel-level enforcement for user-space security controls

**Use cases:**
- Override `RAND_bytes()` in OpenSSL to detect or prevent crypto misuse
- Override authentication check functions to test security logic
- Simulate failures for application testing

---

## 10. Action Telemetry & Stats

### Action counters per policy — **[NEW v1.6.0]**

**Evidence:** [#4074](https://github.com/cilium/tetragon/pull/4074)

Tetragon v1.6.0 adds per-policy action counters, tracking how many times each enforcement action (Signal, Override, NotifyEnforcer) has been triggered per policy. Visible via:
- `tetragon_tracingpolicy_*` metrics
- `tetra tp list` command

**EDR value:** Enforcement telemetry is as important as detection telemetry. Action counters reveal:
- Whether enforcement policies are triggering
- Whether enforcement rates are unexpectedly high (policy misconfiguration)
- Enforcement trends over time

---

## 11. Monitoring Mode for Safe Rollout

### Monitoring Mode — **[NEW v1.4.0]**

**Evidence:** [#3393](https://github.com/cilium/tetragon/pull/3393)

See also [Detection Policy Engine](detection-policy-engine.md#11-monitoring-mode).

Setting a TracingPolicy to monitoring mode:
- All detection/event generation functions normally
- Enforcement actions (Signal, Override, NotifyEnforcer) are **silently suppressed**
- Events include context indicating the policy is in monitoring mode

**Recommended workflow:**
```
Phase 1: Deploy in monitoring mode → tune selectors → eliminate false positives
Phase 2: Enable enforcement on a staging workload
Phase 3: Roll out enforcement to production
```

---

## 12. Kernel & Deployment Constraints

| Feature | Constraint |
|---------|-----------|
| Signal action | Linux ≥4.9; no kernel config dependency |
| Override (kprobe) | `CONFIG_BPF_KPROBE_OVERRIDE=y` required |
| fmod_ret | Linux ≥5.7, `BPF_MODIFY_RETURN` support |
| security_* hooks | Linux ≥5.7, `CONFIG_BPF_LSM=y` |
| Persistent enforcement | BPF program pinning requires bpffs mount |
| Uprobe override | Linux ≥3.5 + uprobe override support |

**Deployment security note:** The enforcer sensor requires elevated BPF privileges. In Kubernetes, the Tetragon daemonset must have the appropriate seccomp/capability settings (typically `SYS_ADMIN` or specific BPF capabilities).

---

## 13. Lifecycle: killer → enforcer Rename

This is a documented breaking change that affected anyone upgrading from v1.0.x to v1.1.0:

```
v1.0.0:  killers:          + action: NotifyKiller
v1.1.0:  enforcers:        + action: NotifyEnforcer  ← rename required
v1.1.0+: persistent enforcement via fmod_ret + security hooks
v1.2.0:  persistent enforcement across restarts
v1.3.0:  dedicated Helm flag for persistent enforcement
v1.6.0:  validation: NotifyEnforcer rejected without Enforcer loaded
```

**Documentation also updated:** The `killer → enforcer` rename in documentation was tracked in [#2887](https://github.com/cilium/tetragon/pull/2887).

---

## 14. EDR Design Notes

### When to Use Signal vs. Override vs. Enforcer

| Scenario | Recommended Action |
|----------|-------------------|
| Terminate suspicious process (async) | `Signal` (SIGKILL) |
| Block a specific syscall synchronously | `Override` (return EPERM) |
| Enforce at LSM hook level | `NotifyEnforcer` + `security_*` hook + fmod_ret |
| Prevent execution of a specific binary | `Override` on `execve` / `security_bprm_check` |
| Kill process on capability misuse | `Signal` or `NotifyEnforcer` |

### Response Timing Caution

- `Signal` is **asynchronous** — the operation may have completed before the process receives SIGKILL
- `Override` is **synchronous** — the syscall never completes (it returns an error)
- `NotifyEnforcer` + fmod_ret is **synchronous** — the security hook returns EPERM before the operation

For security-critical prevention (e.g., blocking container escape), prefer synchronous methods (Override, fmod_ret) over Signal.

### Blast Radius Considerations

Before enabling enforcement at scale:
1. Test policies in monitoring mode (v1.4.0+) to detect false positives
2. Roll out to non-critical workloads first
3. Verify action counters are within expected ranges
4. Have a rollback plan (disable policy, restore from backup)
5. Tetragon does **not** provide automated rollback of files or network state — human response is required for complex incidents

---

*Related: [Detection Policy Engine](detection-policy-engine.md) · [Kernel & Userspace Instrumentation](kernel-and-userspace-instrumentation.md) · [Operations, Performance & Hardening](operations-performance-and-hardening.md)*
