---
layout: post
title: "A 57-Minute macOS Boot on Proxmox: Tracing a Radeon GPU Passthrough Stall to XNU's Video Console"
date: 2026-10-25
categories: [homelab, virtualization, macos]
tags: [proxmox, hackintosh, opencore, qemu, kvm, amd, gpu-passthrough, xnu, debugging]
description: "Upgrading a stable RX 580 passthrough Hackintosh to a Radeon PRO W6600 exposed a 57-minute boot delay. QEMU CPU profiling and XNU symbols revealed the cause."
---

# A 57-Minute macOS Boot on Proxmox: Tracing a Radeon GPU Passthrough Stall to XNU's Video Console

**TL;DR:** A macOS Sequoia VM on Proxmox took about **57 minutes** to become usable when its AMD Radeon PRO W6600 was the only graphics adapter. PCI enumeration itself completed in about **0.34 seconds**. A single QEMU vCPU thread ran at **100% CPU** during the stall. By inspecting its registers, dumping guest instructions, and matching the bytes against an Apple Kernel Debug Kit, I identified the execution hotspot as **`vc_paint_char_32()`**, part of XNU's built-in video-console renderer. **Removing the `-v` verbose-boot flag from OpenCore's `boot-args` eliminated the pathological startup delay in my setup, while retaining physical DisplayPort output and GPU acceleration.**

> **Scope and caution:** This is a record of one particular VM and a reproducible diagnostic method, not a universal cure for slow macOS boots. The instruction-to-symbol match was made against a *nearby*, not identical, kernel build; the decisive confirmation was the controlled boot after removing `-v`. Only make changes to a backed-up EFI, and do not copy someone else's serial numbers, ROM, SMC OSK, or machine identifiers.

## Preface: upgrading a GPU that wasn't broken

This story started with a choice, not a failure. My macOS VM was already working well on Proxmox with an **AMD Radeon RX 580 passed through directly to the guest**. The setup was stable, the physical display output worked, and there was no urgent problem to solve. From a purely practical standpoint, I could have left it alone.

But I wanted to move beyond the RX 580 and run a **professional workstation GPU: the AMD Radeon PRO W6600**. It felt like a worthwhile step for the machine I wanted to build, even though it meant replacing a proven component with a different class of graphics card. I expected to spend some time checking compatibility and passthrough settings. I did *not* expect a change of GPU to turn an otherwise usable macOS VM into a machine that appeared to hang for almost an hour on every cold boot.

![Sapphire Nitro+ Radeon RX 580 in its packaging—the passthrough GPU that already worked, and the card I chose to retire anyway.](/assets/imgs/amd_rx580.jpg)

![AMD Radeon PRO W6600 on its retail box—the single-slot workstation card that was supposed to be the easy part of the upgrade.](/assets/imgs/amd_pro_w6600.jpg)

That is the peculiar challenge of a virtualized Hackintosh: a GPU upgrade is not just a hardware swap. The host firmware and IOMMU, Proxmox/QEMU's virtual hardware, OpenCore, macOS graphics drivers, and the display attached to the passed-through card all interact. When something goes wrong, the symptom can point toward the wrong layer. In my case, the boot screen seemed to implicate PCI configuration and Radeon initialization. Both were convincing distractions.

I spent multiple rounds comparing **GPU-only passthrough against an emulated VMware SVGA display**, collecting macOS logs, reverting OpenCore versions, inspecting framebuffer properties, and testing an SMC change. The breakthrough came only when I stopped treating the last visible boot message as the location of the problem and inspected **what the guest CPU was actually executing**.

This post is the full investigation—including the dead ends—because the method is more broadly useful than the eventual one-line fix. It ends with a **working Radeon PRO W6600, physical displays, graphics acceleration, and fast booting**, without giving up the dedicated-GPU setup that motivated the upgrade.

## The machine

The GPU upgrade took place in my existing Proxmox homelab, built on an **ASUS Pro WS C422-ACE** with an **Intel Xeon W-2195**. The affected VM was configured with **16 vCPUs**, **64 GiB RAM**, **Q35/OVMF**, and macOS **Sequoia 15.7.2, build 24G325**. The graphics card was an **AMD Radeon PRO W6600 (Navi 23, 8 GiB)** passed directly through to the guest; an ASMedia USB controller was also passed through.

![Inside the Proxmox tower: the W6600 in the lower slot, an RTX 4000 above it, and enough spinning rust to keep a homelab busy.](/assets/imgs/proxmox_host.jpg)

At the time of the successful baseline test, the relevant Proxmox configuration looked like this (with storage identifiers and any secrets omitted):

```ini
vga: none
hostpci0: 0000:67:00,pcie=1,romfile=w6600-native.rom,x-vga=1
hostpci1: 0000:05:00.0
boot: order=virtio6;virtio0
```

The guest booted through an OpenCore EFI on a small VirtIO disk. The earlier known-good bootloader was **OpenCore 0.9.5** with **RELEASE** Lilu 1.7.2, WhateverGreen 1.7.1, and VirtualSMC 1.3.7. The VM was configured as `iMacPro1,1`. Following the fix, I also upgraded OpenCore, Lilu, and WhateverGreen to newer **RELEASE** binaries and confirmed that the VM continued to work.

## The symptom: "PCI configuration begin" was a red herring

With `vga: none` and the W6600 selected as the primary passed-through GPU, the VM appeared stuck for roughly an hour. The physical monitors stayed black, even though the VM eventually reached macOS and, on some boots, both displays and Metal acceleration worked.

This looked like a classic PCI passthrough or graphics-initialization failure. The first attempts focused on PCI/BIOS and OpenCore graphics settings: Above 4G Decoding, Resize BAR, GPU ROM presentation, `x-vga=1`, the `agdpmod=pikera` boot argument, AGDC messages, and framebuffer connector properties.

![Verbose boot on the physical display, parked at PCI configuration—the screen that convinced me the stall was in enumeration, until the logs said otherwise.](/assets/imgs/boot_wait.jpg)

But the boot logs showed:

```text
22:19:26.185  (IOPCIFamily) [ PCI configuration begin ]
22:19:26.526  (IOPCIFamily) [ PCI configuration end, bridges 10, devices 21 ]
```

**PCI enumeration lasted approximately 0.34 seconds.** The message visible on the boot screen was not evidence that the guest spent 57 minutes enumerating PCI devices.

## First useful comparison: VMware SVGA versus real GPU output

The fastest way to get diagnostic access was to enable an emulated VMware SVGA display temporarily:

- **VMware SVGA enabled:** the guest became usable in under a minute, and the W6600 appeared as a PCI graphics device, but its physical DisplayPort monitors were black.
- **VMware SVGA disabled (`vga: none`):** the guest took roughly 57 minutes, but the original OpenCore configuration eventually produced real DisplayPort output and acceleration.

This was a useful controlled contrast, although other OpenCore versions had also been tested during the broader troubleshooting effort.

Working IORegistry captures showed two active W6600 DisplayPort links (including a Dell P2419H and a Samsung display), with real display timing and EDID information. The fast VMware capture had Radeon framebuffer objects but no equivalent active physical connections. Importantly, the AMD `aty_config` and `aty_properties` dictionaries were the same across captures. Blindly injecting more of the same framebuffer properties was unlikely to help.

## The misleading logs

During slow boots, macOS emitted many genuine warnings:

```text
registry root held busy, kernelmanagerd has not checked in
GPUWranglerWaitForPublish: timed out; status=0xe00002d6
IOFB?: newUserClient waitQuietController failed: 0xe00002d6
```

There were also long periods of missing activity in `cfprefsd`, `lsd`, and `sandboxd`; an `fskitd` process was repeatedly launched around every ten seconds for about 34 minutes. At various points we also found `X86PlatformPlugin` power-state warnings, HighPoint kext-loading errors, and many AppleSMC bad-command messages.

These were **clues**, but none established the root cause. In fact, some AGDC and framebuffer warnings occurred transiently even during the *successful* GPU-only boot. Treating every alarming log line as causal would have sent the investigation in circles.

### A failed SMC experiment

The running VM originally exposed QEMU's `isa-applesmc` **and** loaded VirtualSMC. In the working configuration, Apple's SMC service reported `IOMode = PMIO`. We tested removing `isa-applesmc` while leaving VirtualSMC enabled; the guest then reported `IOMode = MMIO`.

That test was illuminating but unsuccessful:

| Observation | With QEMU AppleSMC | Without QEMU AppleSMC |
|---|---|---|
| Time until SSH | About 57 minutes | 57 minutes 40 seconds |
| Physical W6600 displays | Worked in the known-good boot | Remained unavailable in the test |
| macOS AppleSMC mode | PMIO | MMIO |

This showed that the SMC path changed, **but it did not fix the startup delay**. We restored the original QEMU configuration. A single failed graphics boot is not enough to conclude that removing AppleSMC always breaks physical output; it simply was not a useful fix here.

## The turning point: profile QEMU during the stall

Rather than gathering more logs *after* an hour-long boot, I inspected the running QEMU process from Proxmox while macOS was stalled.

The host was healthy: approximately **164 GiB available RAM**, no swap, no memory pressure, and no significant I/O wait. QEMU mapped roughly the VM's allocated 64 GiB; that alone did not imply the guest had exhausted memory.

A direct `/proc` thread listing showed that 15 guest vCPUs were sleeping in `kvm_vcpu_block`, while **`CPU 1/KVM`** was runnable. A five-second scheduler-runtime measurement confirmed it:

```text
TID          THREAD                    CPU %
1177976      CPU 1/KVM                100.0%
1177936      kvm                        0.6%
1177975      CPU 0/KVM                  0.5%
...          ...                       ~0.0%
```

So this wasn't a host-memory problem or a completely idle guest waiting for a timer. **One guest CPU was actually burning an entire core.**

Here is a reusable *read-only* Proxmox-side sampling script:

```python
import time
from pathlib import Path

pid = Path('/var/run/qemu-server/500.pid').read_text().strip()
taskdir = Path(f'/proc/{pid}/task')

def sample():
    result = {}
    for task in taskdir.iterdir():
        try:
            name = (task / 'comm').read_text().strip()
            runtime_ns = int((task / 'schedstat').read_text().split()[0])
            result[task.name] = (name, runtime_ns)
        except (OSError, ValueError):
            pass
    return result

before = sample()
time.sleep(5)
after = sample()
for tid, (name, runtime_ns) in after.items():
    if tid in before:
        pct = (runtime_ns - before[tid][1]) / 5e9 * 100
        if pct > 1:
            print(f'{tid}: {name}: {pct:.1f}%')
```

## Inspecting the guest CPU without booting a debugger

The Proxmox QEMU monitor let us inspect guest registers live:

```text
qm monitor 500
qm> info cpus
qm> info registers 1
```

The instruction pointer (`RIP`) on vCPU 1 repeatedly returned to a very narrow region:

```text
0xffffff800d3008dc
0xffffff800d30095c
0xffffff800d3009dc
0xffffff800d300a1d
```

All samples were in kernel mode (`CPL=0`), and the vCPU continued to consume 100% of a core. It looked like a hot kernel routine, not a blocked user-space process.

The monitor's disassembler wasn't available in this QEMU build (`Asm output not supported on this arch`), so we read instruction bytes instead:

```text
qm> cpu 1
qm> x/64xb 0xffffff800d300900
```

The captured bytes contained repeated 32-bit bitwise selection and pixel-like stores. One portion decoded to:

```asm
xor  r9d, esi
and  r9d, r8d
xor  r9d, esi
mov  dword ptr [rdi+4], r9d
```

Initially this looked like generic bitwise computation—possibly a cryptographic primitive. Without symbols, that guess was not reliable.

We also tried following frame pointers through guest stack memory, but some apparent return addresses were invalid. Because this was a **live changing stack**, manual stack unwinding quickly became less reliable than code identification.

## Symbolication: the actual function was drawing text

The guest ran:

```text
macOS Sequoia 15.7.2 (24G325)
Darwin 24.6.0
Kernel UUID: 90688F0F-33FC-3A1A-8DB6-EF9F59DBA1A9
```

Apple's download portal did not offer a Kernel Debug Kit (KDK) for that exact build. I downloaded **KDK 15.7.3 (24G419)** on another Mac instead. Its kernel binary and `.dSYM` agreed with each other, but **their UUID did not match the guest**. This means the KDK cannot be used for direct, exact address symbolication of the guest kernel.

Instead, I searched the nearby KDK kernel **for an exact sequence of machine-code bytes** captured from the stalled vCPU:

```python
from pathlib import Path

kernel = Path('path/to/KDK/System/Library/Kernels/kernel')
pattern = bytes.fromhex(
    '41 31 f1 45 21 c1 41 31 f1 '
    '44 89 4f 04 44 8b 40 08'
)
data = kernel.read_bytes()
print(data.find(pattern))  # one match, at file offset 0x2179df
```

There was exactly **one match** in the 18,355,376-byte kernel. Mapping its Mach-O file offset into the `__TEXT` segment produced virtual address `0xffffff80004179df`. Using the 15.7.3 KDK's own matching `.dSYM` yielded:

```text
vc_paint_char (in kernel) (video_console.c:1632)
```

The DWARF lookup further identified an **inlined** helper:

```text
vc_paint_char_32
Source: xnu/osfmk/console/video_console.c
```

**That was the breakthrough.** The busy vCPU was executing code matching XNU's **32-bit video-console character painter**. The bitwise masking and stores weren't hashing data: they were consistent with painting characters into a framebuffer.

> **Technical precision:** This was a unique *code-pattern match* between different macOS kernel builds, not an exact symbolication of build 24G325. The source attribution is strong evidence for the hotspot; the next boot test established whether verbose rendering explained the pathological elapsed time.

## One boot argument fixed it

The OpenCore configuration still contained the familiar verbose boot flag:

```text
-v
```

That flag directs macOS to show verbose kernel startup messages on the boot console. Given the kernel hotspot, the obvious hypothesis became: **console text rendering against the passed-through W6600 framebuffer was extraordinarily expensive in this configuration**.

We changed **only one variable** in the otherwise known-good OpenCore setup: removed `-v` from `NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82 -> boot-args`.

For illustration only (preserve the other args from *your own* config):

```text
Before: keepsyms=1 alcid=1 debug=0x100 agdpmod=pikera -v
After:  keepsyms=1 alcid=1 debug=0x100 agdpmod=pikera
```

The change was made **offline**: stop the VM, resolve its EFI disk with `pvesm path`, attach it with `losetup --partscan`, mount the FAT32 partition, edit `EFI/OC/config.plist`, validate with Python's `plistlib`, then unmount and detach cleanly. The EFI was backed up first.

On the next startup, **macOS came up quickly, the physical W6600 display worked, and the system was usable with graphics acceleration**. A ping monitor recorded the guest switching from OFFLINE at **17:44:30** to ONLINE at **17:45:31**; those samples bracket network availability, not the exact elapsed time from the `qm start` command. The visual result was immediate enough to be unmistakable compared with the former ~57-minute behavior.

Once stable, I upgraded OpenCore and its Lilu/WhateverGreen dependencies to newer **RELEASE** versions. The fast-boot, accelerated W6600 configuration continued working. This also explains why earlier OpenCore-version experiments were misleading: the verbose-console issue was still present.

## Why `vga: vmware` masked the problem

VMware SVGA had let the guest boot quickly, whereas making the physical W6600 the only display caused pathological delay. A plausible explanation is that the video-console output path behaves very differently depending on the selected framebuffer, and that frequent verbose text drawing to this particular passed-through GPU was expensive.

The evidence strongly supports that interpretation, **but we did not benchmark individual framebuffer writes or isolate the exact hardware/firmware interaction**. It would be overclaiming to say that *every* W6600 passthrough setup suffers the same slowdown.

## Practical troubleshooting checklist

1. **Measure the supposedly stuck phase.** A screen that says `PCI configuration begin` does not prove the kernel is still enumerating PCI devices. Search logs for the corresponding end timestamp.
2. **Compare one variable at a time.** A fast emulated display can isolate the physical framebuffer path, but avoid changing OpenCore, GPU settings, and kexts simultaneously.
3. **Observe the VM from the host during the stall.** Check memory pressure, disk activity, and per-vCPU CPU utilization. A single hot vCPU changes the investigation completely.
4. **Inspect the guest execution hotspot.** QEMU's `info registers` and `x/` can provide useful data even when the monitor cannot disassemble instructions.
5. **Symbolicate carefully.** Confirm matching kernel and dSYM UUIDs. If the guest build's KDK is unavailable, a unique machine-code match in a nearby build is useful for *identification*, but not exact symbolication.
6. **Test the smallest hypothesis.** Here, removing `-v`—not changing PCI settings, power management, or GPU properties—was the winning experiment.
7. **Save the working baseline.** Back up both the VM and its EFI before updating OpenCore or macOS.

## What this taught me

This investigation started with what looked like an AMD GPU initialization defect and led through AGDC, IOKit, FSKit, VirtualSMC, and QEMU. Many of the warnings were real; some were consequences of the delay, and several appeared even during successful boots.

The decisive observation was far simpler: **one virtual CPU spent the stall painting boot-console characters**. Identifying its instructions and mapping them to `vc_paint_char_32()` turned a vague one-hour hang into a testable, one-flag hypothesis.

If you're running macOS on Proxmox with a physical GPU and a mysteriously long verbose boot, checking `-v` is a low-effort diagnostic—**but make a backup and keep investigating if removing it doesn't help**.

### References

- [Apple XNU source: `video_console.c`](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/console/video_console.c)
- [Apple Developer Downloads (Kernel Debug Kits)](https://developer.apple.com/download/all/)
- [OpenCorePkg releases](https://github.com/acidanthera/OpenCorePkg/releases)
- [WhateverGreen releases](https://github.com/acidanthera/WhateverGreen/releases)
- [Lilu releases](https://github.com/acidanthera/Lilu/releases)
- [Proxmox VE QEMU/KVM manual](https://pve.proxmox.com/pve-docs/qm.1.html)
