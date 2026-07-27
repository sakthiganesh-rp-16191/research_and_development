# Sources — Tetragon EDR Research Pack

> **Navigation:** [README](README.md) | [Matrix](edr-capability-matrix.md) | [Release Timeline](release-timeline.md) | [Adoption Guide](edr-adoption-guide.md)
>
> All stable release links and important referenced pull requests, grouped by release and then by module. All links point to the upstream `cilium/tetragon` GitHub repository.

---

## Primary Source

| Resource | URL |
|---------|-----|
| Tetragon GitHub Repository | https://github.com/cilium/tetragon |
| Tetragon Releases (all) | https://github.com/cilium/tetragon/releases |
| Tetragon Documentation | https://tetragon.io/docs |

---

## Stable Release Links

| Release | Tag | Release Notes URL |
|---------|-----|------------------|
| v0.8.0 | Initial | https://github.com/cilium/tetragon/releases/tag/v0.8.0 |
| v0.8.1 | Patch | https://github.com/cilium/tetragon/releases/tag/v0.8.1 |
| v0.8.2 | Patch | https://github.com/cilium/tetragon/releases/tag/v0.8.2 |
| v0.8.3 | Patch | https://github.com/cilium/tetragon/releases/tag/v0.8.3 |
| v0.8.4 | Patch | https://github.com/cilium/tetragon/releases/tag/v0.8.4 |
| v0.9.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v0.9.0 |
| v0.10.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v0.10.0 |
| v0.11.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v0.11.0 |
| v1.0.0 | Major | https://github.com/cilium/tetragon/releases/tag/v1.0.0 |
| v1.0.1 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.0.1 |
| v1.0.2 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.0.2 |
| v1.0.3 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.0.3 |
| v1.1.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v1.1.0 |
| v1.1.2 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.1.2 |
| v1.2.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v1.2.0 |
| v1.2.1 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.2.1 |
| v1.3.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v1.3.0 |
| v1.4.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v1.4.0 |
| v1.4.1 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.4.1 |
| v1.5.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v1.5.0 |
| v1.6.0 | Minor | https://github.com/cilium/tetragon/releases/tag/v1.6.0 |
| v1.6.1 | Patch | https://github.com/cilium/tetragon/releases/tag/v1.6.1 |
| v1.7.0 | Minor (latest as of 2026-07-27) | https://github.com/cilium/tetragon/releases/tag/v1.7.0 |

---

## Important Pull Requests by Module

### Process & Lineage

| PR | Description | Release |
|----|-------------|---------|
| [#1296](https://github.com/cilium/tetragon/pull/1296) | UID/GID credentials and privileged execution detection in process_exec | v1.0.0 |
| [#1429](https://github.com/cilium/tetragon/pull/1429) | Kernel stack traces alpha feature in kprobe events | v1.0.0 |
| [#499](https://github.com/cilium/tetragon/pull/499) | Anonymous binary detection (memfd/fileless execution) | v1.1.0 |
| [#1786](https://github.com/cilium/tetragon/pull/1786) | Detection of binary execution that raises process privileges | v1.1.0 |
| [#1256](https://github.com/cilium/tetragon/pull/1256) | Improved TID/PID handling in GetProcessCopy | v1.0.0 |
| [#1559](https://github.com/cilium/tetragon/pull/1559) | Read task real parent for accurate parent tracking | v1.0.0 |
| [#888](https://github.com/cilium/tetragon/pull/888) | ProcessCredentials object and credential tracking | v0.11.0 |
| [#895](https://github.com/cilium/tetragon/pull/895) | Monitor process credentials at syscalls | v0.11.0 |
| [#1189](https://github.com/cilium/tetragon/pull/1189) | Record Linux capability usage | v0.11.0 |
| [#2938](https://github.com/cilium/tetragon/pull/2938) | Ancestors in process events | v1.4.0 |
| [#3209](https://github.com/cilium/tetragon/pull/3209) | in_init_tree flag + container_id/in_init_tree export filters | v1.3.0 |
| [#3039](https://github.com/cilium/tetragon/pull/3039) | Fix process exit signal when core dumped | v1.3.0 |
| [#3338](https://github.com/cilium/tetragon/pull/3338) | Fix in_init_tree for processes started before Tetragon | v1.4.0 |
| [#3827](https://github.com/cilium/tetragon/pull/3827) | Fix inInitTree not properly accounting pre-Tetragon processes | v1.5.0 |
| [#3186](https://github.com/cilium/tetragon/pull/3186) | Fix tracking of matchBinary children | v1.3.0 |
| [#1390](https://github.com/cilium/tetragon/pull/1390) | Kernel module operation tracking | v1.0.0 |
| [#3972](https://github.com/cilium/tetragon/pull/3972) | Fix argument corruption with long executable filenames | v1.6.0 |
| [#3974](https://github.com/cilium/tetragon/pull/3974) | Robust process argument parsing | v1.6.0 |
| [#4257](https://github.com/cilium/tetragon/pull/4257) | Memory leak fix in process and event caches | v1.6.1 |

### File & Integrity

| PR | Description | Release |
|----|-------------|---------|
| [#2818](https://github.com/cilium/tetragon/pull/2818) | IMA hashes in LSM events | v1.3.0 |
| [#3427](https://github.com/cilium/tetragon/pull/3427) | Fix path truncation — dentry read extended to 4096 bytes | v1.4.0 |
| [#3423](https://github.com/cilium/tetragon/pull/3423) | Dentry type in generic sensor | v1.4.0 |
| [#3261](https://github.com/cilium/tetragon/pull/3261) | Fix vfsmnt assignment | v1.4.0 |
| [#1902](https://github.com/cilium/tetragon/pull/1902) | prepend_name BPF function fixes | v1.1.0 |
| [#3480](https://github.com/cilium/tetragon/pull/3480) | Path offload support | v1.5.0 |

### Network & IPC

| PR | Description | Release |
|----|-------------|---------|
| [#3211](https://github.com/cilium/tetragon/pull/3211) | IP/CIDR helpers in CEL filters | v1.3.0 |
| [#1458](https://github.com/cilium/tetragon/pull/1458) | IPv6 support in BPF rate limit | v1.0.0 |
| [#3358](https://github.com/cilium/tetragon/pull/3358) | struct socket / struct sockaddr argument types | v1.4.0 |
| [#1807](https://github.com/cilium/tetragon/pull/1807) | Validation for sock and skb argument types | v1.1.0 |
| [#2929](https://github.com/cilium/tetragon/pull/2929) | TCP listen example TracingPolicy | v1.3.0 |
| [#4124](https://github.com/cilium/tetragon/pull/4124) | BPF program argument type (bpf_prog) | v1.6.0 |

### Kernel & Userspace Instrumentation

| PR | Description | Release |
|----|-------------|---------|
| [#1914](https://github.com/cilium/tetragon/pull/1914) | Multi-link uprobe support | v1.1.0 |
| [#1978](https://github.com/cilium/tetragon/pull/1978) | Uprobe arguments support | v1.1.0 |
| [#2175](https://github.com/cilium/tetragon/pull/2175) | User-space stack traces | v1.1.0 |
| [#953](https://github.com/cilium/tetragon/pull/953) | LSM and Tracing program loading | v0.11.0 |
| [#3558](https://github.com/cilium/tetragon/pull/3558) | Raw tracepoints | v1.5.0 |
| [#3943](https://github.com/cilium/tetragon/pull/3943) | USDT sensor | v1.6.0 |
| [#4078](https://github.com/cilium/tetragon/pull/4078) | USDT action support | v1.6.0 |
| [#4005](https://github.com/cilium/tetragon/pull/4005) | USDT set action | v1.6.0 |
| [#4198](https://github.com/cilium/tetragon/pull/4198) | USDT policy resolve support | v1.6.0 |
| [#4173](https://github.com/cilium/tetragon/pull/4173) | Uprobe override action | v1.6.0 |
| [#3676](https://github.com/cilium/tetragon/pull/3676) | Uprobes with actions | v1.5.0 |
| [#2986](https://github.com/cilium/tetragon/pull/2986) | ABI information for syscall64 type | v1.3.0 |
| [#2898](https://github.com/cilium/tetragon/pull/2898) | 32-bit ARM (aarch32) syscall support | v1.3.0 |
| [#1816](https://github.com/cilium/tetragon/pull/1816) | 32-bit syscall matching on x86 | v1.1.0 |
| [#2948](https://github.com/cilium/tetragon/pull/2948) | All operators for syscall64 type including Mask | v1.3.0 |
| [#1986](https://github.com/cilium/tetragon/pull/1986) | linux_binprm CO-RE extraction | v1.1.0 |
| [#3143](https://github.com/cilium/tetragon/pull/3143) | Attribute resolution via BTF | v1.4.0 |
| [#3305](https://github.com/cilium/tetragon/pull/3305) | get_current_task_btf BPF helper | v1.4.0 |
| [#2937](https://github.com/cilium/tetragon/pull/2937) | BTF/kallsyms cache removal (memory optimization) | v1.3.0 |
| [#3577](https://github.com/cilium/tetragon/pull/3577) | Windows: process create/exit observer changes | v1.5.0 |
| [#3578](https://github.com/cilium/tetragon/pull/3578) | Windows: process create/exit sensor changes | v1.5.0 |

### Detection & Policy Engine

| PR | Description | Release |
|----|-------------|---------|
| [#1574](https://github.com/cilium/tetragon/pull/1574) | policy_name field in kprobe/tracepoint/uprobe events | v1.0.0 |
| [#1647](https://github.com/cilium/tetragon/pull/1647) | Namespaced policies + pod label filters on by default | v1.0.0 |
| [#1731](https://github.com/cilium/tetragon/pull/1731) | matchBinaries selector rework | v1.1.0 |
| [#1926](https://github.com/cilium/tetragon/pull/1926) | Read process exe at execve for matchBinaries | v1.1.0 |
| [#1732](https://github.com/cilium/tetragon/pull/1732) | Prefix/NotPrefix operators for matchBinaries | v1.1.0 |
| [#2231](https://github.com/cilium/tetragon/pull/2231) | containerSelector for per-container policy scoping | v1.1.0 |
| [#1962](https://github.com/cilium/tetragon/pull/1962) | rateLimitScope for per-process vs. global rate limiting | v1.1.0 |
| [#2107](https://github.com/cilium/tetragon/pull/2107) | Capability-based export filter | v1.1.0 |
| [#3098](https://github.com/cilium/tetragon/pull/3098) | CEL export filter | v1.3.0 |
| [#3124](https://github.com/cilium/tetragon/pull/3124) | CEL filter in tetra CLI | v1.4.0 |
| [#3393](https://github.com/cilium/tetragon/pull/3393) | Monitoring mode in TracingPolicy | v1.4.0 |
| [#3155](https://github.com/cilium/tetragon/pull/3155) | Regex filter for parent process arguments | v1.3.0 |
| [#3210](https://github.com/cilium/tetragon/pull/3210) | matchBinary name + args combined matching | v1.3.0 |
| [#3523](https://github.com/cilium/tetragon/pull/3523) | BPF complexity fix in selectors | v1.4.0 |
| [#3056](https://github.com/cilium/tetragon/pull/3056) | policyfilter beta → stable (GA) | v1.3.0 |
| [#1401](https://github.com/cilium/tetragon/pull/1401) | Policy lists documentation | v1.0.0 |
| [#2069](https://github.com/cilium/tetragon/pull/2069) | String matching extended to 4096 characters | v1.1.0 |
| [#1806](https://github.com/cilium/tetragon/pull/1806) | Prefix/postfix matching fix for long strings | v1.1.0 |
| [#4022](https://github.com/cilium/tetragon/pull/4022) | Empty matchBinaries correctly ignored | v1.6.0 |
| [#4170](https://github.com/cilium/tetragon/pull/4170) | Selector off-by-one bounds check fix | v1.6.0 |
| [#3437](https://github.com/cilium/tetragon/pull/3437) | Deprecated sensor gRPC API removed | v1.4.0 |
| [#3491](https://github.com/cilium/tetragon/pull/3491) | FollowFD/UnfollowFD/CopyFD actions deprecated | v1.4.0 |
| [#4109](https://github.com/cilium/tetragon/pull/4109) | Range filter for matchArgs | v1.6.0 |

### Enforcement & Response

| PR | Description | Release |
|----|-------------|---------|
| [#1205](https://github.com/cilium/tetragon/pull/1205) | Killer sensor introduction | v1.0.0 |
| [#2117](https://github.com/cilium/tetragon/pull/2117) | Killer renamed to Enforcer; NotifyKiller → NotifyEnforcer | v1.1.0 |
| [#1953](https://github.com/cilium/tetragon/pull/1953) | fmod_ret support in killer/enforcer sensor | v1.1.0 |
| [#2002](https://github.com/cilium/tetragon/pull/2002) | security_* functions in killer/enforcer | v1.1.0 |
| [#2977](https://github.com/cilium/tetragon/pull/2977) | Dedicated Helm flag for persistent enforcement | v1.3.0 |
| [#2994](https://github.com/cilium/tetragon/pull/2994) | tetragon_enforcer_missed_notifications_total metric | v1.3.0 |
| [#4008](https://github.com/cilium/tetragon/pull/4008) | NotifyEnforcer rejected without Enforcer present | v1.6.0 |
| [#4173](https://github.com/cilium/tetragon/pull/4173) | Uprobe override action | v1.6.0 |
| [#4074](https://github.com/cilium/tetragon/pull/4074) | Action counters per TracingPolicy | v1.6.0 |
| [#3393](https://github.com/cilium/tetragon/pull/3393) | Monitoring mode in TracingPolicy | v1.4.0 |

### Kubernetes & Container Context

| PR | Description | Release |
|----|-------------|---------|
| [#888](https://github.com/cilium/tetragon/pull/888) | ProcessCredentials object and credential tracking | v0.11.0 |
| [#3170](https://github.com/cilium/tetragon/pull/3170) | Nested cgroup pod association | v1.3.0 |
| [#3053](https://github.com/cilium/tetragon/pull/3053) | Improved cgroupv1/cgroupv2 handling | v1.3.0 |
| [#3400](https://github.com/cilium/tetragon/pull/3400) | Relaxed deployment detection logic | v1.4.0 |
| [#4069](https://github.com/cilium/tetragon/pull/4069) | Pod UID field in events | v1.6.0 |
| [#3527](https://github.com/cilium/tetragon/pull/3527) | Pod annotations in events | v1.5.0 |
| [#3661](https://github.com/cilium/tetragon/pull/3661) | Privileged container flag in PodInfo | v1.5.0 |
| [#1842](https://github.com/cilium/tetragon/pull/1842) | OCI hook setup (rthooks) | v1.1.0 |
| [#3443](https://github.com/cilium/tetragon/pull/3443) | Multiple operator replicas HA support | v1.4.0 |
| [#3909](https://github.com/cilium/tetragon/pull/3909) | Operator non-root default | v1.6.0 |
| [#1931](https://github.com/cilium/tetragon/pull/1931) | Run Tetragon without CRD access | v1.1.0 |
| [#2123](https://github.com/cilium/tetragon/pull/2123) | Node name set to hostname in standalone mode | v1.1.0 |
| [#3180](https://github.com/cilium/tetragon/pull/3180) | cgtracker policyfilter support | v1.3.0 |
| [#2930](https://github.com/cilium/tetragon/pull/2930) | Deleted pod cache | v1.3.0 |
| [#2188](https://github.com/cilium/tetragon/pull/2188) | Container FS scan fix for CRI-O | v1.1.0 |
| [#4011](https://github.com/cilium/tetragon/pull/4011) | K8s control plane for non-K8s deployment | v1.6.0 |
| [#4060](https://github.com/cilium/tetragon/pull/4060) | Reduced RBAC for non-K8s deployment | v1.6.0 |
| [#3382](https://github.com/cilium/tetragon/pull/3382) | CRI socket metrics (cgidmap) | v1.4.0 |
| [#1410](https://github.com/cilium/tetragon/pull/1410) | PodInfo CRD | v1.0.0 |

### Export, Integration & Privacy

| PR | Description | Release |
|----|-------------|---------|
| [#1575](https://github.com/cilium/tetragon/pull/1575) | Export file permissions default changed to 0600 | v1.0.0 |
| [#2243](https://github.com/cilium/tetragon/pull/2243) | Redaction filters for sensitive strings | v1.1.0 |
| [#2107](https://github.com/cilium/tetragon/pull/2107) | Capability-based export filter | v1.1.0 |
| [#3209](https://github.com/cilium/tetragon/pull/3209) | in_init_tree + container_id export filters | v1.3.0 |
| [#3025](https://github.com/cilium/tetragon/pull/3025) | cluster_name field in GetEventsResponse | v1.3.0 |
| [#1867](https://github.com/cilium/tetragon/pull/1867) | Policy name filter in tetra getevents | v1.1.0 |
| [#3044](https://github.com/cilium/tetragon/pull/3044) | Fix --policy-names to apply to all event types | v1.3.0 |
| [#4051](https://github.com/cilium/tetragon/pull/4051) | Container name regex filter in CLI | v1.6.0 |
| [#3438](https://github.com/cilium/tetragon/pull/3438) | --reconnect option for tetra getevents | v1.4.0 |
| [#1432](https://github.com/cilium/tetragon/pull/1432) | Use message copy when applying fieldFilters in exec events | v1.0.0 |
| [#1700](https://github.com/cilium/tetragon/pull/1700) | Fix segfault in PID field filtering | v1.1.0 |
| [#1882](https://github.com/cilium/tetragon/pull/1882) | Fix top-level info missing from events (field filter regression) | v1.1.0 |
| [#3042](https://github.com/cilium/tetragon/pull/3042) | Remove --expose-kernel-addresses flag | v1.3.0 |
| [#1444](https://github.com/cilium/tetragon/pull/1444) | Metrics label filter configuration | v1.0.0 |
| [#1549](https://github.com/cilium/tetragon/pull/1549) | Event type filter in tetra getevents | v1.0.0 |

### Operations, Performance & Hardening

| PR | Description | Release |
|----|-------------|---------|
| [#480](https://github.com/cilium/tetragon/pull/480) | --rb-size / --rb-size-total perf ring buffer options | v0.9.0 |
| [#593](https://github.com/cilium/tetragon/pull/593) | Buffer between perf reader and event processing | v1.0.0 |
| [#4075](https://github.com/cilium/tetragon/pull/4075) | BPF ring buffer default on kernel ≥5.11 | v1.6.0 |
| [#1674](https://github.com/cilium/tetragon/pull/1674) | Missed events metric per event type | v1.1.0 |
| [#2994](https://github.com/cilium/tetragon/pull/2994) | Enforcer missed notifications metric | v1.3.0 |
| [#3040](https://github.com/cilium/tetragon/pull/3040) | BPF overhead metrics | v1.3.0 |
| [#3074](https://github.com/cilium/tetragon/pull/3074) | Fix overhead metrics for return probes | v1.3.0 |
| [#3217](https://github.com/cilium/tetragon/pull/3217) | Aggregate overhead metrics in userspace | v1.3.0 |
| [#2984](https://github.com/cilium/tetragon/pull/2984) | BPF map memory metrics per TracingPolicy | v1.3.0 |
| [#3205](https://github.com/cilium/tetragon/pull/3205) | BPF error metrics | v1.3.0 |
| [#4074](https://github.com/cilium/tetragon/pull/4074) | Action counters per TracingPolicy | v1.6.0 |
| [#2162](https://github.com/cilium/tetragon/pull/2162) | Metrics initialized to 0 at startup | v1.1.0 |
| [#2090](https://github.com/cilium/tetragon/pull/2090) | Policy status through metrics and tetra CLI | v1.1.0 |
| [#2246](https://github.com/cilium/tetragon/pull/2246) | Process cache dump (tetra dump processcache) | v1.3.0 |
| [#2967](https://github.com/cilium/tetragon/pull/2967) | Debug programs command (tetra debug progs) | v1.3.0 |
| [#2959](https://github.com/cilium/tetragon/pull/2959) | Debug maps command (tetra debug maps) | v1.3.0 |
| [#2007](https://github.com/cilium/tetragon/pull/2007) | pprof heap in bugtool | v1.1.0 |
| [#2880](https://github.com/cilium/tetragon/pull/2880) | Memory-related info in bugtool | v1.3.0 |
| [#2963](https://github.com/cilium/tetragon/pull/2963) | BPF map JSON dump in bugtool | v1.3.0 |
| [#2937](https://github.com/cilium/tetragon/pull/2937) | BTF/kallsyms cache removal | v1.3.0 |
| [#1650](https://github.com/cilium/tetragon/pull/1650) | Userspace/BPF struct alignment check at startup | v1.0.0 |
| [#2928](https://github.com/cilium/tetragon/pull/2928) | Configurable event cache retry count and delay | v1.3.0 |
| [#3130](https://github.com/cilium/tetragon/pull/3130) | Process cache GC interval configurable | v1.3.0 |
| [#3346](https://github.com/cilium/tetragon/pull/3346) | map_errors_update_total / map_errors_delete_total metrics | v1.4.0 |
| [#4257](https://github.com/cilium/tetragon/pull/4257) | Memory leak fix in process and event caches | v1.6.1 |
| [#4020](https://github.com/cilium/tetragon/pull/4020) | tetra probe config kernel compatibility check | v1.6.0 |
| [#3909](https://github.com/cilium/tetragon/pull/3909) | Operator non-root default | v1.6.0 |
| [#3443](https://github.com/cilium/tetragon/pull/3443) | Multiple operator replicas HA | v1.4.0 |

---

## Additional Reference Documentation

| Resource | URL |
|---------|-----|
| Stack Traces migration guide | https://tetragon.io/docs/concepts/tracing-policy/selectors/#stack-traces |
| TracingPolicy reference | https://tetragon.io/docs/concepts/tracing-policy/ |
| Tetragon helm chart | https://github.com/cilium/tetragon/tree/main/install/kubernetes |
| Example TracingPolicies | https://github.com/cilium/tetragon/tree/main/examples/tracingpolicy |
| Policy library | https://github.com/cilium/tetragon/tree/main/bpf/policy |
| Tetragon architecture | https://tetragon.io/docs/reference/architecture/ |

---

*Research conducted from upstream release notes as of July 27, 2026. Latest stable release: v1.7.0.*
