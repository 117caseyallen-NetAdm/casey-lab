# Diagnosing a Failing SSD — Ten Wrong Answers and One Decisive Test

> **Part of [CASEY-LAB](../README.md)** — a dual-site, multi-vendor enterprise
> homelab. This is a troubleshooting log, kept because the symptoms were generic
> and every early hypothesis was wrong for an interesting reason.

**System:** Proxmox VE on an Apple Mac Pro (MacPro6,1, 2013) — Xeon E5-1650 v2,
31 GB RAM, Apple SSD SM1024F (1 TB)
**Presenting symptom:** "the Windows VM is slow"
**Actual cause:** SSD write-path failure — reads at 367 MB/s, writes at 1.7 MB/s
**Cost of getting there:** four days, two full OS reinstalls

---

## The symptom

I was building a Windows Server domain controller. Everything was slow in ways
that didn't quite add up:

- Server 2025 installed, but the OS was sluggish — Settings, Services, and Task
  Manager slow **to open**
- Task Manager showed **100% disk active time** at **~0 MB/s throughput** with
  **867 ms average response time**
- A rebuild on Server 2022 crawled: **1% complete after 30 minutes**
- On the host, `qm create` failed outright: `/sbin/lvs ... failed: got timeout`

## Method

Change one variable at a time, and prefer tests that split the problem space in
half. Every wrong hypothesis below still earned its keep — each one eliminated a
layer.

One note on metrics before the list, because it matters: **Task Manager's disk
percentage is "active time"** — the fraction of an interval with at least one
I/O outstanding. It is not saturation. A disk doing a trickle of tiny reads
reports 100%. The numbers that mean something are **latency**, **queue depth**,
and **throughput**.

---

## The hypotheses

### 1. The guest OS is too heavy — ❌

Server 2025 with Desktop Experience is genuinely heavier than 2022, so I rebuilt
the entire VM on 2022.

**Eliminated by:** the 2022 install crawled identically.

**Lesson:** software bloat consumes CPU and RAM. It does not make a disk take a
second to answer while transferring no data. The *shape* of the symptom should
have ruled this out before I spent a rebuild on it.

### 2. VM undersized — ⚠️ real problem, not the cause

Started at 2 vCPU / 4 GB; the VM was visibly starved. Moved to 4 vCPU / 8 GB and
responsiveness genuinely improved.

**Eliminated as root cause by:** Resource Monitor → Memory →
**Hard Faults/sec = 4** at 38% memory used, while the disk problem persisted.
Sustained hard faults mean paging — a real memory shortage. Four per second
means memory is fine.

**Lesson:** hard-fault rate is what separates "out of RAM" from "storage is
slow." Disk thrashing *caused by paging* is a memory problem wearing a disk
costume, and neither Task Manager's disk graph nor the event log distinguishes
the two.

### 3. Thermal throttling — ⚠️ real problem, not the cause

SMART reported the SSD at **61 °C idle**, max recorded **70 °C**. Most SSDs
throttle around 70 °C, and a throttling drive behaves exactly like this: stalls,
huge latency, **no errors logged, SMART passes**.

The cause of the heat turned out to be its own finding: **Apple hardware running
Linux often doesn't manage its fans.** The SMC was idling the chassis fan at
**1066 RPM** (floor 790, max 2484) while the drive baked. macOS drives that fan
curve; Debian doesn't.

```bash
lsmod | grep applesmc
cat /sys/devices/platform/applesmc.768/fan1_input    # 1066
echo 2000 > /sys/devices/platform/applesmc.768/fan1_min
```

| Fan | SSD temp | Chassis sensor |
|---|---|---|
| 1066 RPM | 61 °C | 71 °C |
| 1600 RPM | 54 °C | — |
| 2000 RPM | **47 °C** | **65 °C** |

Persisting it, since sysfs resets at boot:

```bash
echo 'w /sys/devices/platform/applesmc.768/fan1_min - - - - 2000' \
  > /etc/tmpfiles.d/applesmc-fan.conf
echo applesmc > /etc/modules-load.d/applesmc.conf
```

**Eliminated as root cause by:** the stall persisted unchanged at 47 °C. Worth
keeping regardless — 61 °C idle is bad independently.

### 4. LVM thin-pool metadata exhaustion — ❌

Thin pools track data and metadata in **separate budgets**, and metadata
exhaustion causes latency collapse while being invisible in `pvesm status`.

```bash
lvs -a    # Data% 0.16, Meta% 0.24
```

Both essentially empty.

### 5. Accumulated snapshots — ❌

Two snapshots existed. On LVM-thin, writing to a block shared with a snapshot
requires allocate-copy-write, which genuinely does add latency.

**Eliminated by:** problem persisted after the VM and both snapshots were gone.

### 6. Disk cache mode and iothread — ❌

The original config used `cache=none` (O_DIRECT — the guest gets the host's raw
latency with nothing absorbing it). Rebuilt with `cache=writeback` and
`iothread=1` on `virtio-scsi-single`.

**Eliminated by:** no improvement — and a host-side `dd` bypassing the VM
entirely was just as slow. No guest setting can explain that.

### 7. The drive is failing (per SMART) — ❌, and this one is instructive

```
SMART overall-health self-assessment: PASSED
Reallocated_Sector_Ct       0
Current_Pending_Sector      0
UDMA_CRC_Error_Count        0
Raw_Read_Error_Rate         0
Power_On_Hours          43105      (~4.9 years)
Host_Writes_MiB      29310305      (~28 TB)
```

Every attribute that indicates a *dying* SSD is clean.

**Lesson, and it's the big one:** SMART thresholds are calibrated for **imminent
total failure**, not for "this drive has become unusable for its job." A drive
can pass SMART comprehensively and still be functionally dead. Measure
performance; don't trust a pass/fail flag.

### 8. SATA link power management — ❌

Aggressive link power saving is a classic cause of unexplained SATA stalls.

```bash
cat /sys/class/scsi_host/host*/link_power_management_policy   # max_performance
```

Already at maximum.

### 9. NCQ — ❌

Some drive/controller combinations hang under queued I/O.

```bash
cat /sys/block/sda/device/queue_depth   # 32
echo 1 > /sys/block/sda/device/queue_depth
```

Writes measured 2.2 and 1.6 MB/s with queuing disabled. No change.

### 10. TRIM never running — ❌ (good hypothesis, wrong answer)

28 TB written to a 1 TB drive is 28 full-drive overwrites. Without TRIM, the
flash translation layer has no idea which blocks are free, every write becomes
read-modify-write against dirty erase blocks, and write performance collapses
**exactly like this**.

```bash
lsblk -D                       # DISC-GRAN 4K, DISC-MAX 2G — discard supported
systemctl status fstrim.timer  # enabled, weekly, active
fstrim -v /                    # 10.7 GiB trimmed
```

TRIM was configured and running. Only 10.7 GiB to reclaim — had it never run,
that number would be in the hundreds of gigabytes.

---

## The decisive test

With every software layer eliminated, one measurement split what remained —
**compare read speed against write speed on the same device:**

```bash
hdparm -t /dev/sda
# Timing buffered disk reads: 1104 MB in 3.00 seconds = 367.74 MB/sec

dd if=/dev/zero of=/var/lib/vz/dd-test bs=1M count=2000 oflag=direct conv=fdatasync
# 508 MB copied, 292.602 s, 1.7 MB/s
```

**Reads: 367 MB/s. Writes: 1.7 MB/s. A ~200× asymmetry.**

This is conclusive. Nothing in the SATA link, the AHCI driver, the kernel block
layer, LVM, or the filesystem can be fast in one direction and catastrophically
slow in the other — every one of those is direction-agnostic. **The failure is
inside the drive, in its write path specifically.**

The flash still reads perfectly, which is why the host boots normally and light
operations feel fine. Only writes expose the controller's degraded state.

### Why those `dd` flags

The result is meaningless without them:

- `if=/dev/zero` — a costless data source, so we measure writes, not reads
- `bs=1M count=2000` — 2 GB, large enough that caching can't fake it
- `oflag=direct` — O_DIRECT, bypassing the page cache to reach the actual device
- `conv=fdatasync` — forces a flush to physical media before reporting, so the
  number isn't a buffered lie

Drop the last two and you're benchmarking RAM.

---

## Root cause

The SSD's **write path has failed** after 43,105 power-on hours and ~28 TB
written. Reads are unaffected. No SMART attribute measures "writes became 200×
slower," so nothing flagged it.

### Follow-up: a full power cycle didn't help — and exposed one more lesson

Cold power-off, on the theory that the controller might re-initialize out of a
bad state. It didn't. But the read results were instructive:

```
Temperature:  43 °C  (coolest yet — thermal definitively excluded)
Write:        1.9 MB/s  (was 1.7 — unchanged)

Read, three consecutive runs of hdparm -t:
              1.52 MB/s
           1457.34 MB/s
            657.28 MB/s
```

**SATA 3.0 is 6 Gb/s — roughly 600 MB/s of real throughput after encoding
overhead. 1457 MB/s and 657 MB/s are physically impossible over this link.**
Those are page-cache reads, not disk reads. The only honest number is the cold
read: **1.52 MB/s**.

So the read path is degrading too, and the wild variance between identical
back-to-back runs is itself a failure signature — healthy storage returns
consistent numbers.

**Lesson:** sanity-check every benchmark against the physical ceiling of the
interface it crossed. A result that exceeds what the bus can carry is measuring
something other than what you think — usually cache. Taken at face value, two of
those three numbers would have led to exactly the wrong conclusion.

Resolution is storage replacement. Worth noting for anyone with the same
machine: the SM1024F is Apple's proprietary blade format, **not M.2** — so the
options are an OEM pull, a third-party adapter with inconsistent MacPro6,1
compatibility, or relocating VM storage to external USB 3 / Thunderbolt 2.

---

## Transferable lessons

1. **The shape of a failure localizes it before any diagnostic runs.** High I/O
   wait with *zero* throughput indicates a stall, not a bandwidth limit. Read the
   structure of a symptom before reaching for tools.
2. **"100% utilization" usually isn't.** Active time is not saturation. Latency
   and queue depth are the real metrics.
3. **SMART passing ≠ healthy drive.** Its thresholds detect imminent death, not
   degradation.
4. **When a batch operation fails, reduce it to one variable.** A GUI reporting
   "feature installation failed" hid that one Features-on-Demand component had
   failed while the role I actually wanted was fine. Installing that single role
   from the command line succeeded instantly.
5. **Fixing something real doesn't mean fixing the right thing.** The RAM bump
   and the fan fix were both correct and both improved the system. Neither was
   the root cause. Resist declaring victory when a change helps.
6. **Prove hardware last, and prove it properly.** It's the hardest conclusion to
   reach honestly, because it requires convincing yourself that every software
   layer above it is innocent first.

## Side finding: Apple hardware under Linux has no fan control

Worth its own mention, because almost nobody documents it. The `applesmc` driver
exposes fan management through sysfs, but nothing drives a curve by default —
the SMC idles the fan and the machine cooks. On a MacPro6,1 running Proxmox,
raising `fan1_min` dropped SSD temperature by 14 °C. If you're repurposing Mac
hardware as a server, check this on day one.
