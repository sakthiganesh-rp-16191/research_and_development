# Module: Network & IPC Visibility

> **Navigation:** [README](../README.md) | [Matrix](../edr-capability-matrix.md) | [Release Timeline](../release-timeline.md)
>
> **EDR Role:** Network telemetry identifies command-and-control communication, lateral movement, data exfiltration, and local inter-process communication abuse. Tetragon provides socket-level visibility tied to process context, which standard network monitoring (NetFlow, packet capture) cannot provide.

---

## Table of Contents

1. [Socket & SKB Telemetry](#1-socket--skb-telemetry)
2. [TCP Connection Monitoring](#2-tcp-connection-monitoring)
3. [IP / CIDR Filtering](#3-ip--cidr-filtering)
4. [IPv6 Support](#4-ipv6-support)
5. [AF_UNIX Socket Path Extraction](#5-af_unix-socket-path-extraction)
6. [BPF Program Argument Type](#6-bpf-program-argument-type)
7. [Rate Limiting for Network Events](#7-rate-limiting-for-network-events)
8. [Limitations](#8-limitations)
9. [EDR Design Notes](#9-edr-design-notes)

---

## 1. Socket & SKB Telemetry

### Baseline Socket/SKB Argument Types — **[NEW v0.8.0]**

Tetragon supports `sock` (socket) and `skb` (socket buffer / packet) as argument types in TracingPolicy kprobe rules. This enables instrumentation of kernel network functions that operate on sockets and packets.

**Socket fields available:**
- Source/destination IP and port
- Protocol (TCP, UDP, etc.)
- Socket state (connected, listening, etc.)
- Address family (AF_INET, AF_INET6, AF_UNIX)

### `struct socket` and `struct sockaddr` Support — **[NEW v1.4.0]**

**Evidence:** [#3358](https://github.com/cilium/tetragon/pull/3358)

Added support for reading `struct socket` and `struct sockaddr` directly as argument types, enabling richer socket metadata in events (socket type, address structure).

**[FIX v1.1.0]** Validation added for `sock` and `skb` types to prevent misconfigured policies ([#1807](https://github.com/cilium/tetragon/pull/1807)).

---

## 2. TCP Connection Monitoring

### TCP Connect / Accept / Listen Events — **[NEW v0.8.0]**

Tetragon's built-in network sensors and TracingPolicy examples provide:
- **TCP connect** — outbound connection attempts with destination IP/port and process context
- **TCP accept** — inbound connection acceptance with source IP/port
- **TCP listen** — socket bind/listen events (TCP listen example policy added in v1.3.0 [#2929](https://github.com/cilium/tetragon/pull/2929))

**EDR use cases:**

| Scenario | Signal |
|----------|--------|
| C2 beaconing | Unexpected process opening outbound TCP connections |
| Reverse shell | Shell process accepting inbound TCP connection |
| Port scanner | Rapid sequential connect events from same process |
| Container runtime socket abuse | Unix socket connect to `/var/run/docker.sock` (see §5) |
| Lateral movement | Unexpected SSH/RDP/SMB connections from non-admin processes |

**Instrumented kernel functions (common examples):**
```
tcp_connect
inet_csk_accept
inet_listen
security_socket_connect
security_socket_bind
```

---

## 3. IP / CIDR Filtering

### IP/CIDR Helpers in CEL Filters — **[NEW v1.3.0]**

**Evidence:** [#3211](https://github.com/cilium/tetragon/pull/3211)

Tetragon v1.3.0 adds IP address and CIDR range helpers to the Common Expression Language (CEL) filter engine. This enables:
- Filtering events to/from specific IP ranges
- Exclusion of RFC1918 (internal) addresses from C2-detection rules
- CIDR-based allow/deny list evaluation at export time

**Example CEL expression:**
```
!(destination.ip.isInRange("10.0.0.0/8") || destination.ip.isInRange("192.168.0.0/16"))
```

**EDR value:** Reduces alert noise for internal network traffic without writing separate SIEM correlation rules.

---

## 4. IPv6 Support

### IPv6 in BPF Rate Limit — **[NEW v1.0.0]**

**Evidence:** [#1458](https://github.com/cilium/tetragon/pull/1458)

BPF rate limiting for network events now correctly handles IPv6 addresses, preventing bypasses of rate-limit policies by using IPv6 endpoints.

---

## 5. AF_UNIX Socket Path Extraction

### AF_UNIX Socket Path in Events — **[NEW v1.7.0]**

**Evidence:** [v1.7.0](https://github.com/cilium/tetragon/releases/tag/v1.7.0)

Tetragon v1.7.0 adds extraction of AF_UNIX (Unix domain socket) paths in network events. This enables direct visibility into:

| Unix Socket Path | Threat Scenario |
|-----------------|----------------|
| `/var/run/docker.sock` | Docker socket abuse for container escape |
| `/var/run/containerd/containerd.sock` | Containerd socket abuse |
| `/var/run/crio/crio.sock` | CRI-O socket abuse |
| `/run/k8s/kubelet.sock` | Kubelet API abuse |
| `/tmp/*.sock`, `/run/*.sock` | Suspicious custom Unix socket use |

**EDR value:** Container runtime socket abuse (e.g., a compromised container process connecting to `/var/run/docker.sock` to escape) is a high-confidence container escape indicator. Prior to v1.7.0, Unix socket connections were visible but without the socket path.

**Why it matters:** Unlike TCP connections, Unix socket connections are purely local. Detecting a non-privileged container process connecting to `/var/run/docker.sock` is a direct indicator of a container escape attempt or privilege escalation.

---

## 6. BPF Program Argument Type

### `bpf_prog` Argument Type — **[NEW v1.6.0]**

**Evidence:** [#4124](https://github.com/cilium/tetragon/pull/4124)

Support for `bpf_prog` as an argument type in kprobe rules enables monitoring of eBPF program load events. This provides:
- Detection of suspicious BPF program loading by processes
- Visibility into which processes are loading BPF programs
- Detection of potential eBPF rootkits / offensive BPF tooling

**EDR value:** eBPF-based rootkits (e.g., TripleCross, ebpfkit) load BPF programs as part of their installation. Tetragon can detect unauthorized BPF program loading.

---

## 7. Rate Limiting for Network Events

Network hooks (especially on busy systems) can generate extremely high event volumes. Tetragon provides:
- **`rateLimit`** in TracingPolicy — per-policy event rate limiting
- **`rateLimitScope`** — introduced v1.1.0, controls whether rate limiting applies per-process or globally ([#1962](https://github.com/cilium/tetragon/pull/1962))
- **IPv6 rate-limit correctness** — v1.0.0

**Recommendation:** For network monitoring, rate limiting is essential. Start with a rate limit that captures connection establishment events (first connection) while suppressing per-packet events.

---

## 8. Limitations

| Limitation | Detail |
|-----------|--------|
| **No packet content capture** | Tetragon records connection metadata (IP, port, process), not packet payload. |
| **No NetFlow/IPFIX output** | Tetragon events are process-context-enriched events, not flow records. Integration with a flow collector is separate. |
| **High event volume for network-heavy workloads** | Without rate limiting and careful selector scoping, network monitoring can overwhelm the ring buffer. |
| **Encrypted traffic** | TCP/TLS connection metadata is visible, but encrypted payload is not. Use uprobes on TLS libraries for plaintext access (see [kernel-and-userspace-instrumentation.md](kernel-and-userspace-instrumentation.md)). |
| **AF_UNIX path requires v1.7.0+** | Unix socket paths were not extracted in earlier versions. |
| **No built-in firewall integration** | Tetragon can detect but does not natively update firewall rules. Prevention requires Override/Signal actions or integration with network policy enforcement. |
| **UDP datagram telemetry** | UDP is instrumentable via `udp_sendmsg`/`udp_recvmsg` kprobes, but less common in default policies. |

---

## 9. EDR Design Notes

### Process-Attributed Network Events

The key differentiator of Tetragon network events vs. NetFlow/packet capture is **process attribution**:

```
Event: TCP connect
  process: /usr/bin/python3 (pid=12345)
  parent: /bin/bash (pid=12300)
  ancestors: sshd → bash → python3
  destination: 185.220.101.x:443 (Tor exit node range)
  pod: web-frontend-abc123
  namespace: production
```

This single event provides:
- LOLBin detection (Python making outbound TLS)
- Lineage context (spawned from interactive shell)
- Workload attribution (production web-frontend)
- C2 indicator (known-bad IP)

### Recommended Detection Rules

1. **Outbound connections from shell processes** — bash/sh/zsh making TCP connections
2. **Reverse shell indicators** — shell binary as both the parent and the process making inbound accept calls
3. **Docker/containerd socket connects** — any process connecting to container runtime sockets (high-confidence container escape)
4. **DNS-over-HTTPS or unusual DNS ports** — non-standard DNS usage
5. **Scanning behavior** — rapid sequential connection attempts from the same process

---

*Related: [Detection Policy Engine](detection-policy-engine.md) · [Kernel & Userspace Instrumentation](kernel-and-userspace-instrumentation.md) · [Export, Integration & Privacy](export-integration-and-privacy.md)*
