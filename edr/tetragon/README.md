# Tetragon EDR Research Pack

> **Scope:** How the open-source [`cilium/tetragon`](https://github.com/cilium/tetragon) project can serve as a telemetry, detection, prevention, and response foundation for an endpoint detection and response (EDR) agent.
>
> **Cut-off date:** July 27, 2026 — latest stable release confirmed: **v1.7.0**

---

## Table of Contents

1. [Methodology](#methodology)
2. [Executive Summary](#executive-summary)
3. [File Index](#file-index)
4. [Terminology & Conventions](#terminology--conventions)
5. [Key Caveats](#key-caveats)

---

## Methodology

**Primary evidence source:** GitHub release notes at `https://github.com/cilium/tetragon/releases`.

**Release scope reviewed:**

| Series | Stable releases included |
|--------|--------------------------|
| v0.8.x | v0.8.0, v0.8.1, v0.8.2, v0.8.3, v0.8.4 |
| v0.9.x | v0.9.0 |
| v0.10.x | v0.10.0 |
| v0.11.x | v0.11.0 |
| v1.0.x | v1.0.0, v1.0.1, v1.0.2, v1.0.3 |
| v1.1.x | v1.1.0, v1.1.2 |
| v1.2.x | v1.2.0, v1.2.1 |
| v1.3.x | v1.3.0 |
| v1.4.x | v1.4.0, v1.4.1 |
| v1.5.x | v1.5.0 |
| v1.6.x | v1.6.0, v1.6.1 |
| v1.7.x | v1.7.0 |

Pre-release, RC, and `-pre` tags are **excluded** from the main feature inventory unless necessary to explain provenance. The legacy `tetragon-cli` tag is also excluded.

**Research approach:**
- Release notes and linked upstream PRs treated as primary evidence.
- Features are **not invented** — every claim traces to a release note entry.
- Features are distinguished as: new feature, material enhancement, reliability fix/backport, breaking change, deprecation, or removal.
- Maturity signals (alpha, experimental, deprecated, scheduled for removal, removed) are preserved as stated in release notes.
- Where a feature first appears in a minor/major release and is later fixed in a patch release, both facts are recorded.

---

## Executive Summary

Tetragon is an **eBPF-based security observability and runtime enforcement** project from the Cilium project. It is designed to instrument the Linux kernel at the syscall/function level, collect high-fidelity telemetry on process, file, network, and credential activity, and optionally enforce policy responses at the kernel level.

### What Tetragon provides (EDR perspective)

**Strong capabilities:**
- **Deep process telemetry** — exec, exit, fork/clone events with full path, arguments, credentials (UID/GID), username, capability transitions, privilege escalation detection, ancestor lineage trees, environment variables (v1.7.0+), and anonymous binary detection.
- **Detection policy engine** — `TracingPolicy` CRD with rich selector combinators (matchBinaries, matchArgs, matchCapabilities, matchNamespaces, matchParentBinaries, CEL, rate-limit), pod/namespace/container/host scope filtering.
- **Kernel instrumentation breadth** — kprobes, kretprobes, tracepoints, raw tracepoints, LSM, uprobes, USDT, fentry, with BTF-driven argument resolution.
- **Runtime enforcement** — Signal (SIGKILL), Override (syscall return override), and Enforcer/killer sensor (fmod_ret + security hooks), including persistent enforcement across restarts.
- **File & integrity** — path/dentry telemetry, file permission monitoring, IMA hash recording in LSM events, file-type selectors (v1.7.0).
- **Network & IPC** — socket/SKB argument types, TCP connect/accept/listen, CIDR/IP helpers in CEL, AF_UNIX socket path extraction (v1.7.0).
- **Kubernetes context** — pod/namespace/workload enrichment, container metadata, pod UID, privileged container flag, cgroup-based pod association (including nested cgroup support).
- **Export & privacy** — gRPC/JSON event stream, field filters, redaction filters (v1.1.0+), CEL export filters, in_init_tree/container_id export filters, Unix socket default (v1.7.0).
- **Observability/operations** — ring buffer tuning, BPF ring buffer default on ≥5.11 (v1.6.0+), action/policy/process-cache metrics, missed-event metrics, diagnostics (bugtool, debug progs).

**Limitations / not-built-in:**
- Tetragon is **not** a complete EDR product. It provides the telemetry and enforcement substrate but does not include: SIEM correlation, case management, asset inventory, quarantine/isolation of network interfaces (beyond kill/override), rollback of file changes, or remote remediation workflows.
- The **Windows support** introduced in v1.4.0–v1.5.0 is limited to process create/exit events via eBPF-equivalent mechanisms; full parity with Linux feature set is not yet claimed.
- Kernel-version constraints apply to several features — see individual module documents.

---

## File Index

| File | Description |
|------|-------------|
| [`README.md`](README.md) | This file — scope, methodology, executive summary, terminology |
| [`edr-capability-matrix.md`](edr-capability-matrix.md) | Master EDR capability table, organized by EDR module |
| [`release-timeline.md`](release-timeline.md) | Compact stable release-by-release delta from v0.8.0 through v1.7.0 |
| [`edr-adoption-guide.md`](edr-adoption-guide.md) | Practical phased adoption guide (observe → tune → detect → validate → enforce → integrate) |
| [`sources.md`](sources.md) | Stable-release links and important linked PRs |
| [`modules/process-and-lineage.md`](modules/process-and-lineage.md) | Process exec/exit/clone telemetry, credentials, ancestors, lineage |
| [`modules/file-and-integrity.md`](modules/file-and-integrity.md) | File/path telemetry, IMA hashes, file-type selectors |
| [`modules/network-and-ipc.md`](modules/network-and-ipc.md) | Socket/SKB telemetry, socket tracking, IP/CIDR, AF_UNIX |
| [`modules/kernel-and-userspace-instrumentation.md`](modules/kernel-and-userspace-instrumentation.md) | kprobes, tracepoints, LSM, uprobes, USDT, fentry, BTF, stack traces |
| [`modules/detection-policy-engine.md`](modules/detection-policy-engine.md) | TracingPolicy, selectors, matchBinaries, CEL, monitoring mode, rate limits |
| [`modules/enforcement-and-response.md`](modules/enforcement-and-response.md) | Override, Signal, killer/enforcer evolution, persistent enforcement |
| [`modules/kubernetes-and-container-context.md`](modules/kubernetes-and-container-context.md) | Pod/workload/cgroup enrichment, OCI hooks, NRI, policy filtering |
| [`modules/export-integration-and-privacy.md`](modules/export-integration-and-privacy.md) | gRPC/JSON export, field/CEL/redaction filters, Unix socket, SIEM |
| [`modules/operations-performance-and-hardening.md`](modules/operations-performance-and-hardening.md) | Ring buffers, metrics, memory, bugtool, operator HA/non-root |

---

## Terminology & Conventions

| Term | Meaning |
|------|---------|
| **TracingPolicy** | The Tetragon Kubernetes CRD (or standalone YAML) that defines what to instrument and what actions to take |
| **kprobe / kretprobe** | eBPF hook on kernel function entry / return |
| **tracepoint / raw tracepoint** | Stable kernel instrumentation point; raw tracepoints have lower overhead |
| **LSM** | Linux Security Module hook — used for security-enforcement hook points |
| **uprobe / uretprobe** | User-space function entry / return probe |
| **USDT** | User-Space Defined Tracepoints — statically defined probe points in user-space binaries |
| **fentry** | Fast eBPF entry hook using BPF trampolines (kernel ≥5.5 with BTF) |
| **BTF** | BPF Type Format — enables CO-RE (compile-once, run-everywhere) argument resolution |
| **Enforcer sensor** | The Tetragon enforcement subsystem (formerly called "killer sensor" until v1.1.0) |
| **NotifyEnforcer** | TracingPolicy action that triggers the enforcer to kill a process or override a syscall |
| **Override** | Action that makes a hooked kernel function return a specified error code |
| **Signal** | Action that sends a Unix signal (e.g., SIGKILL) to a process |
| **policyfilter** | Tetragon subsystem for scoping policies to specific pods/namespaces/containers |
| **in_init_tree** | Flag indicating whether a process is descended from its container's init process |
| **IMA** | Linux Integrity Measurement Architecture — used for file hash recording |
| **CEL** | Common Expression Language — used in export filters and (v1.7.0) in-BPF evaluation |
| **alpha / experimental** | Feature explicitly flagged as unstable; API may change |
| **deprecated** | Feature/API marked for removal in a future release |
| **removed** | Feature/API confirmed removed in the stated release |

### Classification conventions used in module documents

- **[NEW]** — Feature first introduced in this release
- **[ENH]** — Material enhancement to an existing feature
- **[FIX]** — Reliability or correctness fix (may be backported to patch releases)
- **[BREAK]** — Breaking change requiring migration
- **[DEP]** — Deprecated in this release
- **[REM]** — Removed in this release
- **[ALPHA]** — Marked alpha/experimental by upstream
- **[BETA]** — Marked beta by upstream (policyfilter was beta through v1.2, stable from v1.3)

---

## Key Caveats

1. **Tetragon is an EDR building block, not a complete EDR product.** It must be integrated with a SIEM, alerting pipeline, policy management workflow, and incident-response tooling to form a functional EDR program.

2. **Kernel version dependencies** are material. Many features require Linux ≥4.19, some require ≥5.5 (fentry, BTF), ≥5.10 (kprobe multi), ≥5.11 (BPF ring buffer), or ≥6.x (some LSM hooks). Always check module documents for per-feature constraints.

3. **eBPF verifier complexity** limits some long-path or multi-selector policies on older kernels. The BPF verifier may reject complex programs; Tetragon attempts feature detection at startup.

4. **Windows support** (v1.4.0–v1.5.0) is scoped to process create/exit events and does not replicate the full Linux eBPF feature set.

5. **Patch releases** (e.g., v1.0.1, v1.1.2) contain reliability and security backports. They should be consumed in production; features documented as "introduced in v1.x.0" may have critical fixes in patch releases.

6. **API lifecycle:** Several APIs have been deprecated and removed across the release history. See [`release-timeline.md`](release-timeline.md) for a full chronological record and [`modules/enforcement-and-response.md`](modules/enforcement-and-response.md) for the killer→enforcer rename lifecycle.

---

*Generated from upstream release notes as of July 27, 2026. Primary source: https://github.com/cilium/tetragon/releases*
