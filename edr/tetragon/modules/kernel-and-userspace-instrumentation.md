# Module: Kernel & Userspace Instrumentation

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** Instrumentation breadth determines the "attack surface coverage" of Tetragon. This module documents every hook mechanism available — kprobes, tracepoints, LSM, uprobes, USDT, fentry — along with stack trace support, BTF-based argument resolution, and architecture/kernel constraints.

---

## Table of Contents

1. [kprobes & kretprobes](#1-kprobes--kretprobes)
2. [Tracepoints & Raw Tracepoints](#2-tracepoints--raw-tracepoints)
3. [LSM (Linux Security Module) Sensor](#3-lsm-linux-security-module-sensor)
4. [uprobes & uretprobes](#4-uprobes--uretprobes)
5. [USDT Sensor](#5-usdt-sensor)
6. [fentry Sensor](#6-fentry-sensor)
7. [Kernel Stack Traces](#7-kernel-stack-traces)
8. [User-Space Stack Traces](#8-user-space-stack-traces)
9. [BTF & Attribute Resolution](#9-btf--attribute-resolution)
10. [Syscall Handling & ABI Coverage](#10-syscall-handling--abi-coverage)
11. [Architecture & Kernel Constraints](#11-architecture--kernel-constraints)
12. [Windows Support](#12-windows-support)

---

## 1. kprobes & kretprobes

### Baseline kprobe support — **[NEW v0.8.0]**

kprobes are the primary Tetragon instrumentation mechanism. A kprobe attaches to any kernel function by name and executes a BPF program on entry. kretprobes do the same on function return.

**Capabilities:**
- Instrument any exported kernel function
- Read function arguments (entry) or return value (exit)
- Multiple argument types: `int`, `uint`, `string`, `bytes`, `sock`, `skb`, `file`, `path`, `dentry`, `cred`, `bpf_prog`, `fdinstall`, `syscall64`, etc.
- Boolean/string/integer/range operators in `matchArgs` selectors

**kprobe multi (multi-link) — [ENH v0.8.3]:**
- Attach a single BPF program to multiple kernel functions simultaneously
- Reduces BPF program count and improves load performance
- Fix for override program pin in v1.4.0 ([#3298](https://github.com/cilium/tetragon/pull/3298))

**Multi-symbol kprobe spec — [ENH v1.3.0]:**
Multiple symbols can be specified in a single kprobe spec ([#3121](https://github.com/cilium/tetragon/pull/3121)).

**Return filter moved to kernel — [ENH v1.1.0]:**
Return value filtering moved from userspace to the BPF program ([#1773](https://github.com/cilium/tetragon/pull/1773)), reducing event throughput for filtered return-value checks.

**Kernel constraints:** kprobes work on Linux ≥4.9. kprobe multi requires ≥5.10.

---

## 2. Tracepoints & Raw Tracepoints

### Tracepoints — **[NEW v0.8.0]**

Tracepoints are stable, named hook points built into the kernel at compile time. Unlike kprobes, they are not affected by kernel function inlining and have more stable argument structures.

**Advantages over kprobes:**
- More stable across kernel versions
- Explicitly maintained kernel ABI
- Lower risk of breakage during kernel upgrades

**TracingPolicy tracepoints:** Defined with `tracepoints:` in a TracingPolicy, referencing the tracepoint subsystem and event name (e.g., `syscalls/sys_enter_execve`).

### Raw Tracepoints — **[NEW v1.5.0]**

**Evidence:** [#3558](https://github.com/cilium/tetragon/pull/3558)

Raw tracepoints provide the same stable hook points as tracepoints but with:
- **Lower overhead** — raw tracepoints skip the argument-formatting layer
- **Unformatted arguments** — directly access the raw kernel structures
- Better suited for high-frequency events where performance is critical

**Kernel requirement:** Raw tracepoints are available from Linux ≥4.17.

---

## 3. LSM (Linux Security Module) Sensor

### LSM Sensor — **[NEW v1.2.0]**

**Evidence:** [v1.2.0](https://github.com/cilium/tetragon/releases/tag/v1.2.0)

The LSM sensor enables hooking Linux Security Module hook points. LSM hooks are called by the kernel before security-sensitive operations, making them ideal for:
- File access control (`security_file_open`, `security_inode_*`)
- Process execution control (`security_bprm_check`)
- Network security (`security_socket_connect`, `security_socket_bind`)
- IPC control
- Capability use (`security_capable`)

**IMA hashes in LSM events — [NEW v1.3.0]** — see [file-and-integrity.md](file-and-integrity.md).

**LSM resolve flag kernel-version check removed — [FIX v1.4.0]:**
The hard kernel version check for LSM resolve was removed ([#3415](https://github.com/cilium/tetragon/pull/3415)), allowing LSM programs to use attribute resolution on more kernels.

**LSM programs return bounded value — [FIX v1.3.0]:**
Ensures BPF LSM programs return values within the allowed range ([#3032](https://github.com/cilium/tetragon/pull/3032)), preventing verifier rejection.

**Kernel requirements:**
- BPF LSM: Linux ≥5.7, `CONFIG_BPF_LSM=y`
- LSM securityfs mount: Tetragon auto-mounts securityfs to check BPF LSM availability ([#3512](https://github.com/cilium/tetragon/pull/3512))

---

## 4. uprobes & uretprobes

### Generic uprobe sensor — **[NEW v0.9.0]**

uprobes hook user-space functions in any binary or library. This enables:
- Monitoring of security-sensitive library calls (OpenSSL, glibc auth functions)
- Detection of credential access in application-level code
- Visibility into encrypted traffic at the application layer

**Enhancements:**

| Release | Enhancement |
|---------|------------|
| v1.1.0 | Multi-link uprobe support (single BPF program, multiple functions) ([#1914](https://github.com/cilium/tetragon/pull/1914)) |
| v1.1.0 | `symbol` field replaced with `symbols` (array); policy migration required ([upgrade note](https://github.com/cilium/tetragon/releases/tag/v1.1.0)) |
| v1.1.0 | Uprobe arguments support ([#1978](https://github.com/cilium/tetragon/pull/1978)) — read function arguments from user-space functions |
| v1.5.0 | Uprobe actions (uprobes can now trigger TracingPolicy actions) ([#3676](https://github.com/cilium/tetragon/pull/3676)) |
| v1.6.0 | Uprobe override action (return value override for user-space functions) ([#4173](https://github.com/cilium/tetragon/pull/4173)) |
| v1.6.0 | Uprobe `sib` argument parsing ([#4095](https://github.com/cilium/tetragon/pull/4095)) |

**EDR use cases:**
- Hook `SSL_read`/`SSL_write` (OpenSSL) for plaintext TLS traffic inspection
- Hook `PAM` authentication functions to detect authentication bypass
- Hook `execve`-equivalent calls in scripting language runtimes
- Override user-space function return values for intrusion prevention

**Kernel requirements:** Uprobes: Linux ≥3.5. `uretprobes` require a patched or recent enough uprobe implementation.

---

## 5. USDT Sensor

### USDT (User-Space Defined Tracepoints) Sensor — **[NEW v1.6.0]**

**Evidence:** [#3943](https://github.com/cilium/tetragon/pull/3943)

USDT probes are statically defined, named tracepoints compiled into user-space binaries (similar to DTrace probes). Applications like Python, Ruby, Node.js, MySQL, PostgreSQL, and others include USDT probes.

**Features added in v1.6.0:**
- USDT sensor for attaching BPF programs to USDT probes ([#3943](https://github.com/cilium/tetragon/pull/3943))
- USDT action support (execute TracingPolicy actions from USDT events) ([#4078](https://github.com/cilium/tetragon/pull/4078))
- USDT `set` action ([#4005](https://github.com/cilium/tetragon/pull/4005))
- USDT policy `resolve:` support (BTF argument resolution) ([#4198](https://github.com/cilium/tetragon/pull/4198))

**EDR use cases:**
- Monitor Python/Ruby function calls without code modification
- Track MySQL/PostgreSQL query execution (data exfiltration detection)
- Observe JVM method calls (Java EDR coverage)

**Constraints:**
- Requires binaries compiled with USDT probes (SDT probes)
- amd64 is the primary supported architecture for USDT in v1.6.0
- Requires the target binary to be instrumented with `<sys/sdt.h>` or equivalent

---

## 6. fentry Sensor

### fentry — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

`fentry` (BPF function entry trampoline) is a modern alternative to kprobes for kernel function instrumentation. It uses BPF trampolines attached directly to function prologues.

**Advantages over kprobes:**
- Lower overhead (no `int3` breakpoint)
- Better verifier support for certain argument types
- More predictable behavior for inlined functions (where kernel BTF is available)

**Kernel requirements:** Linux ≥5.5 with BTF (`CONFIG_DEBUG_INFO_BTF=y`).

---

## 7. Kernel Stack Traces

### Kernel Stack Traces — **[NEW v1.0.0, ALPHA]**

**Evidence:** [#1429](https://github.com/cilium/tetragon/pull/1429)

Tetragon v1.0.0 introduces kernel stack trace capture in kprobe events as an **alpha feature**.

**EDR value:**
- Reveals the kernel call chain leading to the hooked function
- Helps identify bypasses or exploitation routes (e.g., which kernel path reached a `vfs_write` on a protected path)

**API evolution:**

| Release | Change |
|---------|--------|
| v1.0.0 | `stack_trace` field; alpha |
| v1.1.0 | **[BREAK]** Renamed to `kernel_stack_trace` to distinguish from user stack traces; `stackTrace` policy field renamed to `kernelStackTrace` |
| v1.7.0 | Legacy stacktrace-tree format deprecated/removed per upgrade notes |

**Constraints:**
- Alpha: API may change
- Requires kallsyms for symbol resolution; kallsyms cache removed for memory savings in v1.3.0 (online resolution used instead)

---

## 8. User-Space Stack Traces

### User-Space Stack Traces — **[NEW v1.1.0]**

**Evidence:** [#2175](https://github.com/cilium/tetragon/pull/2175)

Tetragon v1.1.0 adds support for capturing user-space call stacks in events. Enabled per-policy with `userStackTrace: true` in the Post action.

**EDR value:**
- Reveals the user-land call chain that led to a kernel hook
- Essential for detecting shellcode execution (unexpected user-land call stack)
- Detects injected code paths (JIT-sprayed or heap-sprayed shellcode)
- Complements kernel stack traces for full cross-boundary stack context

**Use case example:**
A `vfs_write` to a sensitive path triggered from `[unknown]` (unmapped/shellcode region) at the user-space level is a high-confidence code injection indicator.

---

## 9. BTF & Attribute Resolution

### BTF (BPF Type Format) — **[Core capability from v0.8.0]**

BTF enables CO-RE (Compile-Once, Run-Everywhere) eBPF programs that can read kernel structure fields by name rather than hardcoded offsets. Tetragon uses BTF to:
- Access kernel struct fields portably across kernel versions
- Read task/process/socket/file structures without per-kernel patches

**Enhancements:**

| Release | Enhancement |
|---------|------------|
| v1.4.0 | `get_current_task_btf` BPF helper ([#3305](https://github.com/cilium/tetragon/pull/3305)) |
| v1.4.0 | Attribute resolution via BTF for TracingPolicy `resolve:` ([#3143](https://github.com/cilium/tetragon/pull/3143)) |
| v1.3.0 | BTF cache removal — memory optimization; BTF data resolved on-demand ([#2937](https://github.com/cilium/tetragon/pull/2937)) |

**`linux_binprm` CO-RE — [ENH v1.1.0]:**
Portable extraction of `linux_binprm` struct members (executable path, arguments) using CO-RE ([#1986](https://github.com/cilium/tetragon/pull/1986)).

**Kernel requirement:** BTF: Linux ≥5.2 (`CONFIG_DEBUG_INFO_BTF=y`) for full CO-RE support.

---

## 10. Syscall Handling & ABI Coverage

### `syscall64` Type — **[Available from v0.8.0; enhanced v1.3.0]**

Tetragon's `syscall64` argument type handles system call numbers.

**ABI disambiguation — [ENH v1.3.0]:**

Prior to v1.3.0, different ABI syscall numbers (x86_64 vs i386) were output as the same `size_arg` integer, making them ambiguous. v1.3.0 changed the output to a `SyscallId` object that includes the ABI:

```json
{"syscall_id": {"id": 309, "abi": "x64"}}
{"syscall_id": {"id": 318, "abi": "i386"}}
```

**Compatibility flag:** `--enable-compatibility-syscall64-size-type` provided for one release cycle (v1.3.0 only; removed v1.4.0).

**32-bit syscall matching on x86 — [ENH v1.1.0]:** ([#1816](https://github.com/cilium/tetragon/pull/1816))

**32-bit ARM (aarch32) syscall support — [ENH v1.3.0]:** ([#2898](https://github.com/cilium/tetragon/pull/2898))

**All operators for syscall64 type — [ENH v1.3.0]:** Including `Mask` operator ([#2948](https://github.com/cilium/tetragon/pull/2948)).

---

## 11. Architecture & Kernel Constraints Summary

| Feature | Min Kernel | Notes |
|---------|-----------|-------|
| kprobes | Linux ≥4.9 | Universal |
| kprobe multi | Linux ≥5.10 | Multi-link attach |
| Tracepoints | Linux ≥4.9 | Stable ABI |
| Raw tracepoints | Linux ≥4.17 | Lower overhead |
| LSM BPF | Linux ≥5.7 | `CONFIG_BPF_LSM=y` |
| uprobes | Linux ≥3.5 | |
| USDT | Linux ≥3.5 | Requires instrumented binary |
| fentry | Linux ≥5.5 | Requires BTF |
| BPF ring buffer | Linux ≥5.8 (full: ≥5.11) | Default in Tetragon from v1.6.0 on ≥5.11 |
| BTF (CO-RE) | Linux ≥5.2 | `CONFIG_DEBUG_INFO_BTF=y` |
| Kernel stack traces | Linux ≥4.9 | Alpha maturity |
| User stack traces | Linux ≥4.9 | GA from v1.1.0 |
| CGROUPv2 + BPF | Linux ≥4.15 | |
| LSM IMA | Kernel must have IMA enabled | |

**cgroupv1 note:** Kernels ≥6.11 require specific cgroupv1 configurations — see [v1.4.0 docs note](https://github.com/cilium/tetragon/pull/3284).

---

## 12. Windows Support

### Windows Process Create/Exit — **[NEW v1.5.0, ALPHA-class]**

**Evidence:** [#3577](https://github.com/cilium/tetragon/pull/3577), [#3578](https://github.com/cilium/tetragon/pull/3578), [#3591](https://github.com/cilium/tetragon/pull/3591)

Tetragon v1.4.0–v1.5.0 adds initial Windows porting work. v1.5.0 delivers:
- Process create and exit event support on Windows
- Ring-buffer equivalent on Windows
- Observer/sensor compilation for Windows

**Scope:** Limited to process lifecycle events. The full eBPF-based kernel instrumentation (kprobes, LSM, uprobes, etc.) is Linux-specific and does not apply to Windows.

**Not available on Windows:** kprobes, LSM, uprobes, USDT, fentry, network socket telemetry, file telemetry, stack traces, BTF.

---

*Related: [Detection Policy Engine](detection-policy-engine.md) · [Enforcement & Response](enforcement-and-response.md) · [Operations, Performance & Hardening](operations-performance-and-hardening.md)*
