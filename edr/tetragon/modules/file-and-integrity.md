# Module: File & Integrity Monitoring

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** File telemetry provides visibility into read, write, execute, and attribute-change events on the filesystem. Integrity monitoring adds hash-based evidence. Together they enable ransomware detection, config/binary tampering detection, and compliance auditing.

---

## Table of Contents

1. [File Telemetry via Kprobes](#1-file-telemetry-via-kprobes)
2. [Path & Dentry Resolution](#2-path--dentry-resolution)
3. [File Permission Monitoring](#3-file-permission-monitoring)
4. [IMA Hashes in LSM Events](#4-ima-hashes-in-lsm-events)
5. [File-Type Selectors](#5-file-type-selectors)
6. [String & Path Matching Improvements](#6-string--path-matching-improvements)
7. [Path Offload Support](#7-path-offload-support)
8. [Limitations](#8-limitations)
9. [EDR Design Notes](#9-edr-design-notes)

---

## 1. File Telemetry via Kprobes

### Baseline File Instrumentation — **[NEW v0.8.0]**

Tetragon instruments kernel file-related functions (e.g., `vfs_read`, `vfs_write`, `vfs_open`, `vfs_unlink`, `security_file_open`, `security_inode_*`) via kprobes/LSM hooks to generate file access events.

Each event carries:
- **Path/filename** — resolved from the dentry (up to 4096 chars as of v1.4.0)
- **Process context** — full exec metadata of the accessing process
- **Kubernetes/container context** — pod, namespace, workload
- **Flags / mode** — file open flags, permissions where available

**Example instrumentation targets:**
```
vfs_write         → file write detection
security_file_open → file open with LSM hook
security_inode_rename → file rename / exfiltration staging
security_inode_unlink → file deletion
```

**EDR use cases:**
- Web shell / dropper write detection
- Configuration tampering (`/etc/passwd`, `sshd_config`, `sudoers`)
- Log deletion / anti-forensics
- Binary replacement / supply chain compromise

---

## 2. Path & Dentry Resolution

### Baseline dentry path resolution — **[NEW v0.8.0]**

Tetragon resolves file paths from kernel `dentry` structures at the time of the event.

### Path truncation fix to 4096 bytes — **[FIX v1.4.0]**

**Evidence:** [#3427](https://github.com/cilium/tetragon/pull/3427)

Prior to v1.4.0, the function responsible for reading dentry paths was capable of handling 4096 bytes, but some code paths still used the previous 256-byte limit, causing silent path truncation in event values for `cwd` and path/file function arguments.

**v1.4.0 fix:** All dentry read paths now use the 4096-byte limit. This is critical for accurate detection of long paths (e.g., deeply nested container overlay paths).

### Dentry type support in generic sensor — **[NEW v1.4.0]**

**Evidence:** [#3423](https://github.com/cilium/tetragon/pull/3423)

A dedicated `dentry` argument type was added to the generic sensor, enabling more precise path resolution in custom TracingPolicy rules.

**[FIX v1.4.0]** Additional dentry resolution fixes in the same release cycle ([#3450](https://github.com/cilium/tetragon/pull/3450)).

### `vfsmnt` assignment fix — **[FIX v1.4.0]**

**Evidence:** [#3261](https://github.com/cilium/tetragon/pull/3261)

Correct assignment of `vfsmnt` (virtual filesystem mount point) ensures that paths are resolved relative to the correct mount namespace, important in container environments with overlayfs.

### `prepend_name` BPF function fixes — **[FIX v1.1.0]**

**Evidence:** [#1902](https://github.com/cilium/tetragon/pull/1902)

Unit tests and fixes for the `prepend_name` BPF helper that builds path strings from dentry chains. Prevents incorrect path output for files in deep directory hierarchies.

---

## 3. File Permission Monitoring

### File Permission Tracing in Policies — **[NEW v1.1.0]**

TracingPolicy rules can instrument kernel functions that modify file permissions (e.g., `chmod`, `fchmod`, `security_inode_setattr`), generating events that include:
- File path being modified
- Old and new permission bits (where accessible from the hooked function)
- Process identity

**EDR use cases:**
- Detecting world-writable creation of executables (`chmod 777` on binaries)
- Monitoring permission changes on sensitive files
- Detecting SUID/SGID bit setting by non-root processes

---

## 4. IMA Hashes in LSM Events

### IMA Hashes in LSM Events — **[NEW v1.3.0]**

**Evidence:** [#2818](https://github.com/cilium/tetragon/pull/2818)

When using the LSM sensor (also introduced in v1.2.0), Tetragon can now record **IMA (Linux Integrity Measurement Architecture) hashes** in LSM events. This provides:

- **Cryptographic file fingerprint** at the time of access
- Direct integrity evidence without a separate IMA policy daemon
- Hash of the file content readable from the kernel IMA subsystem

**EDR value:**
- Detect tampered binaries that have been modified in place
- Correlate executed binary hash against a known-good allowlist (golden image comparison)
- Detect subtle binary modifications (e.g., function hooking / shimming)

**Constraints:**
- Requires IMA to be enabled and configured in the kernel (`CONFIG_IMA=y`)
- Requires the LSM sensor (kernel ≥5.7 for BPF LSM hooks)
- Hash availability depends on IMA measurement policy

---

## 5. File-Type Selectors

### `matchFileType` Selectors — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Tetragon v1.7.0 introduces file-type selectors that allow TracingPolicy rules to match events based on the type of the file (regular file, directory, symlink, socket, pipe, block device, character device, etc.).

**EDR use cases:**
- Monitor only regular file writes (exclude socket/pipe writes from file-write rules)
- Detect creation of device files in unexpected directories
- Narrow symlink-following policies to symlinks only
- Exclude directories from file-open events to reduce noise

**Example policy snippet:**
```yaml
selectors:
  - matchArgs:
    - index: 0
      operator: "Prefix"
      values:
        - "/etc"
    matchFileType:
      - "regular"
```

---

## 6. String & Path Matching Improvements

### Prefix / NotPrefix Operators — **[NEW v1.1.0]**

**Evidence:** [#1732](https://github.com/cilium/tetragon/pull/1732)

`matchBinaries` gains `Prefix` and `NotPrefix` operators, enabling path-prefix based binary matching (e.g., match any binary under `/usr/local/bin`).

### String Length Extended to 4096 Characters — **[NEW v1.1.0]**

**Evidence:** [#2069](https://github.com/cilium/tetragon/pull/2069)

String matching (including path matching) extended from 144 characters to 4096 characters. Earlier versions could silently mismatch strings that exceeded the 144-character limit.

**[ENH]** Prefix/postfix character limit increased to 256 characters in the same release ([#1779](https://github.com/cilium/tetragon/pull/1779)).

### Prefix/Postfix Length Fix — **[FIX v1.1.0]**

**Evidence:** [#1806](https://github.com/cilium/tetragon/pull/1806)

Fixed matching for strings longer than the prefix or postfix maximum length — previously, strings longer than the limit would fail to match even if the prefix/postfix was correct.

---

## 7. Path Offload Support

### Path Offload to Kernel — **[NEW v1.5.0]**

**Evidence:** [#3480](https://github.com/cilium/tetragon/pull/3480)

Path resolution can be offloaded to the BPF kernel program, reducing userspace work and event-processing latency for file-heavy workloads.

**EDR value:** Enables higher-throughput file monitoring without proportional userspace CPU overhead.

---

## 8. Limitations

| Limitation | Detail |
|-----------|--------|
| **No inotify equivalent** | Tetragon does not provide a persistent watch list in the inotify/fanotify style; it hooks kernel functions. Policy must be defined proactively. |
| **No file content capture** | Tetragon records file access metadata, not file contents. Actual content (e.g., written payload) is not recorded by default. |
| **IMA dependency** | File hashes require a working IMA configuration in the kernel. |
| **Path namespace correctness** | In complex multi-mount-namespace environments, path resolution requires correct `vfsmnt` context (fixed in v1.4.0). |
| **No rollback / quarantine** | Tetragon cannot roll back file writes or quarantine files. Prevention is limited to blocking the system call via Override or Signal actions. |
| **Large-file event volume** | Monitoring all `vfs_write` calls on a busy node generates extreme event volumes; selector-based scoping is mandatory. |
| **Kernel ≥5.7 for LSM hooks** | IMA hashes and LSM-based file events require BPF LSM support (Linux ≥5.7, `CONFIG_BPF_LSM=y`). |

---

## 9. EDR Design Notes

### Recommended Policy Strategy

1. **Prioritize high-value paths** — `/etc`, `/bin`, `/sbin`, `/usr/bin`, `/proc/*/mem`, `/dev`, cron directories, SSH authorized keys.
2. **Use FileType selectors** (v1.7.0+) to exclude noisy non-regular files.
3. **Combine with process context** — write events are most valuable when paired with the writing process's binary path and ancestry.
4. **Enable IMA** if you need hash-based integrity evidence; otherwise path + process context is the primary signal.
5. **Use monitoring mode first** ([`detection-policy-engine.md`](detection-policy-engine.md)) to understand event volume before enabling enforcement.

### Prevention Approach

Tetragon can prevent file operations by:
- Using the **Override** action on `vfs_write` / `vfs_open` calls to return `EPERM`
- Using the **Enforcer** with LSM `security_file_open` / `security_inode_create` hooks (requires kernel ≥5.7)

These are opt-in and require careful policy tuning to avoid blocking legitimate operations.

---

*Related: [Detection Policy Engine](detection-policy-engine.md) · [Kernel & Userspace Instrumentation](kernel-and-userspace-instrumentation.md) · [Enforcement & Response](enforcement-and-response.md)*
