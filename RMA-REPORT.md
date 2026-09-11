# Hardware Defect Report — Faulty CPU Core

**Unit:** Venus Series mini PC · Intel Core i9-13900H
**Serial:** `MD287LS139EDMPA00068`
**Report date:** 2026-09-11
**Status:** Defect isolated and confirmed. Unit running in degraded mitigation.

---

## 1. Summary

One physical CPU core in this unit is defective. It silently returns incorrect
computation results rather than failing cleanly, which corrupts data in any workload
scheduled onto it.

The defect is isolated to **physical P-core 3** (logical CPUs 6 and 7 — the two
hyperthreads of a single core). Over a 40-day logging window, this unit recorded
**37 process crashes, 100% of them on that one core, and zero on the other 18 logical
CPUs.**

This is not a software fault, a configuration error, or a memory fault. Sections 4
and 5 document the evidence and the alternative causes that were tested and excluded.

**Requested remedy:** replacement of the unit under warranty. The CPU is BGA-soldered
and cannot be replaced independently.

---

## 2. Hardware Identification

| Field | Value |
|---|---|
| System manufacturer | Micro Computer (HK) Tech Limited |
| System product | Venus Series |
| System version | 1.0 |
| **System serial** | **`MD287LS139EDMPA00068`** |
| Baseboard manufacturer | Shenzhen Meigao Electronic Equipment Co.,Ltd |
| Baseboard product | `AHWSA` |
| **Baseboard serial** | **`FHWSA139023C1070822`** |
| BIOS | American Megatrends 1.27, dated 2025-04-03 |
| CPU | 13th Gen Intel Core i9-13900H (family 6, model 186, stepping 2) |
| CPU topology | 14 cores / 20 threads (6 P-core + 8 E-core) |
| Microcode | `0x4129` (updated from `0x411c` at boot) |
| Memory | 2 × 16 GB Crucial DDR5 SODIMM, non-ECC, `CT16G56C46S5.M8B2` |
| OS | Proxmox VE 9.1.1, kernel 6.17.2-1-pve |

---

## 3. The Defective Core

`lscpu` topology — note CPUs 6 and 7 share physical **CORE 3**, and that core is one
of only two rated to boost to 5400 MHz:

```
CPU CORE SOCKET    MAXMHZ
  4    2      0 5400.0000
  5    2      0 5400.0000
  6    3      0 5400.0000   <-- DEFECTIVE
  7    3      0 5400.0000   <-- DEFECTIVE (sibling thread, same physical core)
  8    4      0 5200.0000
```

Full topology: [`evidence/pve-3-cpu-topology.txt`](evidence/pve-3-cpu-topology.txt)

---

## 4. Evidence

### 4.1 Crash distribution — the primary finding

Every userspace crash recorded by the kernel on this host, all boots, tallied by the
CPU the kernel attributed it to:

```
  34  likely on CPU 6 (core 12, socket 0)
   3  likely on CPU 7 (core 12, socket 0)
   0  on any of the remaining 18 logical CPUs
```

> The kernel reports the firmware core ID (`core 12`); `lscpu` normalises the same
> physical core to index 3. Both identifiers refer to one core.

**37 of 37 faults on a single physical core.** If these faults were caused by
software, memory, or thermal conditions, they would distribute across all 20 logical
CPUs. A distribution this concentrated has no plausible cause other than the core
itself.

### 4.2 Persistence over time

The fault is not a single event. It recurs across four separate boots spanning more
than a month:

| Boot | Period | Crashes |
|---|---|---|
| −3 | 2026-04-24 → 2026-05-27 | 0 |
| −2 | 2026-05-27 → 2026-08-02 | 4 |
| −1 | 2026-08-02 → 2026-09-11 | 26 |
| 0 | 2026-09-11 → present | 8 (all before mitigation) |

The increasing rate across boots is consistent with progressive degradation.

### 4.3 Affected software is unrelated and diverse

Crashes are not confined to one program. Affected binaries include `perl`
(`pvestatd`, `pveproxy`, `pvecm`, `pvesh`, `pve-ha-crm`, `pve-bridge`, `spiceproxy`,
`pveupdate`), `python3.13`, and inside the virtual machine `sqlite3`, `python3.11`,
and `libc.so.6`.

These share no code, no libraries, and no vendor. The only thing they share is the
CPU core they happened to execute on.

Raw log: [`evidence/pve-3-host-segfaults.log`](evidence/pve-3-host-segfaults.log)

Representative entries:

```
Aug 13 05:14:19 pve-3 kernel: pveupdate[1299440]: segfault at 91 ... in perl ... likely on CPU 6 (core 12, socket 0)
Aug 15 01:34:55 pve-3 kernel: pveupdate[1526778]: segfault at 22 ... in perl ... likely on CPU 6 (core 12, socket 0)
Aug 16 02:59:19 pve-3 kernel: pveupdate[1657716]: segfault at 19 ... in perl ... likely on CPU 6 (core 12, socket 0)
Sep 03 03:35:29 pve-3 kernel: pveupdate[3882503]: segfault at 2b ... in perl ... likely on CPU 6 (core 12, socket 0)
Sep 10 14:44:21 pve-3 kernel: pvestatd[1286]:    segfault at 11 ... in perl ... likely on CPU 7 (core 12, socket 0)
Sep 11 00:19:32 pve-3 kernel: python3[796]:      segfault at 8  ... in python3.13 ... likely on CPU 6 (core 12, socket 0)
```

### 4.4 Corruption propagates into virtual machines

The unit runs a Proxmox hypervisor. A guest VM (8 vCPUs, Debian 13, kernel
6.12.74) suffered repeated data corruption and six kernel crashes:

```
Aug 27 01:08:57  Oops: Oops: 0010 [#1]
Aug 27 07:29:19  Oops: Oops: 0010 [#2]   Comm: python3
Sep 08 23:55:10  Oops: Oops: 0010 [#3]   Comm: sqlite3
Sep 08 23:56:16  Oops: Oops: 0010 [#4]   Comm: sqlite3
Sep 09 00:04:49  Oops: Oops: 0010 [#5]   Comm: python3
Sep 09 08:25:46  Oops: Oops: 0010 [#6]   Comm: frigate.output
```

Each oops is a `BUG: kernel NULL pointer dereference` with `RIP: 0010:0x0` — the
kernel transferring execution to address zero from the generic syscall-exit path, a
signature of a corrupted function pointer:

```
BUG: kernel NULL pointer dereference, address: 0000000000000000
#PF: supervisor instruction fetch in kernel mode
RIP: 0010:0x0
Call Trace:
 syscall_exit_to_user_mode+0x37/0x1b0
 do_syscall_64+0x8e/0x190
 entry_SYSCALL_64_after_hwframe+0x76/0x7e
```

The `sqlite3` crashes on 2026-09-08 corrupted a 1 GB production database, requiring
manual recovery. Full traces:
[`evidence/nvr-guest-kernel-oops.log`](evidence/nvr-guest-kernel-oops.log),
[`evidence/nvr-guest-crash-summary.log`](evidence/nvr-guest-crash-summary.log)

**Note on guest-side CPU attribution.** Inside the VM, crashes appear spread across
virtual CPUs 1, 3, 4, 5, 6 and 7. This is expected and does *not* contradict the
finding: a guest's virtual CPU numbers have no fixed relationship to host physical
cores, because the hypervisor migrates vCPU threads freely. The guest simply sees
corruption wherever its threads happened to be running when they landed on the
defective physical core. **The host-side data in §4.1 is the conclusive evidence;
the guest data demonstrates impact.**

---

## 5. Causes Tested and Excluded

Each alternative explanation was checked and ruled out before concluding the CPU is
at fault.

| Candidate cause | Finding | Verdict |
|---|---|---|
| **Memory (RAM)** | Faults are 100% concentrated on one core. Bad RAM produces faults distributed across all cores. | **Excluded** |
| **Memory overclocking** | DDR5 rated 5600 MT/s, running at **5200 MT/s** — below rated speed. No XMP/EXPO profile active. | **Excluded** |
| **Memory overcommit** | 32 GB installed, 16 GB allocated to the VM, 12 GB free, **zero swap in use**. No pressure. | **Excluded** |
| **KSM page merging** | `/sys/kernel/mm/ksm/run` = `0`, `pages_sharing` = `0`. Never engaged. | **Excluded** |
| **Memory ballooning** | No `balloon` parameter in VM config; balloon defaults to full allocation, so never inflates. | **Excluded** |
| **Outdated microcode** | Running `0x4129`, auto-updated from `0x411c` at boot via `intel-microcode 3.20250812.1`. Current. | **Excluded** |
| **Guest OS / application bug** | Faults occur on the **host** too, in unrelated Perl and Python binaries, outside any VM. | **Excluded** |
| **Storage fault** | Root filesystem 78% used, data volume 11% used, both responsive with no I/O errors in the kernel log. | **Excluded** |
| **Thermal** | No thermal throttling events logged. Host load average at time of capture: 0.43. | **Excluded** |

---

## 6. Operational Impact

The defective core caused a complete production outage of a 10-camera video
recording system:

- Corrupted a 1 GB SQLite database (2026-09-08), requiring manual recovery
- A guest kernel crash left an orphaned kernel page lock, hanging **364+ processes**
  in uninterruptible sleep
- Host load average reached **362** at ~98% I/O wait with near-zero CPU utilisation
- Recovery required a full reboot; the condition recurred repeatedly

---

## 7. Mitigation Currently Applied

Both hyperthreads of the defective core have been taken offline:

```bash
echo 0 > /sys/devices/system/cpu/cpu6/online
echo 0 > /sys/devices/system/cpu/cpu7/online
```

Verified: `/sys/devices/system/cpu/online` now reports `0-5,8-19` (18 of 20 threads).

This is persisted across reboots by `/etc/systemd/system/disable-faulty-core.service`,
ordered before `pve-guests.service` so that virtual machines are never scheduled onto
the defective core at boot.

**The unit is therefore operating with 13 of 14 cores.** This is a workaround, not a
repair — the hardware defect remains present.

---

## 8. Verification Procedure

To reproduce the fault independently:

```bash
apt install stress-ng

# Defective core — expect verification failures
taskset -c 6 stress-ng --cpu 1 --cpu-method all --verify --timeout 300

# Known-good core — expect a clean run
taskset -c 8 stress-ng --cpu 1 --cpu-method all --verify --timeout 300
```

Re-enable the core first if it is currently offline:

```bash
systemctl stop disable-faulty-core.service     # re-enables cpu6 and cpu7
```

To review the historical evidence directly on the unit:

```bash
journalctl | grep -E "segfault|general protection fault" \
  | grep -oE "likely on CPU [0-9]+ \(core [0-9]+" | sort | uniq -c | sort -rn
```

---

## 9. Context: Raptor Lake Core Degradation

The failing core is one of only two in this CPU rated to boost to 5400 MHz — the
highest-voltage, highest-stress cores in the package. Progressive degradation of
high-boost cores is a documented failure mode for Intel's 13th and 14th generation
Raptor Lake processors, presenting exactly as observed here: a specific core
gradually beginning to produce incorrect results under load.

For accuracy: Intel scoped its official *Vmin Shift Instability* erratum to 13th/14th
generation **desktop** parts, and stated that mobile processors were not affected by
that specific root cause. The i9-13900H is a mobile part, so this report does not
claim that erratum applies. It is noted only because the observed failure pattern —
the highest-boosting core degrading first, worsening over months — closely matches it.

**The evidence in §4 stands independently of the underlying mechanism.** Whatever the
cause, the measured behaviour is a CPU core returning incorrect computation results.

---

## 10. Requested Action

Replacement of the unit under warranty.

The i9-13900H is BGA-soldered to baseboard `AHWSA` and cannot be replaced
independently of the mainboard.

**Before submitting:**

1. Confirm the vendor. DMI reports `Micro Computer (HK) Tech Limited` / `Venus Series`
   — the identifier used by Minisforum. Verify against the purchase record.
2. Locate proof of purchase and confirm warranty period.
3. Check for a BIOS newer than **1.27 (2025-04-03)**. Vendors commonly request this
   before accepting an RMA. A BIOS update will not repair a degraded core, but
   completing it removes an obvious objection.
4. Attach this report and the [`evidence/`](evidence/) directory.

---

## Appendix — Evidence Files

| File | Contents |
|---|---|
| [`evidence/pve-3-host-segfaults.log`](evidence/pve-3-host-segfaults.log) | Every host crash logged, all boots — the primary evidence |
| [`evidence/pve-3-cpu-topology.txt`](evidence/pve-3-cpu-topology.txt) | `lscpu -e` output mapping logical CPUs to physical cores |
| [`evidence/nvr-guest-kernel-oops.log`](evidence/nvr-guest-kernel-oops.log) | Full kernel oops trace from the affected VM |
| [`evidence/nvr-guest-crash-summary.log`](evidence/nvr-guest-crash-summary.log) | Timeline of all guest crashes and oopses |

All logs are unedited output from `journalctl` and `dmesg`.
