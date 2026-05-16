# A Tale of Two Servers, Part 3
## NUMA, vCPU/Socket Settings, and Final Summary

**Author:** Tobias Kreidl (@tjkreidl)
**Originally Published:** May 7, 2019 on CUGC (Citrix User Group Community)
**Archived Source:** [Wayback Machine Snapshot (2022-05-27)](https://web.archive.org/web/20220527215004/https://www.mycugc.org/blogs/tobias-kreidl/2019/04/30/a-tale-of-two-servers-part-3)

> *Archived and preserved for the XCP-ng community. Original content © Tobias Kreidl.*

---

## Introduction

Parts 1 and 2 covered BIOS tuning and GPU configuration. Part 3 focuses on the final and most impactful layer: **NUMA topology**, **vCPU/socket configuration**, and the **vCPU oversubscription ratio**.

---

## Understanding vNUMA

**Non-Uniform Memory Access (NUMA)** means each CPU socket has its own local memory. Accessing remote memory (on the other socket) incurs ~30-40% higher latency.

### When vNUMA Kicks In

> ⚠️ **Critical:** As soon as a VM's vCPU count exceeds the physical core count of a single socket, XCP-ng must span the VM across NUMA nodes — and performance drops significantly.

For a dual E5-2680 v4 server (14 cores per socket):
- **Safe zone:** 1–14 vCPUs (fits within one socket)
- **Danger zone:** 15+ vCPUs (spans both sockets)

### Checking NUMA Topology on XCP-ng

```bash
# Check host NUMA topology
xl info -n

# Check vCPU-to-physical-CPU assignments
xl vcpu-list

# Check specific VMs
xl vcpu-list <VM1> <VM2>
```

---

## Benchmark Results by vCPU Count

| vCPUs | NUMA Nodes Used | GPU Utilization | Relative FPS |
|---|---|---|---|
| 8 | 1 | 97% | 100% (baseline) |
| 14 | 1 | 97% | ~99% |
| 16 | 2 (even split) | 91% | ~94% |
| 26 | 2 (uneven split) | 81% | ~87% |
| 32 | 2 (even split) | 89% | ~92% |

> 💡 **26 vCPUs performed worse than 32** — the uneven split (14+12) across sockets caused worse scheduling than the even 16+16 split.

---

## vCPU Topology: Sockets and Cores

### Optimal Setting

> ✅ **Best Practice: 1 socket × all cores** (e.g., 1 socket × 8 cores, NOT 8 sockets × 1 core)

```bash
# Set via xe CLI
xe vm-param-set uuid=<vm-uuid> platform:cores-per-socket=8
```

### Why This Matters

- **Licensing:** SQL Server, Windows Server, and some other software is licensed per socket. Fewer virtual sockets = lower licensing costs.
- **Scheduler efficiency:** The Xen scheduler handles single-socket VMs more efficiently.
- **L1TF patch impact:** If the L1 Terminal Fault speculative execution patch is applied, no two workloads share the same physical core — factor this into vCPU planning.

---

## The 1.5:1 vCPU Oversubscription Rule

From Nick Rintalan (Citrix) and Login VSI benchmarks:

> **Do not exceed a 1.5:1 vCPU-to-pCPU ratio for XenApp/RDSH workloads.**

For a dual E5-2680 v4 server (28 physical cores total):

| Scenario | Physical Cores | Max vCPUs at 1.5:1 |
|---|---|---|
| HT Disabled | 28 | 42 |
| HT Enabled | 56 | 84 |

> ⚠️ **Hyperthreading note:** For XenApp workloads, many engineers recommend disabling HT. The 1.5:1 rule applies to **physical cores**, not HT threads.

---

## Cache Architecture and NUMA Latency

### CPU Cache Hierarchy (E5-2680 v4)

| Cache | Size | Latency | Scope |
|---|---|---|---|
| L1 | 32KB per core | ~4 cycles | Per core |
| L2 | 256KB per core | ~12 cycles | Per core |
| LLC (L3) | 35MB shared | ~36 cycles | Per socket |
| Remote NUMA | — | ~100+ cycles | Cross-socket |

> This is why keeping VMs within a single NUMA node matters so much — crossing the NUMA boundary multiplies memory latency by 3× or more compared to LLC access.

---

## VM Startup Order

> ✅ **Start your most important VMs first** to get the best NUMA placement.

XCP-ng assigns NUMA nodes on a first-come, first-served basis. VMs started first get the most favorable placement. Use XO's **VM startup sequence** settings or `xe vm-start` ordering in your startup scripts.

---

## Memory Must Also Fit Within One NUMA Node

It's not just vCPUs — **RAM must also fit within a single NUMA node**:

For a dual-socket server with 192GB total RAM (96GB per socket):
- ✅ VM with 64GB RAM → fits in one NUMA node
- ⚠️ VM with 128GB RAM → must span both NUMA nodes

---

## Final Best Practices Checklist

### BIOS Settings

| Setting | Value |
|---|---|
| CPU Power Management | OS DBPM |
| C1E | Enabled |
| C States | Enabled |
| Turbo Mode | Enabled |
| Uncore Frequency | Dynamic |
| Node Interleaving | Disabled |
| Hyperthreading | Disabled (XenApp) |
| Memory Frequency | Maximum Performance |

### XCP-ng Host Commands

```bash
# Set CPU governor (run on each host)
xenpm set-scaling-governor performance

# Make persistent across reboots
/opt/xensource/libexec/xen-cmdline --set-xen cpufreq=xen:performance

# Verify turbo mode
xenpm get-cpufreq-para | grep turbo

# Check NUMA topology
xl info -n

# Check vCPU placement
xl vcpu-list
```

### VM Configuration

| Setting | Value |
|---|---|
| vCPU count | ≤ physical cores per socket |
| Sockets | 1 |
| Cores per socket | = total vCPU count |
| RAM | ≤ RAM per NUMA node |
| GPU scheduler | Best Effort |

---

## Acknowledgements

Special thanks to the following for their research and documentation:

- **Frank Denneman** — NUMA deep dives and CPU scheduler analysis (frankdenneman.nl)
- **Nick Rintalan** — vCPU oversubscription guidelines (Citrix Blog)
- **Johan van Amersfoort** — VDI Design Guide (vhojan.nl)
- **Login VSI** — Scalability benchmarking methodology

---

## Architecture Caveat

> ⚠️ These results were obtained on **Intel Haswell/Broadwell-EP** architecture (E5-2680 v4). Results may differ significantly on Skylake, Cascade Lake, Ice Lake, or AMD EPYC platforms due to differences in NUMA topology, cache hierarchy, and memory architecture. Always benchmark your specific hardware.

---

*[← Part 2: GPU Settings & Scheduling](./tale-of-two-servers-part-2.md) | [Summary & Quick Reference →](./tale-of-two-servers-summary-xcpng-quick-reference.md)*
