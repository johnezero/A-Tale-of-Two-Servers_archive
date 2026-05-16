# A Tale of Two Servers, Part 1
## How BIOS Settings Affect Apps and GPU Performance

**Author:** Tobias Kreidl (@tjkreidl)
**Originally Published:** March 7, 2019 on CUGC (Citrix User Group Community)
**Archived Source:** [Wayback Machine Snapshot (2022-05-27)](https://web.archive.org/web/20220527221535/https://www.mycugc.org/blogs/tobias-kreidl/2019/03/07/tale-of-two-servers-bios-settings-affect-apps-gpu)

> *Archived and preserved for the XCP-ng community. Original content © Tobias Kreidl.*

---

## Introduction

Servers can only do their job well if they are optimized and tuned to perform their duties that best suit the applications they run and fulfill the expectations of those who use them. This short series of articles strives to address in particular how the increasingly more common combination of servers and embedded GPUs deserves special attention to bring out the best in both.

The servers in question are two Dell PowerEdge rack servers — an **R720** and an **R730** — both running **XenServer** with **NVIDIA GRID/Tesla GPUs**. Despite being similar in many ways, their performance differed significantly until proper tuning was applied.

---

## The Root Cause

The root cause of the performance difference came down to **BIOS settings** — specifically how the CPU power management was configured. On the Dell R730, the default BIOS setting for CPU Power Management was **"System DBPM"**, which hands control of CPU frequency scaling to the system firmware rather than the OS or hypervisor.

> ⚠️ **Problem:** When set to "System DBPM", XenServer/XCP-ng cannot control CPU frequency scaling. The CPU may run at reduced frequencies even under heavy load, directly impacting GPU performance since the CPU feeds data to the GPU.

---

## The Fix: OS DBPM

Changing the setting to **"OS DBPM"** hands frequency control back to the OS/hypervisor, allowing XenServer/XCP-ng to manage CPU performance states properly.

> ✅ **Solution:** Set CPU Power Management to **"OS DBPM"** in the Dell BIOS.

---

## Turbo Mode

With OS DBPM enabled, turbo mode can be properly leveraged. Turbo mode allows the CPU to temporarily exceed its base clock speed when thermal and power conditions allow.

### Impact of Turbo Mode Misconfiguration

| Configuration | Max CPU Frequency | GPU Benchmark (1080p Medium) |
|---|---|---|
| Default (System DBPM, no turbo) | 2.40 GHz | Baseline |
| Turbo misconfigured | ~2.40 GHz | ~48% below optimized |
| Fully optimized (OS DBPM + turbo) | 3.145 GHz (+31%) | +36% above baseline |

---

## XenServer / XCP-ng CPU Governor

Even with BIOS set correctly, the XenServer/XCP-ng CPU frequency governor must also be set to **"performance"** to ensure the hypervisor requests maximum CPU frequency.

### Essential Commands

```bash
# Check current CPU frequency states
xenpm get-cpufreq-states

# Check current idle states
xenpm get-cpuidle-states

# Set governor to performance (run on each host)
xenpm set-scaling-governor performance

# Make persistent across reboots
/opt/xensource/libexec/xen-cmdline --set-xen cpufreq=xen:performance

# Verify turbo mode is active
xenpm get-cpufreq-para | grep turbo

# Monitor average CPU frequency during load
watch -n 5 'xenpm start 1 | grep "Avg freq" | sort | tail -16'
```

> ⚠️ The `xenpm set-scaling-governor performance` command must be run on **each XCP-ng host individually** and does not persist across reboots unless the persistent command above is also applied.

---

## BIOS Settings: Custom Power Profile (Dell R730)

To expose all relevant settings, the **System Profile must be set to "Custom"** in the Dell BIOS. The standard "Performance" profile does not expose individual settings like C-states.

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
| Energy Efficient Policy | **Performance** |
| Number of Turbo Boot Enabled Cores | **All** |
| Monitor/Mwait | **Enabled** |

> 💡 **Why enable C-states?** Enabling C-states allows cores that are not busy to idle more efficiently, freeing up power headroom for active cores to boost higher via turbo. This is counterintuitive but important.

> ⚠️ **Caveat:** Pushing turbo to the maximum cannot typically be sustained for long periods. If **end-user experience (EUX) consistency** is the priority over peak performance, turbo mode might not be the best setting for your environment.

---

## Node Interleaving

**Node Interleaving** should be **Disabled**. When enabled, it distributes memory across NUMA nodes in a round-robin fashion, which sounds balanced but actually prevents the OS and hypervisor from making intelligent NUMA-aware memory placement decisions.

> ✅ **Recommendation: Disable Node Interleaving** — let the OS/hypervisor manage NUMA placement.

---

## Results Summary

Through BIOS tuning and XCP-ng governor configuration alone (no hardware changes):

| Metric | Before | After | Change |
|---|---|---|---|
| Max CPU Frequency | 2.40 GHz | 3.145 GHz | **+31%** |
| GPU Benchmark (1080p Medium) | Baseline | +36% | **+36%** |
| GPU Benchmark (1080p Extreme) | Baseline | +~2% | **+2%** |
| GPU Utilization (1080p Medium) | ~82% | ~97% | **+15 pp** |

---

## Key Takeaways

- ✅ Set **CPU Power Management to "OS DBPM"** — not "System DBPM"
- ✅ Use the **Custom System Profile** to expose all BIOS settings
- ✅ Enable **C-states** to maximize turbo mode effectiveness
- ✅ Set **Node Interleaving to Disabled**
- ✅ Run `xenpm set-scaling-governor performance` on each XCP-ng host
- ✅ Make the governor setting **persistent across reboots**
- ✅ Change **one parameter at a time** when testing — isolate variables for meaningful results
- ✅ Test with **realistic workloads** and repeat runs for consistency

---

*Continued in [Part 2: GPU Settings & Advanced BIOS Tuning](./tale-of-two-servers-part-2.md)*
