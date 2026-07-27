# Module: Kubernetes & Container Context

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** Container-context enrichment is what transforms Tetragon from a generic Linux security monitor into a Kubernetes-native EDR component. Every Tetragon event carries workload identity (pod, namespace, container) allowing security teams to attribute alerts to specific services, tenants, and deployment contexts.

---

## Table of Contents

1. [Pod & Namespace Enrichment](#1-pod--namespace-enrichment)
2. [Container Metadata](#2-container-metadata)
3. [Cgroup-Based Pod Association](#3-cgroup-based-pod-association)
4. [Nested Cgroup Support](#4-nested-cgroup-support)
5. [Pod UID Field](#5-pod-uid-field)
6. [Pod Annotations](#6-pod-annotations)
7. [Node Labels](#7-node-labels)
8. [Privileged Container Detection](#8-privileged-container-detection)
9. [OCI Hooks (rthooks)](#9-oci-hooks-rthooks)
10. [NRI (Node Resource Interface) Hooks](#10-nri-node-resource-interface-hooks)
11. [Policy Filtering by Workload](#11-policy-filtering-by-workload)
12. [Non-Kubernetes Deployment](#12-non-kubernetes-deployment)
13. [Operator High Availability & Hardening](#13-operator-high-availability--hardening)
14. [EDR Design Notes](#14-edr-design-notes)

---

## 1. Pod & Namespace Enrichment

### Kubernetes-aware event model — **[NEW v0.8.0]**

From its initial release, Tetragon was designed as a Kubernetes-native tool. All events carry:
- **Pod name** — the Kubernetes pod name
- **Pod namespace** — the Kubernetes namespace
- **Workload** — the owner (Deployment, DaemonSet, StatefulSet name)
- **Node name** — node where the event occurred

**Event structure example:**
```json
{
  "process": {
    "exec_id": "...",
    "binary": "/bin/bash",
    "pod": {
      "namespace": "production",
      "name": "web-frontend-abc123",
      "workload": "web-frontend",
      "node_name": "node-01"
    }
  }
}
```

**Progressive enrichment improvements:** Pod metadata enrichment quality improved across all releases as Kubernetes watcher stability, cgroup→pod association, and event cache handling matured.

---

## 2. Container Metadata

### Container name and image enrichment — **[NEW v0.8.0]**

Events include:
- Container name (within the pod)
- Container image reference

**Container ID export filter — [NEW v1.3.0]:** ([#3209](https://github.com/cilium/tetragon/pull/3209)) Filter event streams by container ID.

---

## 3. Cgroup-Based Pod Association

### Cgroup → Pod mapping — **[Core from v0.8.0; improved through v1.3.0]**

Tetragon uses cgroup IDs to associate kernel events (which only have a process context) with Kubernetes pods. The cgroup → pod association pipeline:
1. Kernel event carries the cgroup ID of the triggering process
2. Tetragon maintains a cgroup-ID → pod-info mapping
3. Events are enriched with pod metadata at userspace

**Cgroup handling improvements:**

| Release | Change |
|---------|--------|
| v1.2.0 | Improved cgroup-id-based pod association |
| v1.3.0 | Improved cgroupv1/cgroupv2 handling ([#3053](https://github.com/cilium/tetragon/pull/3053)) |
| v1.4.0 | Relaxed deployment detection logic ([#3400](https://github.com/cilium/tetragon/pull/3400)); CRI metrics added ([#3382](https://github.com/cilium/tetragon/pull/3382)) |
| v1.6.0 | Fix for incorrect cgroup path in cgroups fsscan ([#4117](https://github.com/cilium/tetragon/pull/4117)) |

**cgroupv1 on kernel ≥6.11:** Requires specific cgroupv1 configurations — documented in [v1.4.0](https://github.com/cilium/tetragon/pull/3284).

---

## 4. Nested Cgroup Support

### Nested Cgroup Pod Association — **[NEW v1.3.0]**

**Evidence:** [#3170](https://github.com/cilium/tetragon/pull/3170)

In environments where containers run inside VMs or use nested container runtimes (e.g., KataContainers, Firecracker, nested Docker), the cgroup hierarchy can be nested. v1.3.0 adds support for associating pod information when nested cgroups are used.

**EDR value:** Without nested cgroup support, events from processes in nested container environments would have incomplete or missing pod metadata, reducing the value of Kubernetes-context-enriched alerts.

**cgtracker policyfilter support — [ENH v1.3.0]:** ([#3180](https://github.com/cilium/tetragon/pull/3180)) The cgroup tracker now integrates with the policy filter system for scoped policy application in nested-cgroup environments.

---

## 5. Pod UID Field

### Pod UID in Events — **[NEW v1.6.0]**

**Evidence:** [#4069](https://github.com/cilium/tetragon/pull/4069)

The Kubernetes pod UID is now included in Tetragon events. The pod UID is:
- **Unique and immutable** per pod lifetime (unlike pod names which can be reused after recreation)
- Useful for correlating Tetragon events with Kubernetes audit logs
- Required for precise event deduplication across pod restarts

**EDR value:** Event correlation pipelines that join Tetragon events with Kubernetes audit logs (e.g., via SIEM) benefit from the stable pod UID as a join key.

---

## 6. Pod Annotations

### Pod Annotations in Events — **[NEW v1.5.0]**

**Evidence:** [#3527](https://github.com/cilium/tetragon/pull/3527)

Pod annotations are now included in process events. Annotations can carry:
- Security policy labels (`policy.kubernetes.io/compliance-level`)
- Team/owner annotations for alert routing
- Environment annotations (production, staging)
- Custom metadata for SIEM correlation

---

## 7. Node Labels

### Node Label Support — **[NEW v1.5.0]**

**Evidence:** [v1.5.0](https://github.com/cilium/tetragon/releases/tag/v1.5.0)

Node labels are included in events and can be used in policy selectors. This enables:
- Node-type-specific policies (e.g., different rules for GPU nodes, edge nodes)
- Topology-aware detection (e.g., stricter rules on nodes with external access)

---

## 8. Privileged Container Detection

### `container.privileged` Flag — **[NEW v1.5.0]**

**Evidence:** [#3661](https://github.com/cilium/tetragon/pull/3661)

Events now include a `container.privileged` boolean field indicating whether the container is running in privileged mode.

**EDR value:**
- High-confidence signal: any unexpected behavior from a privileged container is a critical alert
- Enables policy rules: `hostSelector: false AND container.privileged: true` as a risk-prioritization filter
- Supports compliance requirements around privileged container usage

---

## 9. OCI Hooks (rthooks)

### OCI hook setup — **[NEW v1.1.0]**

**Evidence:** [#1842](https://github.com/cilium/tetragon/pull/1842)

Tetragon v1.1.0 introduces OCI runtime hook setup (`tetragon-oci-hook`). The OCI hook fires at container creation, enabling:
- Pre-execution process context injection
- Container start tracking that precedes the first process exec event
- Integration with OCI-compliant runtimes (containerd, CRI-O, Docker)

**Lifecycle:**

| Release | Change |
|---------|--------|
| v1.1.0 | OCI hook setup introduced |
| v1.2.0 | `OciHookSetup` Helm section deprecated; NRI preferred |
| v1.5.0 | `OciHookSetup` Helm section **removed** |

**Current recommendation:** Use NRI hooks (v1.2.0+) instead of OCI hooks where possible.

---

## 10. NRI (Node Resource Interface) Hooks

### NRI rthooks — **[NEW v1.2.0]**

**Evidence:** [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0)

NRI (Node Resource Interface) is a container runtime plugin interface that allows Tetragon to receive container lifecycle events (create, update, delete) without requiring OCI runtime hook installation. NRI:
- Works with containerd ≥1.7 and CRI-O ≥1.26
- Does not require privileged hook binaries in the container spec
- Provides more reliable container-start tracking

**Fix — [FIX v1.4.0]:** rootDir resolution in NRI createRuntime hook ([#3466](https://github.com/cilium/tetragon/pull/3466)).

**NRI hook image:** Updated to v0.4 in v1.3.0 ([#3058](https://github.com/cilium/tetragon/pull/3058)).

---

## 11. Policy Filtering by Workload

### policyfilter (pod/namespace scope) — **[NEW v0.9.0; BETA → GA v1.3.0]**

The policyfilter subsystem allows TracingPolicies to be scoped to:
- Specific Kubernetes namespaces
- Specific pods (by label, name, or cgroup ID)
- Specific containers (by name)

**Beta → GA promotion in v1.3.0:** ([#3056](https://github.com/cilium/tetragon/pull/3056))

**Listpolicies command — [ENH v1.4.0]:** ([#3122](https://github.com/cilium/tetragon/pull/3122)) `tetra policyfilter listpolicies` command to inspect which policies are active for which pods.

**Container FS scan fix for CRI-O — [FIX v1.1.0]:** ([#2188](https://github.com/cilium/tetragon/pull/2188))

---

## 12. Non-Kubernetes Deployment

### Standalone (non-K8s) mode — **[Supported from v0.8.0]**

Tetragon runs without a Kubernetes API server in standalone mode. Improvements:
- **[ENH v1.1.0]** — Node name set to hostname in standalone mode ([#2123](https://github.com/cilium/tetragon/pull/2123))
- **[ENH v1.1.0]** — Run Tetragon without CRD access ([#1931](https://github.com/cilium/tetragon/pull/1931))
- **[ENH v1.3.0]** — Custom K8s object validation on standalone ([#1521](https://github.com/cilium/tetragon/pull/1521))
- **[ENH v1.6.0]** — K8s control plane enabled for non-K8s deployment mode ([#4011](https://github.com/cilium/tetragon/pull/4011))
- **[ENH v1.6.0]** — Reduced RBAC permissions for non-K8s deployment ([#4060](https://github.com/cilium/tetragon/pull/4060))

**EDR value:** Enables Tetragon deployment on bare-metal hosts, VMs, and edge nodes without Kubernetes.

---

## 13. Operator High Availability & Hardening

### Multiple operator replicas (HA) — **[NEW v1.4.0]**

**Evidence:** [#3443](https://github.com/cilium/tetragon/pull/3443)

Multiple Tetragon operator replicas can run simultaneously with leader election (`failoverLease.enabled=true`). Configuration:
- `tetragonOperator.replicas=2`
- `tetragonOperator.failoverLease.enabled=true`
- Default rolling update strategy to reduce downtime
- Default pod anti-affinity for distribution across nodes

### Operator non-root default — **[NEW v1.6.0]**

**Evidence:** [#3909](https://github.com/cilium/tetragon/pull/3909)

The Tetragon operator now defaults to running as non-root (UID 65532). Override with `tetragonOperator.runAsRoot: true` if needed.

**EDR value:** Reduces the blast radius of operator compromise.

### PodInfo Custom Resource — **[NEW v1.0.0]**

**Evidence:** [#1410](https://github.com/cilium/tetragon/pull/1410)

The `PodInfo` CRD provides a Kubernetes-native resource representing real-time pod metadata for Tetragon enrichment.

---

## 14. EDR Design Notes

### Event Attribution Pipeline

```
Kernel Event (cgroup_id)
      ↓
cgroup → pod mapping (in-memory map, updated via K8s watcher + CRI hooks)
      ↓
Event enrichment (pod/namespace/container/node metadata)
      ↓
Policy filter (restrict to matching pods/namespaces)
      ↓
Export (gRPC / JSON with full K8s context)
```

### Gaps & Reliability Considerations

1. **Short-lived pods** — very short pod lifetimes (milliseconds) may result in events with incomplete pod metadata if the K8s watcher hasn't yet processed the pod creation event.
2. **Deleted pod cache** — added in v1.3.0 ([#2930](https://github.com/cilium/tetragon/pull/2930)) to handle events arriving after pod deletion.
3. **CRI socket dependency** — cgroup→pod mapping relies on CRI socket access. CRI socket misconfiguration can result in events without pod context.
4. **Nested namespaces** — processes in deeply nested namespace hierarchies (cgroups, pid namespaces) benefit from v1.3.0's nested cgroup support.

### Multi-tenant Cluster Considerations

For multi-tenant clusters:
1. Use `TracingPolicyNamespaced` to confine policies to specific namespaces
2. Use `policyfilter` (now GA) to prevent cross-tenant policy interference
3. Export `cluster_name` (v1.3.0+) in events for multi-cluster SIEM correlation
4. Use pod UID (v1.6.0+) as the stable join key across Tetragon and audit logs

---

*Related: [Detection Policy Engine](detection-policy-engine.md) · [Export, Integration & Privacy](export-integration-and-privacy.md) · [Operations, Performance & Hardening](operations-performance-and-hardening.md)*
