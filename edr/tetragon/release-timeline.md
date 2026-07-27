# Tetragon Stable Release Timeline

> **Navigation:** [README](README.md) | [Matrix](edr-capability-matrix.md) | [Adoption Guide](edr-adoption-guide.md) | [Sources](sources.md)
>
> Compact release-by-release delta from v0.8.0 through v1.7.0 (the latest stable release as of July 27, 2026).
>
> **Legend:** 🆕 Feature | ⬆️ Enhancement | 🔧 Fix/Backport | 💥 Breaking Change | ⚠️ Deprecation | 🗑️ Removal | 🔐 Security | 🧪 Alpha/Experimental

---

## v0.8.0 — Initial Stable Release

**Release:** [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0)

Foundation release establishing the core Tetragon architecture.

| Type | Item | Module |
|------|------|--------|
| 🆕 | Process exec/exit/clone events | Process & Lineage |
| 🆕 | kprobe / kretprobe instrumentation | Kernel Instrumentation |
| 🆕 | Tracepoints | Kernel Instrumentation |
| 🆕 | Signal action (SIGKILL) | Enforcement |
| 🆕 | Override action (syscall return) | Enforcement |
| 🆕 | TracingPolicy CRD | Policy Engine |
| 🆕 | matchBinaries, matchArgs selectors | Policy Engine |
| 🆕 | Socket/SKB argument types | Network |
| 🆕 | TCP connect/accept events | Network |
| 🆕 | File/path telemetry via kprobes | File & Integrity |
| 🆕 | Field filters | Export |
| 🆕 | gRPC GetEvents API | Export |
| 🆕 | JSON file export | Export |
| 🆕 | Kubernetes pod/namespace enrichment | K8s Context |
| 🆕 | BTF/CO-RE support | Kernel Instrumentation |

---

## v0.8.1 — Patch

**Release:** [v0.8.1](https://github.com/cilium/tetragon/releases/tag/v0.8.1)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Process tracking and exec event reliability fixes | Process & Lineage |
| 🔧 | Policy loading stability fixes | Policy Engine |

---

## v0.8.2 — Patch

**Release:** [v0.8.2](https://github.com/cilium/tetragon/releases/tag/v0.8.2)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Reliability fixes for process events | Process & Lineage |

---

## v0.8.3 — Patch

**Release:** [v0.8.3](https://github.com/cilium/tetragon/releases/tag/v0.8.3)

| Type | Item | Module |
|------|------|--------|
| ⬆️ | kprobe multi (multi-link) support | Kernel Instrumentation |
| 🔧 | BPF program loading fixes | Operations |

---

## v0.8.4 — Patch

**Release:** [v0.8.4](https://github.com/cilium/tetragon/releases/tag/v0.8.4)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Process and BPF stability fixes | Process & Lineage |

---

## v0.9.0 — Minor Release

**Release:** [v0.9.0](https://github.com/cilium/tetragon/releases/tag/v0.9.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | Generic uprobe sensor (user-space function instrumentation) | Kernel Instrumentation |
| 🆕 | `--rb-size` / `--rb-size-total` perf ring buffer size options | Operations |
| 🆕 | Pod label filters (initial policyfilter) | Policy Engine / K8s Context |
| 🆕 | `matchNamespaces` selector | Policy Engine |
| 🆕 | bugtool initial diagnostics tooling | Operations |
| ⬆️ | matchBinaries filter moved earlier in processing pipeline | Policy Engine |
| 🔧 | BPF program loading improvements | Kernel Instrumentation |

---

## v0.10.0 — Minor Release

**Release:** [v0.10.0](https://github.com/cilium/tetragon/releases/tag/v0.10.0)

| Type | Item | Module |
|------|------|--------|
| ⬆️ | Socket tracking and richer socket/SKB handling | Network |
| ⬆️ | Pod/workload metadata enrichment improvements | K8s Context |
| ⬆️ | LT/GT operator support for matchReturnArgs | Policy Engine |
| ⬆️ | gRPC Unix socket transparent handling in tetra CLI | Export |
| 🔧 | Cgroup detection and association improvements | K8s Context |
| 🆕 | ProcessCredentials object; credential change tracking at syscall level | Process & Lineage |

---

## v0.11.0 — Minor Release

**Release:** [v0.11.0](https://github.com/cilium/tetragon/releases/tag/v0.11.0)

> ⚠️ Upgrade note: `tracingpolicies*` CRDs need to be manually deleted — see [issue #1394](https://github.com/cilium/tetragon/issues/1394).

| Type | Item | Module |
|------|------|--------|
| 🆕 | `ProcessCredentials` object & capability usage recording | Process & Lineage |
| 🆕 | LSM + Tracing program loading support | Kernel Instrumentation |
| ⬆️ | Credential changes at syscalls monitoring | Process & Lineage |
| ⬆️ | matchBinaries filter performance improvements | Policy Engine |
| 💥 | TracingPolicy CRD breaking changes requiring CRD deletion | Policy Engine |

---

## v1.0.0 — Major Release

**Release:** [v1.0.0](https://github.com/cilium/tetragon/releases/tag/v1.0.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | UID/GID credentials in process_exec | Process & Lineage |
| 🆕 | Privileged execution detection | Process & Lineage |
| 🆕 | Kernel stack traces (**alpha**) | Kernel Instrumentation |
| 🆕 | Killer sensor (enforcement substrate) | Enforcement |
| 🆕 | `policy_name` field in kprobe/tracepoint/uprobe events | Policy Engine / Export |
| 🆕 | Namespaced TracingPolicy + pod label filters enabled by default | Policy Engine / K8s Context |
| 🆕 | Kernel module operation tracking | Process & Lineage |
| 🆕 | Rate limiting in TracingPolicy | Policy Engine |
| 🆕 | Policy lists (named symbol/string lists) | Policy Engine |
| 🆕 | Event type filter in `tetra getevents` | Export |
| 🆕 | IPv6 support in BPF rate limit | Network |
| 🆕 | PodInfo CRD | K8s Context |
| ⬆️ | arm64 tarball build | Operations |
| ⬆️ | String matching with hash lookups for performance | Policy Engine |
| ⬆️ | Buffer between perf reader and event processor | Operations |
| ⬆️ | Ring buffer size options with size suffixes | Operations |
| ⬆️ | `bpf: read the task real parent` — accurate parent tracking | Process & Lineage |
| 💥 | Export file permissions default changed to `0600` | Export / Security |

---

## v1.0.1 — Patch

**Release:** [v1.0.1](https://github.com/cilium/tetragon/releases/tag/v1.0.1)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Process exec cache improvements | Process & Lineage |
| 🔧 | Stability fixes | Operations |

---

## v1.0.2 — Patch

**Release:** [v1.0.2](https://github.com/cilium/tetragon/releases/tag/v1.0.2)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Process-related stability fixes | Process & Lineage |

---

## v1.0.3 — Patch

**Release:** [v1.0.3](https://github.com/cilium/tetragon/releases/tag/v1.0.3)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Stability and process tracking fixes | Process & Lineage |

---

## v1.1.0 — Minor Release

**Release:** [v1.1.0](https://github.com/cilium/tetragon/releases/tag/v1.1.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | User-space stack traces | Kernel Instrumentation |
| 🆕 | Anonymous binary detection (memfd/fileless execution) | Process & Lineage |
| 🆕 | Enforcer sensor (killer renamed; fmod_ret + `security_*` hooks) | Enforcement |
| 🆕 | fmod_ret support | Enforcement |
| 🆕 | `security_*` hook support in enforcer | Enforcement |
| 🆕 | OCI hook setup for container start tracking | K8s Context |
| 🆕 | Redaction filters (censor sensitive strings) | Export / Privacy |
| 🆕 | Capability export filter | Export |
| 🆕 | `containerSelector` for per-container policy scoping | Policy Engine |
| 🆕 | `rateLimitScope` per-process vs. global rate limiting | Policy Engine |
| 🆕 | Missed events metric per type | Operations |
| 🆕 | Policy status through metrics and `tetra` | Operations |
| 🆕 | File permission tracing | File & Integrity |
| 🆕 | Uprobe arguments support | Kernel Instrumentation |
| 🆕 | Multi-link uprobe (single BPF program, multiple symbols) | Kernel Instrumentation |
| ⬆️ | `matchBinaries` rework — exec path read at execve time | Policy Engine |
| ⬆️ | Prefix/NotPrefix operators for matchBinaries | Policy Engine |
| ⬆️ | String matching extended to 4096 characters | Policy Engine |
| ⬆️ | `matchCapabilities` / capability-change filters | Policy Engine |
| ⬆️ | LT/GT operators | Policy Engine |
| ⬆️ | 32-bit syscall matching on x86 | Kernel Instrumentation |
| ⬆️ | Binary privilege-raise detection | Process & Lineage |
| ⬆️ | Namespace label filtering in pod label selectors | Policy Engine |
| ⬆️ | Policy name filter (`--policy-names`) | Export |
| ⬆️ | Metrics initialization to 0 at startup | Operations |
| ⬆️ | Process cache size / capacity metrics | Operations |
| ⬆️ | pprof heap in bugtool | Operations |
| ⬆️ | Run Tetragon without CRD access | K8s Context |
| ⬆️ | Node name set to hostname in standalone mode | K8s Context |
| 💥 | Killer → Enforcer rename; `killers:` → `enforcers:`; `NotifyKiller` → `NotifyEnforcer` | Enforcement |
| 💥 | `stack_trace` field renamed to `kernel_stack_trace`; `stackTrace` → `kernelStackTrace` | Kernel Instrumentation |
| 💥 | `pod.labels` event field removed; use `pod.pod_labels` | Export |
| 💥 | `symbol` (uprobe spec) replaced with `symbols` (array) | Policy Engine |
| 🔧 | Multiple field filter segfault / regression fixes | Export |
| 🔧 | matchBinaries exe-read fix ([#1926](https://github.com/cilium/tetragon/pull/1926)) | Policy Engine |
| 🔧 | prepend_name BPF path function fixes | File & Integrity |
| 🔧 | Prefix/postfix matching for long strings fix | Policy Engine |
| 🔧 | Container FS scan fix for CRI-O | K8s Context |

---

## v1.1.2 — Patch

**Release:** [v1.1.2](https://github.com/cilium/tetragon/releases/tag/v1.1.2)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Process event regression fix | Process & Lineage |
| 🔧 | Stability fixes | Operations |

---

## v1.2.0 — Minor Release

**Release:** [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | LSM sensor | Kernel Instrumentation |
| 🆕 | Persistent enforcement (across Tetragon restarts) | Enforcement |
| 🆕 | NRI (Node Resource Interface) rthooks | K8s Context |
| 🆕 | Username in process events | Process & Lineage |
| ⬆️ | Improved cgroup-id-based pod association | K8s Context |
| ⬆️ | Memory footprint reduction for unused features | Operations |
| ⬆️ | gRPC liveness probe default in Helm | Operations |
| ⬆️ | `tetragonOperator.skipCRDCreation` deprecated; use `crds.installMethod=none` | K8s Context |
| ⬆️ | OciHookSetup deprecated in favor of NRI | K8s Context |
| 🔧 | Various process cache and pod association fixes | Process & Lineage / K8s Context |

---

## v1.2.1 — Patch

**Release:** [v1.2.1](https://github.com/cilium/tetragon/releases/tag/v1.2.1)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Process cache robustness fixes | Process & Lineage |
| 🔧 | Pod association reliability fixes | K8s Context |

---

## v1.3.0 — Minor Release

**Release:** [v1.3.0](https://github.com/cilium/tetragon/releases/tag/v1.3.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | IMA hashes in LSM events | File & Integrity |
| 🆕 | Nested cgroup pod association | K8s Context |
| 🆕 | `in_init_tree` flag in process events | Process & Lineage |
| 🆕 | CEL export filter (Common Expression Language) | Export |
| 🆕 | IP/CIDR helpers in CEL | Network / Export |
| 🆕 | `cluster_name` field in GetEventsResponse | Export |
| 🆕 | `container_id` + `in_init_tree` export filters | Export |
| 🆕 | Syscall ABI in `syscall64` type (`SyscallId` object with ABI) | Kernel Instrumentation |
| 🆕 | 32-bit ARM (aarch32) syscall support | Kernel Instrumentation |
| 🆕 | Regex filter for parent process arguments | Policy Engine |
| 🆕 | BPF overhead metrics | Operations |
| 🆕 | BPF map memory metrics per TracingPolicy | Operations |
| 🆕 | BPF error metrics | Operations |
| 🆕 | Process cache dump (`tetra dump processcache`) | Operations |
| 🆕 | Debug programs command (`tetra debug progs`) | Operations |
| 🆕 | Debug maps command (`tetra debug maps`) | Operations |
| 🆕 | Deleted pod cache | K8s Context |
| 🆕 | Enforcer missed notifications metric | Enforcement / Operations |
| 🆕 | cgtracker policyfilter support | K8s Context |
| 🆕 | Configurable event cache retry count/delay | Operations |
| 🆕 | All syscall64 operators including Mask | Policy Engine |
| 🆕 | Multiple symbol instances in kprobe spec | Kernel Instrumentation |
| 🆕 | matchBinary name + args combined matching | Policy Engine |
| ⬆️ | policyfilter promoted from BETA to GA | Policy Engine |
| ⬆️ | GC interval configurable for process cache | Operations |
| ⬆️ | BTF/kallsyms caches removed (memory optimization) | Operations |
| ⬆️ | BPF bpffs layout improvement | Operations |
| ⬆️ | LRU data cache metrics | Operations |
| 💥 | `syscall64` type output changed to `SyscallId` object; use `--enable-compatibility-syscall64-size-type` to revert (removed v1.4.0) | Policy Engine |
| 💥 | `tetragon_ratelimit_dropped_total` renamed to `tetragon_export_ratelimit_events_dropped_total` | Operations |
| 💥 | Export file permission rotation behavior changed | Export |
| 🔧 | Clone event cache retry handler fix | Process & Lineage |
| 🔧 | Fix matchBinary children tracking ([#3186](https://github.com/cilium/tetragon/pull/3186)) | Policy Engine |
| 🔧 | Fix process exit signal for core dump | Process & Lineage |
| 🔧 | cgroupv1/v2 handling improvements | K8s Context |
| 🔧 | Overhead metric fix for return probes | Operations |
| 🗑️ | `--expose-kernel-addresses` flag removed | Export / Security |
| 🗑️ | `--pprof-addr` flag removed | Operations |

---

## v1.4.0 — Minor Release

**Release:** [v1.4.0](https://github.com/cilium/tetragon/releases/tag/v1.4.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | Ancestors in process events | Process & Lineage |
| 🆕 | Monitoring mode in TracingPolicy (detect without enforce) | Policy Engine / Enforcement |
| 🆕 | Attribute resolution (BTF-based `resolve:`) in TracingPolicy | Kernel Instrumentation |
| 🆕 | Windows build (Part 1) | Kernel Instrumentation |
| 🆕 | Dentry type in generic sensor | File & Integrity |
| 🆕 | CEL filter in `tetra` CLI | Export |
| 🆕 | Multiple operator replicas (HA) | K8s Context / Operations |
| 🆕 | struct socket + struct sockaddr argument types | Network |
| 🆕 | `--reconnect` for `tetra getevents` | Export |
| 🆕 | `tetra policyfilter listpolicies` | Policy Engine |
| ⬆️ | Dentry path reading extended to 4096 bytes (was 256) | File & Integrity |
| ⬆️ | `get_current_task_btf` BPF helper | Kernel Instrumentation |
| ⬆️ | LSM resolve flag kernel-version check removed | Kernel Instrumentation |
| ⬆️ | Kernel ≥6.11 cgroupv1 config documentation | K8s Context |
| ⬆️ | CRI socket metrics (cgidmap) | K8s Context |
| ⬆️ | Relaxed deployment detection logic | K8s Context |
| 💥 | `FollowFD`, `UnfollowFD`, `CopyFD` actions deprecated (removed v1.5.0) | Policy Engine |
| 🗑️ | Deprecated sensor gRPC API removed ([#3437](https://github.com/cilium/tetragon/pull/3437)) | Policy Engine |
| 🗑️ | `tetragon_map_errors_total` replaced by `map_errors_update_total`/`map_errors_delete_total` | Operations |
| 🗑️ | `--enable-compatibility-syscall64-size-type` flag removed | Kernel Instrumentation |
| 🔧 | Path truncation fix — dentry read unified to 4096 bytes ([#3427](https://github.com/cilium/tetragon/pull/3427)) | File & Integrity |
| 🔧 | `in_init_tree` fix for processes before Tetragon start | Process & Lineage |
| 🔧 | matchPIDs using first PID only fix | Policy Engine |
| 🔧 | vfsmnt assignment fix | File & Integrity |
| 🔧 | nspid assignment fix | K8s Context |
| 🔧 | BPF complexity fix in selectors ([#3523](https://github.com/cilium/tetragon/pull/3523)) | Policy Engine |
| 🔧 | Various dentry resolution fixes | File & Integrity |

---

## v1.4.1 — Patch

**Release:** [v1.4.1](https://github.com/cilium/tetragon/releases/tag/v1.4.1)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Ancestor-related event correctness backports | Process & Lineage |
| 🔧 | Process event reliability fixes | Process & Lineage |

---

## v1.5.0 — Minor Release

**Release:** [v1.5.0](https://github.com/cilium/tetragon/releases/tag/v1.5.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | Raw tracepoints | Kernel Instrumentation |
| 🆕 | Uprobes with actions | Kernel Instrumentation |
| 🆕 | Windows process create/exit events (observer + sensor) | Kernel Instrumentation |
| 🆕 | Path offload support | File & Integrity |
| 🆕 | Pod annotations in events | K8s Context |
| 🆕 | Node labels | K8s Context |
| 🆕 | Privileged container flag (`container.privileged`) | K8s Context |
| 🆕 | `--enable-ancestors` unified flag replacing per-event-type ancestor flags | Process & Lineage |
| ⬆️ | Logging library migrated to Go `log/slog` | Operations |
| ⬆️ | Pod event source attribution fix (HUBBLE_NODE_NAME) | K8s Context |
| ⬆️ | PodInfo `container.privileged` field | K8s Context |
| ⬆️ | OciHookSetup Helm section removed | K8s Context |
| ⬆️ | Default metrics scrape interval changed to 60s | Operations |
| ⬆️ | v1beta1 CRD handling logic removed | K8s Context |
| ⬆️ | Fix `inInitTree` for processes before Tetragon start (v1.5.0 backport) | Process & Lineage |
| 💥 | `FollowFD`, `UnfollowFD`, `CopyFD` actions **removed** | Policy Engine |
| ⚠️ | Per-event-type ancestor flags (`--enable-process-*-ancestors`) deprecated (removed v1.6.0) | Process & Lineage |
| 🔧 | Argument order fix in resolve argument option | Kernel Instrumentation |
| 🔧 | Multiple inactive selector fix | Policy Engine |
| 🔧 | Load sensor failure with mixed rate-limited kprobes | Policy Engine |
| 🔧 | Path permissions fix | File & Integrity |

---

## v1.6.0 — Minor Release

**Release:** [v1.6.0](https://github.com/cilium/tetragon/releases/tag/v1.6.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | USDT (User-Space Defined Tracepoints) sensor | Kernel Instrumentation |
| 🆕 | USDT action support | Kernel Instrumentation |
| 🆕 | USDT set action | Kernel Instrumentation |
| 🆕 | USDT policy `resolve:` support | Kernel Instrumentation |
| 🆕 | Uprobe override action | Enforcement |
| 🆕 | BPF ring buffer default on kernel ≥5.11 | Operations |
| 🆕 | Pod UID field in events | K8s Context |
| 🆕 | Action counters per TracingPolicy | Enforcement / Operations |
| 🆕 | Container name regex filter in CLI | Export |
| 🆕 | BPF program argument type (`bpf_prog`) | Network |
| 🆕 | `tetra probe config` kernel compatibility check | Operations |
| 🆕 | Range filter for matchArgs | Policy Engine |
| 🆕 | K8s control plane for non-K8s deployment | K8s Context |
| ⬆️ | Operator non-root default (UID 65532) | K8s Context / Operations |
| ⬆️ | `NotifyEnforcer` rejected at load if no Enforcer present | Enforcement |
| ⬆️ | Reduced RBAC for non-K8s deployment | K8s Context |
| ⬆️ | Add retry support for ControllerManager | K8s Context |
| 🗑️ | Per-event-type ancestor flags removed | Process & Lineage |
| 🔧 | Long executable filename argument corruption fix | Process & Lineage |
| 🔧 | Robust process argument parsing | Process & Lineage |
| 🔧 | Selector off-by-one bounds check | Policy Engine |
| 🔧 | Empty matchBinaries correctly ignored | Policy Engine |
| 🔧 | Cgroup fsscan incorrect path fix | K8s Context |
| 🔧 | CRD standalone custom resource validation fix | Policy Engine |

---

## v1.6.1 — Patch

**Release:** [v1.6.1](https://github.com/cilium/tetragon/releases/tag/v1.6.1)

| Type | Item | Module |
|------|------|--------|
| 🔧 | Memory leak fix in process and event caches ([#4257](https://github.com/cilium/tetragon/pull/4257)) | Operations |
| 🔧 | kretprobe args merge helper fix | Kernel Instrumentation |
| 🔧 | `op_capabilities_gained` filter consistency fix | Policy Engine |
| 🔧 | Various backports | Multiple |

---

## v1.7.0 — Minor Release (Latest as of July 27, 2026)

**Release:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

| Type | Item | Module |
|------|------|--------|
| 🆕 | Environment variable retrieval in process events | Process & Lineage |
| 🆕 | `matchParentBinaries` selector | Policy Engine / Process & Lineage |
| 🆕 | CEL evaluation in BPF (in-kernel) | Policy Engine |
| 🆕 | fentry sensor (BPF trampoline entry hook) | Kernel Instrumentation |
| 🆕 | `hostSelector` for host-level process policies | Policy Engine / K8s Context |
| 🆕 | AF_UNIX socket path extraction | Network |
| 🆕 | File-type selectors (`FileType` / `NotFileType`) | File & Integrity |
| 🆕 | Event log service | Export |
| 🆕 | Policy tags | Policy Engine |
| 💥 | Default gRPC server address changed from `localhost:54321` to `/var/run/tetragon/tetragon.sock` | Export / Security |
| ⚠️ | Legacy stacktrace-tree format deprecated/removed; see [Stack Traces migration guide](https://tetragon.io/docs/concepts/tracing-policy/selectors/#stack-traces) | Kernel Instrumentation |

---

*See [sources.md](sources.md) for direct links to all release notes and referenced PRs.*
