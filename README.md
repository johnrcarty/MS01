# Frigate NVR — Incident Record

Diagnosis of a recurring Frigate NVR outage, traced to a defective CPU core on the
Proxmox host.

## Contents

| Path | Purpose |
|---|---|
| [`RMA-REPORT.md`](RMA-REPORT.md) | **Start here.** Full hardware defect report, written for a warranty claim. |
| [`evidence/`](evidence/) | Unedited kernel and journal logs supporting the report. |

## Outcome

Frigate had gone down repeatedly, most visibly with a corrupted SQLite database on
2026-09-08. Repairing the database did not stop the recurrence, because the database
was a symptom rather than the cause.

The actual fault was **one defective physical CPU core** on Proxmox node `pve-3`
(Intel i9-13900H, physical P-core 3 = logical CPUs 6 and 7). It returned incorrect
computation results instead of failing cleanly, corrupting whatever ran on it.

The evidence: **37 process crashes over 40 days, 34 on CPU 6 and 3 on CPU 7, and zero
on the other 18 logical CPUs.** CPUs 6 and 7 are the two hyperthreads of one physical
core.

Because the Frigate VM's vCPU threads were scheduled across all host cores, any work
that landed on the defective core came back wrong — producing `sqlite3` crashes (the
database corruption), `python3` crashes, and six guest kernel oopses.

RAM, memory overclocking, KSM, ballooning, microcode, storage, and thermal causes were
each tested and excluded. See §5 of the report.

## Mitigation in place

Both threads of the defective core are offline, persisted via
`/etc/systemd/system/disable-faulty-core.service` on `pve-3`, ordered before
`pve-guests.service` so VMs are never scheduled onto it.

```bash
cat /sys/devices/system/cpu/online     # 0-5,8-19  (18 of 20 threads)
```

The host runs on 13 of 14 cores. This is a workaround; the defect remains until the
unit is replaced.

## Separately: three cameras offline

Unrelated to the CPU fault, three cameras are unreachable at the network level and
need a physical check of power/PoE:

| Camera | Address | State |
|---|---|---|
| Kitchen | 192.168.1.52 | ARP `FAILED` |
| LitterBox | 192.168.1.55 | ARP `INCOMPLETE` |
| Hyperbaric | 192.168.4.51 | unreachable |

The other seven cameras stream normally at ~5 FPS.

> **Diagnostic note:** go2rtc logs `method DESCRIBE failed: 404 Not Found` when its
> *upstream* dial fails. The 404 is misleading — the real error is the accompanying
> `no route to host` / `i/o timeout` line.
