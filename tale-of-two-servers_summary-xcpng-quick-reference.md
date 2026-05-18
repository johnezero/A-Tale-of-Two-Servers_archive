# A Tale of Two Servers — Summary & XCP-ng Quick Reference

**Original Series Author:** Tobias Kreidl (@tjkreidl)
**Summary Compiled By:** XCP-ng Community Archive
**Series Originally Published:** 2019 on CUGC (Citrix User Group Community)

> *This document summarizes the key concepts from the three-part "A Tale of Two Servers" series, with specific callouts for XCP-ng and Xen Orchestra (XO) administrators. Links point to the relevant sections in the full archived articles.*

---

## 📖 About This Series

This three-part series by Tobias Kreidl is a deep dive into how **BIOS settings, GPU configuration, NUMA architecture, and VM vCPU assignments** all interact to dramatically affect server and application performance — particularly in **XenServer/XCP-ng environments with NVIDIA GPUs**.

The servers in focus are the **Dell R720 and R730**, but the lessons apply broadly to any multi-socket Intel Xeon server running XCP-ng with GPU workloads.

> **Why it matters:** Through careful tuning alone (no hardware changes), GPU benchmark scores improved by ~36%, GPU utilization rose from 82% → 97%, and maximum CPU frequency increased from 2.40 GHz → 3.145 GHz (+31%).

---

## Part 1 — BIOS Settings & Their Effect on Apps and GPU Performance

📄  [Full Article](https://github.com/tobiaskreidl/Citrix-Tobias-Kreidl-Collection/blob/XenServer-Articles/A%20Tale%20of%20Two%20Servers_%20How%20BIOS%20Settings%20Can%20Affect%20Your%20Apps%20and%20GPU%20Performance.pdf)


### Key Concepts

- **Don't trust default BIOS settings** — out-of-the-box configurations are rarely optimal for virtualization workloads
- **CPU Power Management must be set to "OS DBPM"** (not "System DBPM") on Dell R730s — otherwise the OS/hypervisor cannot control CPU frequency scaling
- **Turbo mode misconfiguration** caused a ~48% GPU performance drop in benchmarks — a silent killer
- **C-states must be enabled** to allow turbo mode to function at its maximum potential
- **Enabling turbo** correctly boosted max CPU frequency from 2.40 GHz → 3.145 GHz (+31%)

### 🔧 XCP-ng / XO Specific

| Topic | Detail |
|---|---|
| Set CPU frequency governor | `xenpm set-scaling-governor performance` |
| Make governor setting persistent across reboots | `/opt/xensource/libexec/xen-cmdline --set-xen cpufreq=xen:performance` |
| Verify turbo mode is active | `xenpm get-cpufreq-para \| grep turbo` |
| Monitor average CPU frequency during load | `watch -n 5 'xenpm start 1\|grep "Avg freq" \| sort \| tail -16'` |
| Check available frequency states | `xenpm get-cpufreq-states` |
| Check idle states | `xenpm get-cpuidle-states` |

> ⚠️ The `xenpm set-scaling-governor performance` command must be run on **each XCP-ng host individually**.

---

## Part 2 — GPU Settings, Scheduling, and Further BIOS Tuning

📄 [Full Article](https://github.com/tobiaskreidl/Citrix-Tobias-Kreidl-Collection/blob/XenServer-Articles/A%20Tale%20of%20Two%20Servers%2C%20Part%202_%20How%20Not%20Only%20BIOS%20Settings%2C%20But%20Also%20GPU%20Settings%20Can%20Affect%20Your%20Apps%20and%20GPU%20Performance.pdf)

### Key Concepts

- **Uncore Frequency should be set to "Dynamic"** (not "Maximum") — allows the uncore to match the fastest core's speed, avoiding a CPU-to-GPU data bottleneck
- **C1E state** (enhanced halt) reduces voltage in addition to stopping the internal clock — enable it
- **NVIDIA GPU Schedulers** — three options available on Pascal and newer GPUs:
  - **Best Effort** *(default)* — no idle waste; best for heavy/consistent GPU loads
  - **Equal Share** — fair but wastes idle GPU cycles
  - **Fixed Share** — QoS-focused; most idle waste; best for cloud/guaranteed minimums
- **GPU choice matters** — NVIDIA M10, M60, P4, T4 each suit different density/performance tradeoffs
- AMD and Intel GPU options also available and supported in XCP-ng

### 🔧 XCP-ng / XO Specific

| Topic | Detail |
|---|---|
| Check NVIDIA vGPU scheduler | `nvidia-smi vgpu -s` |
| Set vGPU scheduler mode | `nvidia-smi vgpu -i <gpu-id> -ss <0\|1\|2>` |
| Check NVIDIA driver version | `nvidia-smi` (on host) |
| Recommended BIOS Uncore Frequency | Set to **Dynamic** (not Maximum) |
| Recommended BIOS C1E setting | **Enabled** |

> 💡 **Informal community consensus:** Most experienced admins stick with **Best Effort** scheduling — it's the only mode where idle GPU cycles are not wasted.

---

## Part 3 — NUMA, vCPU/Socket Settings, and Putting It All Together

📄 [Full Article](https://github.com/tobiaskreidl/Citrix-Tobias-Kreidl-Collection/blob/XenServer-Articles/A%20Tale%20of%20Two%20Servers%2C%20Part%203_%20The%20Influence%20of%20NUMA%2C%20CPUs%20and%20Sockets_Cores-per-Socket%2C%20plus%20Other%20VM%20Settings%20on%20Apps%20and%20GPU%20Performance.pdf)

### Key Concepts

- **vNUMA kicks in** as soon as a VM's vCPU count exceeds the physical core count of a single socket — this is the single biggest hidden performance trap
- **8 or 14 vCPUs** (fitting within one socket on a dual E5-2680 v4 system) gave the best benchmark results
- **26 vCPUs** caused an odd core split between sockets → GPU utilization dropped from 97% to 81–91%
- **VM startup order matters** — start your most important VMs first to get the best NUMA placement
- **1 socket / all cores** is the optimal vCPU topology (e.g., 1 socket × 8 cores, not 8 sockets × 1 core)
- **Nick Rintalan's 1.5–2:1 vCPU oversubscription rule** — total vCPUs across all VMs should be 1.5–2× the physical core count
- Sometimes **more, smaller VMs** outperform fewer, larger ones
- **Memory must also fit within a single NUMA node** — not just vCPUs

### 🔧 XCP-ng / XO Specific

| Topic | Detail |
|---|---|
| Check host NUMA topology | `xl info -n` |
| Check vCPU-to-physical-CPU assignments | `xl vcpu-list` |
| Check vCPU assignments for specific VMs | `xl vcpu-list <VM1> <VM2>` |
| Optimal vCPU topology in XO VM editor | Set **Cores per socket** = total vCPUs, **Sockets** = 1 |
| Avoid odd vCPU counts above socket core count | Causes uneven NUMA splits |
| Start critical VMs first | Ensures best NUMA node placement |

> ⚠️ **Licensing note:** Some software (e.g., Microsoft SQL Server) is licensed per socket. Setting sockets = 1 also avoids unnecessary licensing costs.

> ⚠️ **L1TF patch note:** If the L1 Terminal Fault speculative execution patch is applied, no two workloads will share the same core — factor this into your vCPU planning.

---

## ⚡ Quick Reference: Most Important Settings at a Glance

### BIOS (Dell R730 / Similar Intel Xeon Servers)

| Setting | Recommended Value |
|---|---|
| CPU Power Management | **OS DBPM** |
| System Profile | **Custom** |
| CPU Performance | **Maximum Performance** |
| C1E | **Enabled** |
| C States | **Enabled** |
| Energy Efficient Turbo | **Enabled** |
| Uncore Frequency | **Dynamic** |
| Memory Frequency | **Maximum Performance** |
| Node Interleaving | **Disabled** |
| Collaborative CPU Performance Control | **Disabled** |
| Number of Turbo Boot Enabled Cores | **All** |
| Monitor/Mwait | **Enabled** |

### XCP-ng Host Commands

```bash
# Set CPU governor to performance (run on each host)
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

### VM Configuration (in XO or via CLI)

| Setting | Recommended Value |
|---|---|
| vCPU count | ≤ physical cores per socket |
| Sockets | **1** |
| Cores per socket | = total vCPU count |
| Memory | ≤ RAM per NUMA node |
| GPU NUMA alignment | Align VM to GPU's NUMA node |

---

## 📚 Full Article Links

| Article | Description |
|---|---|
| [Part 1 — BIOS Settings](https://github.com/tobiaskreidl/Citrix-Tobias-Kreidl-Collection/blob/XenServer-Articles/A%20Tale%20of%20Two%20Servers_%20How%20BIOS%20Settings%20Can%20Affect%20Your%20Apps%20and%20GPU%20Performance.pdf) | CPU Power Management, turbo mode, xenpm commands |
| [Part 2 — GPU Settings](./tale-of-two-servers-part-2.md) | Uncore frequency, GPU schedulers, GPU vendor comparison |
| [Part 3 — NUMA & vCPU](./tale-of-two-servers-part-3.md) | NUMA topology, vCPU sizing, socket settings, oversubscription |

---

## 🔗 Original Wayback Machine Sources

- [Part 1 — Wayback Machine](https://web.archive.org/web/20220527221535/https://www.mycugc.org/blogs/tobias-kreidl/2019/03/07/tale-of-two-servers-bios-settings-affect-apps-gpu)
- [Part 2 — Wayback Machine](https://web.archive.org/web/20220527213026/https://www.mycugc.org/blogs/tobias-kreidl/2019/04/30/a-tale-of-two-servers-part-2)
- [Part 3 — Wayback Machine](https://web.archive.org/web/20220527215004/https://www.mycugc.org/blogs/tobias-kreidl/2019/04/30/a-tale-of-two-servers-part-3)

---

*Preserved by the XCP-ng community. Special thanks to @john.c for the web sleuthing, and to @tjkreidl for the original content.*
