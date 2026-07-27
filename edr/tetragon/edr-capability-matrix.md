# EDR Capability Matrix — Tetragon

> **Navigation:** [README](README.md) | [Release Timeline](release-timeline.md) | [Adoption Guide](edr-adoption-guide.md) | [Sources](sources.md)
>
> **Purpose:** Master table mapping Tetragon capabilities to EDR functions. Use module documents for detail; use this table for quick cross-cutting reference.
>
> Legend — **Type:** D = Detection/Visibility, P = Prevention/Response, D+P = Both
> Legend — **Maturity:** GA = Generally Available, ALPHA = Alpha/Experimental, DEP = Deprecated, REM = Removed

---

## 1. Process Telemetry & Lineage

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| Process exec events (`process_exec`) | Process execution tracking, LOLBin detection | v0.8.0 | Improved path handling v1.0–v1.4; argument corruption fix v1.6.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Process exit events (`process_exit`) | Process termination tracking | v0.8.0 | Exit signal/core-dump fix v1.3.0; exit-probe hook fix v1.0.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Clone/fork events | Process lineage tracking | v0.8.0 | Clone event cache retry fix v1.3.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| UID/GID credentials in process_exec | Privileged execution detection, credential abuse | v1.0.0 | — | D | GA | [v1.0.0 #1296](https://github.com/cilium/tetragon/pull/1296) |
| Privileged execution detection | Setuid/capability escalation detection | v1.0.0 | Privilege-raise detection enhancement v1.1.0 | D | GA | [v1.0.0 #1296](https://github.com/cilium/tetragon/pull/1296) |
| Username in process events | Human-readable identity in alerts | v1.2.0 | — | D | GA | [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0) |
| Ancestor process tree in events | Full process lineage for triage | v1.4.0 | Ancestor flags consolidation v1.5.0; flag deprecation + removal v1.5.0/v1.6.0 | D | GA; per-event-type ancestor flags deprecated v1.5 | [v1.4.0 #2938](https://github.com/cilium/tetragon/pull/2938) |
| Anonymous binary detection | Fileless malware / memfd execution | v1.1.0 | — | D | GA | [v1.1.0 #499](https://github.com/cilium/tetragon/pull/499) |
| Binary privilege-raise detection | SUID/capability abuse | v1.1.0 | — | D | GA | [v1.1.0 #1786](https://github.com/cilium/tetragon/pull/1786) |
| Environment variable retrieval | Credential theft detection, LOLBin context | v1.7.0 | — | D | GA (requires policy opt-in) | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| `in_init_tree` flag | Container escape / init-tree isolation | v1.3.0 | Fix for processes before Tetragon start v1.4.0; backport fix v1.5.0 | D | GA | [v1.3.0 #3209](https://github.com/cilium/tetragon/pull/3209) |
| Kernel module operation tracking | Rootkit detection (module load) | v1.0.0 | — | D | GA | [v1.0.0 #1390](https://github.com/cilium/tetragon/pull/1390) |
| `ProcessCredentials` object & capability usage tracking | Credential change detection | v0.11.0 | — | D | GA | [v0.11.0 #888](https://github.com/cilium/tetragon/pull/888) |

---

## 2. File & Integrity Monitoring

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| File/path event instrumentation via kprobes | File access/modification monitoring | v0.8.0 | Path resolution improvements v1.0–v1.4 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| File permission tracing in policies | Permission change monitoring | v1.1.0 | — | D | GA | [v1.1.0](https://github.com/cilium/tetragon/releases/tag/v1.1.0) |
| Path truncation fix (4096 bytes) | Accurate long-path telemetry | v1.4.0 | — | D | GA; replaces 256-char limit | [v1.4.0 #3427](https://github.com/cilium/tetragon/pull/3427) |
| IMA hashes in LSM events | File integrity measurement | v1.3.0 | — | D | GA; requires IMA + LSM hook support in kernel | [v1.3.0 #2818](https://github.com/cilium/tetragon/pull/2818) |
| File-type selectors (`FileType` / `NotFileType`) | Narrow monitoring to specific file types | v1.7.0 | — | D | GA | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| Dentry type support in generic sensor | Richer path/dentry context in events | v1.4.0 | Dentry resolution fixes v1.4.0 | D | GA | [v1.4.0 #3423](https://github.com/cilium/tetragon/pull/3423) |
| Path offload support | Kernel-side path resolution for performance | v1.5.0 | — | D | GA | [v1.5.0 #3480](https://github.com/cilium/tetragon/pull/3480) |
| Prefix matching for long strings (4096 chars) | Long path/binary name matching in selectors | v1.1.0 | — | D | GA; was 144 chars, extended to 4096 | [v1.1.0 #2069](https://github.com/cilium/tetragon/pull/2069) |

---

## 3. Network & IPC Visibility

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| Socket/SKB argument types | Network connection telemetry | v0.8.0 | Socket + sockaddr support v1.4.0; validation fix v1.1.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| TCP connect/accept/listen telemetry | Lateral movement, C2 detection | v0.8.0 | TCP listen example policy v1.3.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| IP/CIDR helpers in CEL filters | Network allow/deny list filtering | v1.3.0 | — | D | GA | [v1.3.0 #3211](https://github.com/cilium/tetragon/pull/3211) |
| IPv6 support in BPF rate limit | Consistent rate limiting for IPv6 traffic | v1.0.0 | — | D | GA | [v1.0.0 #1458](https://github.com/cilium/tetragon/pull/1458) |
| AF_UNIX socket path extraction | Container runtime socket abuse detection | v1.7.0 | — | D | GA; enables containerd.sock / docker.sock visibility | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| `struct socket` / `struct sockaddr` support | Richer socket metadata in events | v1.4.0 | — | D | GA | [v1.4.0 #3358](https://github.com/cilium/tetragon/pull/3358) |
| BPF program argument type | eBPF program abuse / suspicious BPF activity | v1.6.0 | — | D | GA | [v1.6.0 #4124](https://github.com/cilium/tetragon/pull/4124) |

---

## 4. Kernel & User-Space Instrumentation

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| kprobes / kretprobes | Kernel function-level telemetry | v0.8.0 | Multi-symbol kprobe spec v1.3.0; kprobe multi (multi-link) v0.8.3+ | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Tracepoints | Stable kernel instrumentation | v0.8.0 | — | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Raw tracepoints | Lower-overhead tracing | v1.5.0 | — | D | GA | [v1.5.0 #3558](https://github.com/cilium/tetragon/pull/3558) |
| LSM sensor | Security hook coverage (file/process/network) | v1.2.0 | LSM resolve flag kernel-version check removed v1.4.0 | D | GA; requires kernel LSM BPF support | [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0) |
| uprobes / uretprobes | User-space function telemetry | v0.9.0 | Multi-link uprobe v1.1.0; uprobe arguments v1.1.0; uprobe actions v1.5.0; uprobe override action v1.6.0 | D+P | GA | [v0.9.0](https://github.com/cilium/tetragon/releases/tag/v0.9.0) |
| USDT sensor | Static user-space probe points | v1.6.0 | USDT action support v1.6.0; USDT set action v1.6.0; USDT policy resolve v1.6.0 | D | GA; amd64 primary; requires USDT-instrumented binaries | [v1.6.0 #3943](https://github.com/cilium/tetragon/pull/3943) |
| fentry sensor | Fast entry hook via BPF trampolines | v1.7.0 | — | D | GA; requires kernel ≥5.5 with BTF | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| Kernel stack traces | Call-stack context for suspicious activity | v1.0.0 | Renamed `kernel_stack_trace` v1.1.0 (was `stack_trace`); stacktrace-tree removal scheduled | D | ALPHA in v1.0.0; API rename v1.1.0 | [v1.0.0 #1429](https://github.com/cilium/tetragon/pull/1429) |
| User-space stack traces | Full user-land call stack context | v1.1.0 | — | D | GA | [v1.1.0 #2175](https://github.com/cilium/tetragon/pull/2175) |
| BTF / attribute resolution | Portable kernel argument access (CO-RE) | v0.8.0 | `get_current_task_btf` v1.4.0; BTF cache removal (memory opt) v1.3.0 | D | GA; requires kernel BTF | Multiple |
| 32-bit ARM (aarch32) syscall support | Cross-architecture coverage | v1.3.0 | — | D | GA | [v1.3.0 #2898](https://github.com/cilium/tetragon/pull/2898) |
| 32-bit syscall matching on x86 | Mixed-mode binary detection | v1.1.0 | — | D | GA | [v1.1.0 #1816](https://github.com/cilium/tetragon/pull/1816) |
| Syscall ABI identification (`syscall64` SyscallId type) | Distinguish 32-bit vs 64-bit syscall abuse | v1.3.0 | Compatibility flag removed v1.4.0 | D | GA; replaces `size_arg` type | [v1.3.0 #2986](https://github.com/cilium/tetragon/pull/2986) |
| Windows process create/exit | Cross-platform process monitoring | v1.5.0 | — | D | ALPHA; limited to process events; no full eBPF parity | [v1.5.0 #3577](https://github.com/cilium/tetragon/pull/3577) |

---

## 5. Detection & Policy Engine

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| TracingPolicy CRD | Declarative detection rules | v0.8.0 | Namespaced policies + pod-label filters on by default v1.0.0; validation hardening over all releases | D+P | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| `matchBinaries` selector | Target specific binaries in rules | v0.8.0 | Rework v1.1.0; Prefix/NotPrefix v1.1.0; followChildren fix v1.3.0; matchBinary name+args v1.3.0 | D+P | GA | Multiple |
| `matchArgs` selectors (string/int/IP/range) | Argument-based rule conditions | v0.8.0 | String length 4096 v1.1.0; range filter v1.6.0; prefix fix v1.1.0 | D+P | GA | Multiple |
| `matchCapabilities` / capability-change filters | Privilege escalation detection | v1.1.0 | op_capabilities_gained consistency fix v1.6.1 | D | GA | [v1.1.0](https://github.com/cilium/tetragon/releases/tag/v1.1.0) |
| `matchNamespaces` | Namespace-scoped rules | v0.9.0 | — | D+P | GA | [v0.9.0](https://github.com/cilium/tetragon/releases/tag/v0.9.0) |
| Pod label filters | Workload-targeted policies | v0.9.0 | Enabled by default v1.0.0; namespace-label filtering v1.1.0 | D+P | GA | [v0.9.0](https://github.com/cilium/tetragon/releases/tag/v0.9.0) |
| Container name selector (`containerSelector`) | Per-container policy scoping | v1.1.0 | — | D+P | GA | [v1.1.0 #2231](https://github.com/cilium/tetragon/pull/2231) |
| Policy tagging (`policy_name` in events) | Rule attribution in alerts | v1.0.0 | — | D | GA | [v1.0.0 #1574](https://github.com/cilium/tetragon/pull/1574) |
| Rate limiting (`rateLimit`, `rateLimitScope`) | Alert noise reduction | v1.0.0 | `rateLimitScope` v1.1.0 | D | GA | [v1.0.0](https://github.com/cilium/tetragon/releases/tag/v1.0.0) |
| Policy lists (named symbol/string lists) | Reusable rule components | v1.0.0 | — | D+P | GA | [v1.0.0 #1401](https://github.com/cilium/tetragon/pull/1401) |
| CEL export filter | Flexible event-time filtering | v1.3.0 | IP/CIDR CEL helpers v1.3.0; CEL CLI filter v1.4.0 | D | GA | [v1.3.0 #3098](https://github.com/cilium/tetragon/pull/3098) |
| CEL evaluation in BPF | In-kernel CEL-based policy evaluation | v1.7.0 | — | D+P | GA (new in v1.7) | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| `matchParentBinaries` selector | Parent process binary–based rules | v1.7.0 | — | D+P | GA | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| `hostSelector` | Target host-level (non-pod) processes | v1.7.0 | — | D+P | GA | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| Monitoring mode in TracingPolicy | Safe rollout without enforcement | v1.4.0 | — | D | GA | [v1.4.0 #3393](https://github.com/cilium/tetragon/pull/3393) |
| policyfilter (pod/namespace scope) | Multi-tenant policy isolation | v0.9.0 | cgtracker support v1.3.0; beta → stable v1.3.0 | D+P | GA from v1.3.0 (was BETA) | [v0.9.0](https://github.com/cilium/tetragon/releases/tag/v0.9.0) |
| `followChildren` in matchBinaries | Track child processes of matched binaries | v1.0.0 | Fix v1.3.0 | D+P | GA | [v1.3.0 #3186](https://github.com/cilium/tetragon/pull/3186) |
| Regex filter for parent process arguments | Parent-arg pattern matching | v1.3.0 | — | D | GA | [v1.3.0 #3155](https://github.com/cilium/tetragon/pull/3155) |
| Policy tags | Categorization / alert routing | v1.7.0 | — | D | GA | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| `FollowFD` / `UnfollowFD` / `CopyFD` actions | File descriptor tracking | v0.8.0 | Deprecated v1.4.0; **removed v1.5.0** | D | **REMOVED** in v1.5.0 | [v1.4.0 #3491](https://github.com/cilium/tetragon/pull/3491) |
| Sensor API (gRPC) | Programmatic sensor management | v0.8.0 | Deprecated in v1.3.0; **removed v1.4.0** | D | **REMOVED** in v1.4.0 | [v1.4.0 #3437](https://github.com/cilium/tetragon/pull/3437) |

---

## 6. Enforcement & Response

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| Signal action (SIGKILL) | Terminate malicious processes | v0.8.0 | — | P | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Override action (syscall return override) | Block specific syscalls | v0.8.0 | — | P | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Killer sensor (initial enforcement substrate) | Prevent malicious actions at kernel level | v1.0.0 | Renamed to **Enforcer** v1.1.0 | P | GA (as Enforcer from v1.1.0) | [v1.0.0 #1205](https://github.com/cilium/tetragon/pull/1205) |
| Enforcer sensor (`killers` → `enforcers`) | Kernel-level enforcement via fmod_ret | v1.1.0 | security_* hooks v1.1.0; persistent enforcement v1.2.0; namespace support v1.3.0 | P | GA | [v1.1.0 #2117](https://github.com/cilium/tetragon/pull/2117) |
| fmod_ret support in enforcer | Override kernel function return values | v1.1.0 | — | P | GA | [v1.1.0 #1953](https://github.com/cilium/tetragon/pull/1953) |
| `security_*` hook support in enforcer | LSM-level enforcement | v1.1.0 | — | P | GA; requires kernel LSM BPF | [v1.1.0 #2002](https://github.com/cilium/tetragon/pull/2002) |
| Persistent enforcement | Enforcement survives Tetragon restart | v1.2.0 | Dedicated Helm flag v1.3.0 | P | GA | [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0) |
| `NotifyEnforcer` action | Trigger enforcer from kprobe/tracepoint event | v1.1.0 | Validation: reject without Enforcer v1.6.0 | P | GA | [v1.1.0](https://github.com/cilium/tetragon/releases/tag/v1.1.0) |
| Uprobe override action | Override user-space function return values | v1.6.0 | — | P | GA | [v1.6.0 #4173](https://github.com/cilium/tetragon/pull/4173) |
| Action stats per policy | Enforcement action telemetry | v1.6.0 | — | D | GA | [v1.6.0 #4074](https://github.com/cilium/tetragon/pull/4074) |
| Enforcer missed notifications metric | Blind-spot detection for enforcement | v1.3.0 | — | D | GA | [v1.3.0 #2994](https://github.com/cilium/tetragon/pull/2994) |
| Monitoring mode | Non-enforcing observation for safe rollout | v1.4.0 | — | D | GA | [v1.4.0 #3393](https://github.com/cilium/tetragon/pull/3393) |

---

## 7. Kubernetes & Container Context

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| Pod/namespace/workload enrichment | Workload attribution of events | v0.8.0 | Progressive improvements across all releases | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Container metadata enrichment | Container-level event attribution | v0.8.0 | Privileged container flag v1.5.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Cgroup-based pod association | Accurate container→pod mapping | v0.8.0 | Improved cgroupv1/v2 handling v1.3.0; nested cgroup support v1.3.0; relax deployment detection v1.4.0 | D | GA | [v1.3.0 #3170](https://github.com/cilium/tetragon/pull/3170) |
| Nested cgroup pod association | Nested container / VM-in-container support | v1.3.0 | — | D | GA | [v1.3.0 #3170](https://github.com/cilium/tetragon/pull/3170) |
| Pod UID field | Unique pod identity in events | v1.6.0 | — | D | GA | [v1.6.0 #4069](https://github.com/cilium/tetragon/pull/4069) |
| Pod annotations in events | Policy & metadata enrichment | v1.5.0 | — | D | GA | [v1.5.0 #3527](https://github.com/cilium/tetragon/pull/3527) |
| Node labels | Node-level policy targeting | v1.5.0 | — | D | GA | [v1.5.0](https://github.com/cilium/tetragon/releases/tag/v1.5.0) |
| OCI hook setup (rthooks) | Container-start process tracking | v1.1.0 | Deprecated `OciHookSetup` v1.2.0; Helm section removed v1.5.0 | D | GA (NRI path); OciHookSetup section **REMOVED** v1.5.0 | [v1.1.0 #1842](https://github.com/cilium/tetragon/pull/1842) |
| NRI (Node Resource Interface) rthooks | Alternative container runtime hook | v1.2.0 | — | D | GA | [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0) |
| Privileged container flag | Privileged container detection | v1.5.0 | — | D | GA | [v1.5.0 #3661](https://github.com/cilium/tetragon/pull/3661) |
| Non-Kubernetes deployment support | Bare-metal / VM use | v0.8.0 | Node name set to hostname in standalone mode v1.1.0; k8s control plane for non-k8s deployment v1.6.0 | D | GA | Multiple |
| Multiple operator replicas (HA) | Operator high availability | v1.4.0 | Operator non-root default v1.6.0 | D | GA | [v1.4.0 #3443](https://github.com/cilium/tetragon/pull/3443) |
| Operator non-root default | Deployment hardening | v1.6.0 | — | D | GA | [v1.6.0 #3909](https://github.com/cilium/tetragon/pull/3909) |

---

## 8. Export, Integration & Privacy

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| gRPC event stream (`GetEvents` API) | Real-time event consumption by SIEM/EDR | v0.8.0 | — | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| JSON file export | Log-file based ingestion | v0.8.0 | Export file permissions default 0600 v1.0.0; permission rotation fix v1.3.0 | D | GA | [v0.8.0](https://github.com/cilium/tetragon/releases/tag/v0.8.0) |
| Field filters | Reduce exported event fields for privacy | v0.8.0 | Many segfault and perf fixes v1.1.0; top-level field regression fix v1.1.0 | D | GA | Multiple |
| Redaction filters | Censor sensitive string data | v1.1.0 | — | D | GA | [v1.1.0 #2243](https://github.com/cilium/tetragon/pull/2243) |
| Capability export filter | Filter events by process capabilities | v1.1.0 | — | D | GA | [v1.1.0 #2107](https://github.com/cilium/tetragon/pull/2107) |
| `in_init_tree` export filter | Filter container init-tree events | v1.3.0 | — | D | GA | [v1.3.0 #3209](https://github.com/cilium/tetragon/pull/3209) |
| `container_id` export filter | Filter by container ID | v1.3.0 | — | D | GA | [v1.3.0 #3209](https://github.com/cilium/tetragon/pull/3209) |
| CEL-based export filter | Flexible client-side event filtering | v1.3.0 | IP/CIDR helpers v1.3.0 | D | GA | [v1.3.0 #3098](https://github.com/cilium/tetragon/pull/3098) |
| Container name regex filter in CLI | Targeted event monitoring | v1.6.0 | — | D | GA | [v1.6.0 #4051](https://github.com/cilium/tetragon/pull/4051) |
| `cluster_name` field in `GetEventsResponse` | Multi-cluster event correlation | v1.3.0 | — | D | GA | [v1.3.0 #3025](https://github.com/cilium/tetragon/pull/3025) |
| Unix domain socket server default | Reduced attack surface for agent API | v1.7.0 | — | D | GA; was `localhost:54321`, now `/var/run/tetragon/tetragon.sock` | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| Event log service | Structured event logging | v1.7.0 | — | D | GA | [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0) |
| `--expose-kernel-addresses` flag | (Was) expose kernel addresses in events | v0.8.0 | **Removed** v1.3.0 | — | **REMOVED** v1.3.0 | [v1.3.0 #3042](https://github.com/cilium/tetragon/pull/3042) |

---

## 9. Operations, Performance & Hardening

| Capability | EDR Use | First Stable Release | Later Enhancements / Fixes | Type | Maturity / Limitations | Evidence |
|-----------|---------|---------------------|---------------------------|------|----------------------|---------|
| Ring buffer / perf buffer tuning (`--rb-size`) | High-throughput event pipelines | v0.9.0 | Buffer between perf reader and processing v1.0.0 | D | GA | [v0.9.0 #480](https://github.com/cilium/tetragon/pull/480) |
| BPF ring buffer default (kernel ≥5.11) | Improved throughput and reduced overhead | v1.6.0 | — | D | GA; perf buffer fallback for older kernels | [v1.6.0 #4075](https://github.com/cilium/tetragon/pull/4075) |
| Missed events metric per type | Blind-spot awareness | v1.1.0 | — | D | GA | [v1.1.0 #1674](https://github.com/cilium/tetragon/pull/1674) |
| Policy/action metrics | Per-policy enforcement observability | v1.6.0 | — | D | GA | [v1.6.0 #4074](https://github.com/cilium/tetragon/pull/4074) |
| BPF map kernel memory metrics | Memory footprint visibility | v1.3.0 | — | D | GA | [v1.3.0 #2984](https://github.com/cilium/tetragon/pull/2984) |
| Process/event cache metrics | Cache health monitoring | v1.1.0 | EventCache retries metric v1.1.0; GC interval configurable v1.3.0 | D | GA | Multiple |
| BPF overhead metrics | Instrumentation overhead tracking | v1.3.0 | Overhead metrics for return probes fix v1.3.0 | D | GA | [v1.3.0 #3040](https://github.com/cilium/tetragon/pull/3040) |
| Diagnostics / bugtool | Incident response data collection | v0.9.0 | Memory info v1.3.0; BPF map dump v1.3.0; pprof heap v1.1.0 | D | GA | Multiple |
| Debug programs command (`tetra debug progs`) | Policy debugging | v1.3.0 | — | D | GA | [v1.3.0 #2967](https://github.com/cilium/tetragon/pull/2967) |
| Process cache dump | In-memory process state inspection | v1.3.0 | — | D | GA | [v1.3.0 #2246](https://github.com/cilium/tetragon/pull/2246) |
| Memory footprint reduction | Reduced idle memory use | v1.2.0 | BTF/kallsyms cache removal v1.3.0 | D | GA | [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0) |
| BPF error metrics | BPF operation error visibility | v1.3.0 | — | D | GA | [v1.3.0 #3205](https://github.com/cilium/tetragon/pull/3205) |

---

*See individual module documents for full detail. Cross-references: [process-and-lineage](modules/process-and-lineage.md) · [file-and-integrity](modules/file-and-integrity.md) · [network-and-ipc](modules/network-and-ipc.md) · [kernel-and-userspace-instrumentation](modules/kernel-and-userspace-instrumentation.md) · [detection-policy-engine](modules/detection-policy-engine.md) · [enforcement-and-response](modules/enforcement-and-response.md) · [kubernetes-and-container-context](modules/kubernetes-and-container-context.md) · [export-integration-and-privacy](modules/export-integration-and-privacy.md) · [operations-performance-and-hardening](modules/operations-performance-and-hardening.md)*
