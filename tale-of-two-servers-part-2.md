# A Tale of Two Servers, Part 2
## GPU Settings, Scheduling, and Further BIOS Tuning

**Author:** Tobias Kreidl (@tjkreidl)
**Originally Published:** April 30, 2019 on CUGC (Citrix User Group Community)
**Archived Source:** [Wayback Machine Snapshot (2022-05-27)](https://web.archive.org/web/20220527213026/https://www.mycugc.org/blogs/tobias-kreidl/2019/04/30/a-tale-of-two-servers-part-2)

> *Archived and preserved for the XCP-ng community. Original content © Tobias Kreidl.*

---

## Introduction

The previous installment ([Part 1](./tale-of-two-servers-part-1.md)) covered the foundational BIOS settings that affect CPU frequency scaling and turbo mode — and how those settings dramatically affect GPU performance. Part 2 dives deeper into additional BIOS settings, GPU-specific configuration, and the NVIDIA vGPU scheduler options available on Tesla-class GPUs.

---

## Additional BIOS Settings

### Uncore Frequency

The **Uncore** refers to the parts of the CPU that are not part of the core itself — including the LLC (Last Level Cache), memory controller, and QPI/UPI links. The uncore frequency setting controls how fast these components run.

| Setting | Behavior |
|---|---|
| **Maximum** | Uncore runs at a fixed maximum frequency regardless of core activity |
| **Dynamic** *(Recommended)* | Uncore frequency matches the speed of the fastest active core |

> ✅ **Recommendation: Set Uncore Frequency to "Dynamic"**

"Dynamic" outperforms "Maximum" because it keeps the uncore in sync with the fastest active core rather than running at a fixed frequency that may not match actual core activity. This reduces the bottleneck between the CPU cores and the GPU data path.

---

### C1E State (Enhanced Halt State)

The **C1E** state is an enhanced version of the standard C1 halt state. In addition to stopping the internal clock (like C1), C1E also **reduces the CPU voltage**, saving additional power.

> ✅ **Recommendation: Enable C1E**

Enabling C1E allows idle cores to drop voltage, freeing up power headroom for active cores to boost higher via turbo mode. This is counterintuitive but important — idle cores saving power directly benefits the performance of busy cores.

---

### Collaborative CPU Performance Control

This setting allows the BIOS firmware to collaborate with the OS on CPU performance decisions.

> ⚠️ **Recommendation: Disable Collaborative CPU Performance Control**

When enabled, this can interfere with the OS/hypervisor's ability to make independent CPU frequency decisions — similar to the "System DBPM" issue described in Part 1. Disabling it ensures XCP-ng retains full control.

---

## GPU Overview: NVIDIA Tesla/GRID Line

The NVIDIA Tesla/GRID line of GPUs is specifically designed for virtualization and data center workloads. Unlike consumer GeForce cards, Tesla GPUs support:

- **vGPU (Virtual GPU)** — sharing one physical GPU among multiple VMs
- **ECC memory** — error-correcting memory for reliability
- **No display output** — designed for headless server operation
- **GRID licensing** — required for vGPU functionality

### GPU Comparison: M10, M60, P4, T4

| GPU | Architecture | vGPU Instances | Frame Buffer | Best For |
|---|---|---|---|---|
| M10 | Maxwell | Up to 32 | 32 GB total (4×8GB) | High-density VDI |
| M60 | Maxwell | Up to 16 | 16 GB total (2×8GB) | Balanced VDI + graphics |
| P4 | Pascal | Up to 8 | 8 GB | Power-efficient VDI |
| T4 | Turing | Up to 16 | 16 GB | Modern VDI + AI inference |

> 💡 **Note:** The M60 (used in this series) has **two GPU engines** — meaning two separate GPU instances that can each host VMs independently. This is why the author ran **two XenApp VMs per server**, one per GPU engine.

---

## NVIDIA vGPU Scheduler Modes

On **Pascal and newer** NVIDIA GPUs (P4, P40, V100, T4, etc.), three GPU scheduling modes are available. On older Maxwell GPUs (M10, M60), only Best Effort is available.

### The Three Scheduler Modes

#### 1. Best Effort *(Default — Recommended)*
- Each vGPU gets as much GPU time as it needs
- **Idle vGPU time slots are NOT wasted** — other active vGPUs can use them
- No guaranteed minimum or maximum
- Best for environments where GPU load varies across users

> ✅ **Most experienced admins stick with Best Effort** — it's the only mode where idle GPU cycles are not wasted.

#### 2. Equal Share
- GPU time is divided equally among all active vGPUs
- **Idle vGPU time slots ARE wasted** — they are not redistributed
- Provides fairness but reduces overall GPU utilization efficiency
- Use case: environments where equal treatment of all VMs is required

#### 3. Fixed Share
- Each vGPU is assigned a guaranteed fixed share of GPU time
- **Most idle waste** of the three modes
- Provides QoS guarantees — useful for cloud/service provider billing
- Use case: guaranteed minimum GPU performance per VM/tenant

### Scheduler Mode Comparison

| Mode | Idle Waste | Fairness | QoS Guarantee | Best For |
|---|---|---|---|---|
| Best Effort | ❌ None | Variable | ❌ No | Most environments |
| Equal Share | ⚠️ Moderate | Equal | ❌ No | Equal treatment required |
| Fixed Share | ⚠️ High | Fixed | ✅ Yes | Cloud/service providers |

### Checking and Setting the Scheduler

```bash
# Check current vGPU scheduler mode
nvidia-smi vgpu -s

# Set scheduler mode (0=Best Effort, 1=Equal Share, 2=Fixed Share)
nvidia-smi vgpu -i <gpu-id> -ss 0
```

---

## AMD and Intel GPU Options

While NVIDIA dominates the professional vGPU market, AMD and Intel also offer GPU options supported in XCP-ng:

### AMD
- **AMD MxGPU** (Multiuser GPU) — hardware-based GPU virtualization on FirePro/Instinct cards
- Supported in XenServer/XCP-ng via SR-IOV
- Does not require a separate vGPU manager daemon

### Intel
- **Intel GVT-g** — mediated passthrough for Intel integrated graphics
- Supported on Broadwell and newer Intel Xeon processors with integrated graphics
- Lower performance ceiling than discrete NVIDIA/AMD GPUs but zero additional cost

> 💡 **For most XCP-ng VDI deployments, NVIDIA remains the most mature and well-supported option** — particularly for Citrix/XenApp workloads.

---

## Recommended Reading

The following resources are referenced throughout this series and are highly recommended for anyone managing XCP-ng GPU environments:

- 📖 **Johan van Amersfoort's VDI Design Guide** — Comprehensive guide to VDI architecture, GPU sizing, and XenServer/XCP-ng configuration
- 📖 **Nick Rintalan's vCPU Oversubscription Blog** (Citrix) — The source of the 1.5:1 vCPU rule discussed in Part 3
- 📖 **Frank Denneman's NUMA Deep Dives** — Essential reading for understanding NUMA, cache coherency, and CPU topology
- 🎥 **"Citrix and Login VSI: Scalability Update" Webinar** — Nick Rintalan & Mark Plettenberg (April 2019)

---

## Key Takeaways

- ✅ Set **Uncore Frequency to "Dynamic"** — not "Maximum"
- ✅ **Enable C1E** — idle cores saving power benefits active cores via turbo
- ✅ **Disable Collaborative CPU Performance Control** — let XCP-ng manage CPU performance independently
- ✅ **Best Effort GPU scheduling** is the right choice for most XCP-ng environments
- ✅ **Know your GPU's engine count** — the M60 has two engines; plan VM layout accordingly
- ✅ **Pascal and newer GPUs** offer scheduler mode flexibility; Maxwell GPUs do not
- ✅ Match your **vGPU profile** to your workload — don't over-provision frame buffer

---

*[← Part 1: BIOS Settings](./tale-of-two-servers-part-1.md) | Continued in [Part 3: NUMA, vCPU/Socket Settings & Summary →](./tale-of-two-servers-part-3.md)*
