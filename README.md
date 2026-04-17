# Dual-GPU Passthrough (AMD Navi 48 + NVIDIA) on Proxmox VE with AMD Reset-Bug Workaround

A working configuration for passing **two GPUs simultaneously** — an **AMD Radeon RX 9070 XT (Navi 48, `1002:7550`)**
and an **NVIDIA GeForce RTX 3060 (GA106, `10de:2503`)** — to a single Linux guest on **Proxmox VE 9** with an **EPYC
Rome** host. The novel part is the AMD side: Navi 48 exposes only a secondary-bus-reset method, which crashes the card's
internal PCIe switch during VM start. This guide bypasses the reset bug by presenting a dumped VBIOS via QEMU's
`romfile=`. The NVIDIA card is passed through alongside with conventional `vfio-pci` binding (no VBIOS dump needed).

---

## Table of contents

- [TL;DR](#tldr)
- [Prerequisites](#prerequisites)
- [Hardware applicability](#hardware-applicability)
- [Reproduction checklist](#reproduction-checklist)
- [Host Hardware](#host-hardware)
- [Software versions](#software-versions)
- [Required BIOS settings](#required-bios-settings-romed8-2t-ami-p410)
- [Host Linux configuration](#host-linux-configuration)
- [Finding your PCI address](#finding-your-pci-address)
- [IOMMU verification](#iommu-verification)
- [Dumping the card's VBIOS](#dumping-the-cards-vbios)
- [VM hookscript](#vm-hookscript)
- [NVIDIA passthrough (companion)](#nvidia-passthrough-companion)
- [VM configuration](#vm-configuration)
- [Verification](#verification)
- [Guest software stack (Fedora 42 example)](#guest-software-stack-fedora-42-example)
- [Verification — software stack inside the guest](#verification--software-stack-inside-the-guest)
- [Empirical test results](#empirical-test-results)
- [Why this works](#why-this-works)
- [Known quirks](#known-quirks)
- [References](#references)
- [License](#license)

---

## TL;DR

The Navi 48 reset bug breaks naïve GPU passthrough. The working recipe is:

1. Let **`amdgpu`** claim the AMD card at host boot (do **not** put `1002:7550` in `vfio.conf ids=`).
2. Dump the AMD card's **VBIOS** once via sysfs and store it at `/usr/share/kvm/<name>.bin`.
3. VM config uses `hostpci…: …,rombar=1,romfile=<name>.bin` for the AMD card.
4. A **hookscript** unbinds `amdgpu` at VM start and re-binds it at post-stop (with a link-health guard so a crashed
   card can't hang the rebind).
5. The NVIDIA card is passed via conventional `vfio-pci ids=` (no VBIOS dump, no hookscript magic needed).

This produces two working GPUs inside the guest: full compute (ROCm/HIP on AMD, CUDA on NVIDIA), rendering (Vulkan on
both), and hardware video (VAAPI AV1/HEVC/H.264 on AMD, NVENC on NVIDIA). The setup survives unlimited `qm shutdown` →
`qm start` cycles. The only failure path that still requires an IPMI power cycle is a **guest-initiated OS reboot**,
which bypasses the AMD post-stop hook.

---

## Prerequisites

### Required

- **Proxmox VE 9.x** installed on an **IOMMU-capable** host (AMD EPYC/Ryzen with SVM, or Intel Xeon/Core with VT-d).
- **Root shell access** to the Proxmox host (SSH or local console).
- A **physical AMD RX 9070 / 9070 XT** (Navi 48) card installed and detected by the host (`lspci | grep -i navi`).
- A **second PCIe GPU** if you want dual-vendor passthrough as described in this guide (an NVIDIA card here). The
  AMD-only case works identically — just drop `hostpci0` / NVIDIA IDs.
- Access to the host's **BIOS/UEFI setup** to enable IOMMU/SVM and related options (
  see [Required BIOS settings](#required-bios-settings-romed8-2t-ami-p410)) — either a physical keyboard + monitor, or
  IPMI/iKVM (see "Nice to have" below).
- Enough free space on the host for the **VBIOS dump file** (~1 MB; placed in `/usr/share/kvm/`).
- Linux-admin comfort with editing `/etc/default/grub`, `/etc/modules`, `/etc/modprobe.d/`, running `update-grub` and
  `update-initramfs`, and managing Proxmox VM configs via `qm` or the web UI.

---

## Hardware applicability

Tested and verified on the exact hardware in the [Host Hardware](#host-hardware) table below. Extrapolation notes:

| Variation                                                                            | Expected to work?                                                                                                     |
|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Different board partner (Sapphire, ASRock, MSI, PowerColor…) same RX 9070/XT silicon | Yes — **but you must dump the VBIOS from YOUR card** (board-specific ROM)                                             |
| RX 9070 (non-XT) / 9070 GRE (same Navi 48 die, `1002:7550`)                          | Likely yes — same reset-bug class                                                                                     |
| Other RDNA 4 (Navi 44, RX 9060 XT)                                                   | Plausible; not tested — use the same workflow, dump that card's VBIOS                                                 |
| RDNA 3 / Navi 3x (RX 7000)                                                           | Different reset behavior; this exact recipe is not required for most RDNA 3 cards                                     |
| Other EPYC Rome/Milan server boards (Supermicro H12, Gigabyte MZ32, Tyan S8030)      | Host side should work identically; BIOS menu paths differ — apply the equivalents                                     |
| Intel Xeon / Intel desktop host                                                      | Host side should work with `intel_iommu=on iommu=pt`; BIOS menu paths differ                                          |
| Guest OS other than Fedora 42                                                        | Any Linux with kernel ≥ 6.8 and in-tree `amdgpu` should work; software-stack section is Fedora-specific               |
| Windows guest                                                                        | Should work (forum reports). SeaBIOS or OVMF both acceptable                                                          |
| Single AMD 9070/XT only (no NVIDIA)                                                  | Fully supported — drop `hostpci0` / NVIDIA IDs from `vfio.conf`; the AMD setup is independent                         |
| Mixed NVIDIA + AMD in same VM                                                        | Tested and working — standard vfio-pci for NVIDIA alongside the AMD reset-bug workaround (this guide)                 |
| Single NVIDIA-only GPU passthrough                                                   | Overkill — use the Proxmox wiki's standard PCI passthrough guide; you don't need the VBIOS / hookscript sections here |

---

## Reproduction checklist

High-level sequence — each step links to details further below:

1. **BIOS**: set the [required BIOS values](#required-bios-settings-romed8-2t-ami-p410).
2. **GRUB**: set `GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt"`; `update-grub`.
3. **Modules**: put `vfio, vfio_iommu_type1, vfio_pci` in `/etc/modules`.
4. **vfio.conf**: `ids=` containing your non-AMD passthrough devices only; **no** `1002:7550` / `1002:ab40`; add
   `disable_idle_d3=1`.
5. **blacklist.conf**: blacklist the host drivers for other devices you're passing through (only).
6. `update-initramfs -u -k all` and reboot.
7. [Verify IOMMU is active](#iommu-verification) and `amdgpu` owns the AMD card.
8. [Identify your card's PCI address](#finding-your-pci-address) and subsystem.
9. [Dump the VBIOS](#dumping-the-cards-vbios) to `/usr/share/kvm/<name>.bin`.
10. Install the [VM hookscript](#vm-hookscript) at `/var/lib/vz/snippets/`.
11. Configure the VM (Q35, ostype=l26) with `hostpciN: <addr>,pcie=1,rombar=1,romfile=<name>.bin` and `hookscript:`.
12. `qm start <VMID>` and run [host + guest verification](#verification).
13. Inside the guest, install the [software stack](#guest-software-stack-fedora-42-example) (Vulkan, ROCm, VAAPI
    freeworld, FFmpeg full).

---

## Host Hardware

| Component       | Value                                                                                 |
|-----------------|---------------------------------------------------------------------------------------|
| Motherboard     | ASRockRack ROMED8-2T, revision 1.03                                                   |
| BIOS            | AMI P4.10 (2025-06-05)                                                                |
| CPU             | AMD EPYC 7532 (32 cores / 64 threads, AMD-V/SVM)                                      |
| Memory          | 8 × 64 GB DDR4-3200 ECC RDIMM (512 GB total)                                          |
| Boot device     | Samsung 990 PRO NVMe (PCIE2 / M2_1)                                                   |
| GPU #1 (AMD)    | AMD Radeon RX 9070 XT, XFX variant (`1002:7550`, sub `1eae:8811`), PCIe slot PCIE1    |
| GPU #2 (NVIDIA) | NVIDIA GeForce RTX 3060, ASUS variant (`10de:2503`, sub `1043:87f7`), PCIe slot PCIE4 |

### PCI device inventory (relevant endpoints)

| PCI           | Device                                             | Vendor:Device             | Role                         |
|---------------|----------------------------------------------------|---------------------------|------------------------------|
| `01:00.0`     | LSI SAS2308 HBA (9207-8i)                          | `1000:0087`               | passthrough                  |
| `42:00.0/.1`  | Intel X550-T2 dual 10 GbE                          | `8086:1563`               | host networking (`ixgbe`)    |
| `46:00.0`     | ASPEED BMC VGA                                     | `1a03:2000`               | host console                 |
| `47:00.0`     | NVIDIA GA106 (RTX 3060)                            | `10de:2503`               | **passthrough (this guide)** |
| `47:00.1`     | NVIDIA GA106 HDMI audio                            | `10de:228e`               | **passthrough (this guide)** |
| `84:00–86:00` | AMD Navi 48 (card's internal switch + GPU + audio) | `1002:7550` / `1002:ab40` | **passthrough (this guide)** |
| `c1:00.0/.1`  | Intel X710-T2L dual 10 GbE                         | `8086:15ff`               | passthrough                  |

IOMMU groups for the 9070 are clean:

```
Group 36: 86:00.0 VGA compatible controller [AMD Navi 48 RX 9070/XT]
Group 37: 86:00.1 Audio device [AMD Navi 48 HDMI/DP Audio Controller]
```

---

## Software versions

| Component           | Version                                     |
|---------------------|---------------------------------------------|
| Proxmox VE          | 9.1.8                                       |
| Kernel              | 6.17.13-3-pve                               |
| QEMU (from VM meta) | 9.2.0                                       |
| Guest OS (tested)   | Fedora 42 with kernel 6.19 + in-tree amdgpu |

---

## Required BIOS settings (ROMED8-2T, AMI P4.10)

| Menu path                                                                             | Value                                               |
|---------------------------------------------------------------------------------------|-----------------------------------------------------|
| `Advanced → AMD CBS → CPU Common Options → Global C-state Control`                    | **Disabled**                                        |
| `Advanced → AMD CBS → CPU Common Options → Power Supply Idle Control`                 | **Typical Current Idle**                            |
| `Advanced → AMD CBS → NBIO Common Options → SMU Common Options → Determinism Control` | **Manual**                                          |
| `Advanced → AMD CBS → NBIO Common Options → SMU Common Options → Determinism Slider`  | **Performance**                                     |
| `Advanced → AMD CBS → NBIO Common Options → SMU Common Options → DF Cstates`          | **Disabled**                                        |
| `Advanced → Chipset Configuration → Above 4G Decoding`                                | **Enabled**                                         |
| `Advanced → Chipset Configuration → Re-Size BAR Support`                              | **Enabled**                                         |
| `Advanced → Chipset Configuration → SR-IOV Support`                                   | Enabled (optional; does not affect AMD passthrough) |
| `Advanced → CPU Configuration → SVM Mode`                                             | **Enabled** (IOMMU)                                 |
| `Security → Secure Boot`                                                              | Disabled                                            |

---

## Host Linux configuration

### 1. GRUB kernel command line

`/etc/default/grub`:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt"
GRUB_CMDLINE_LINUX=""
```

Apply:

```bash
update-grub
```

> Note: `amd_iommu=on` is not needed (and emits `AMD-Vi: Unknown option` on modern kernels). `iommu=pt` alone is
> sufficient on EPYC.
> Note: `pcie_acs_override` is not needed — the 9070 and every other passthrough device already land in isolated IOMMU
> groups on this board.

### 2. Load vfio kernel modules at boot

`/etc/modules` — one module per line, loaded at boot before any network/PCI services:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

| Module             | Purpose                                                                                           | Required? |
|--------------------|---------------------------------------------------------------------------------------------------|-----------|
| `vfio`             | Core VFIO framework                                                                               | **yes**   |
| `vfio_iommu_type1` | IOMMU backend used by QEMU                                                                        | **yes**   |
| `vfio_pci`         | PCI device driver that binds passthrough devices                                                  | **yes**   |
| `vfio_virqfd`      | Legacy eventfd helper; folded into `vfio` since kernel 5.15 but loading it explicitly is harmless | optional  |

Only `vfio_pci` strictly needs to be in `/etc/modules` for `options` in `/etc/modprobe.d/vfio.conf` to take effect at
boot; the others are pulled in automatically as dependencies. Listing all four is the conventional Proxmox layout and
makes the intent explicit.

### 3. vfio-pci binding and D3 suppression

`/etc/modprobe.d/vfio.conf` — controls which PCI devices `vfio-pci` claims at boot and its runtime behavior:

```
options vfio-pci ids=1000:0087,8086:15ff,10de:2503,10de:228e
options vfio-pci disable_idle_d3=1
```

> **Important — AMD Navi 48 IDs are deliberately NOT in this list.** The 9070 XT must be claimed by `amdgpu` at boot so
> it fully posts the card (VBIOS, PSP/SMU resume, internal switch training). Only after this one-time init can the
> hookscript hand it off to `vfio-pci` cleanly. If you put `1002:7550` / `1002:ab40` in `ids=` to have vfio-pci claim the
> card directly, VM start will fail with `pcieport XX:00.0: broken device, retraining failed` — amdgpu's init is
> prerequisite on this card.

**The `ids=` list** is the set of `vendor:device` pairs that `vfio-pci` claims **before** any other driver can bind. It
is host-specific — include only the devices you want to pass through that do **not** need host-side driver init first:

| ID (this guide) | Device                  | Keep this ID if…                                |
|-----------------|-------------------------|-------------------------------------------------|
| `1000:0087`     | LSI SAS2308 HBA         | you pass an LSI 92xx-series HBA to a storage VM |
| `8086:15ff`     | Intel X710-T (10 GbE)   | you pass an Intel X710 NIC to a VM              |
| `10de:2503`     | NVIDIA GA106 (RTX 3060) | you have an NVIDIA GPU to pass through          |
| `10de:228e`     | NVIDIA GA106 HDMI audio | (paired with the NVIDIA GPU above)              |

Drop any ID whose device you do not own. The **AMD 9070 XT (`1002:7550`) and its audio function (`1002:ab40`) are NOT
in `ids=`** — amdgpu claims them at boot; the VM hookscript unbinds from amdgpu at VM start.

**`disable_idle_d3=1`** is a module-level option (not per-device). It tells `vfio-pci` never to put any of its claimed
devices into runtime D3 idle. For the Navi 48 this is critical: the audio function refuses `D3hot → D0` transitions
under vfio, which crashes the card's internal PCIe switch. Keeping vfio devices in D0 avoids the failure path. This
option is harmless for the other vfio-bound devices in the list; at worst they draw ~1 W more when idle.

### 4. Module blacklist

`/etc/modprobe.d/blacklist.conf` — modules that must **not** load on the host, so they don't race `vfio-pci` for
passthrough devices. Configure this for your own hardware; the table below explains which entries are required when:

```
blacklist mpt3sas
blacklist nouveau
blacklist nvidia
blacklist snd_hda_intel
blacklist i40e
```

| Entry           | Why it's here                                                               | Keep if…                                                                                                                                                  |
|-----------------|-----------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `mpt3sas`       | Native driver for the LSI SAS2308 HBA; would grab the disks before vfio-pci | you pass through an LSI SAS HBA (`1000:0087`)                                                                                                             |
| `nouveau`       | Open-source NVIDIA driver                                                   | you pass through an NVIDIA GPU                                                                                                                            |
| `nvidia`        | Proprietary NVIDIA driver                                                   | you pass through an NVIDIA GPU                                                                                                                            |
| `snd_hda_intel` | Generic HD-audio driver                                                     | your only HD-audio devices are GPU HDMI audio functions being passed through. **Do not blacklist** if the host has onboard analog audio you want to keep. |
| `i40e`          | Intel X710 driver                                                           | you pass through the full X710 NIC (`8086:15ff`)                                                                                                          |

Notes:

- For **AMD Navi 48 passthrough alone**, none of the above are strictly required — `amdgpu` is **not** blacklisted
  because `vfio-pci ids=` claims the card first at boot (the `ids=` list wins the race regardless of module load order).
  The only absolute requirement for this setup is that `vfio-pci` loads before any module that would compete.
- If you only have the 9070 XT and no NVIDIA/LSI/X710, this file can stay empty or unchanged from the Proxmox default.

### 5. Proxmox-default blacklist

`/etc/modprobe.d/pve-blacklist.conf` is installed and managed by Proxmox. It typically contains:

```
blacklist nvidiafb
```

Leave this file alone. It prevents a known crash with NVIDIA framebuffer console on some boards and is unrelated to
passthrough.

### 6. Rebuild initramfs

After any change to `/etc/modules`, `/etc/modprobe.d/*.conf`, or `/etc/default/grub`:

```bash
update-initramfs -u -k all
```

The `vfio.conf` options are baked into the initramfs so that `vfio-pci` sees its `ids=` list and `disable_idle_d3=1` at
the very first moment the module loads during early boot. Without rebuilding the initramfs, changes take effect only on
subsequent boots (or not at all, depending on Proxmox's boot layout).

Then reboot once so the new module options actually load.

---

## Finding your PCI address

The address `0000:86:00.0` used throughout this document is specific to the author's board layout. Find **your** address
before adapting commands:

```bash
# Locate the GPU's PCI address
lspci -nn | grep -iE "navi|radeon.*9070"
#   -> e.g. "86:00.0 VGA compatible controller [0300]:
#       Advanced Micro Devices, Inc. [AMD/ATI] Navi 48 [RX 9070/9070 XT] [1002:7550]"

# The domain is usually 0000: — full address is 0000:<bus>:<device>.<function>
# Note both functions (.0 = GPU, .1 = HDMI audio)
lspci -nn | grep "[1002:7550]\|[1002:ab40]"
```

Also find the card's **subsystem ID** — it identifies the board partner (XFX, Sapphire, ASRock, etc.) and is important
when comparing VBIOS dumps:

```bash
lspci -nn -d 1002:7550 -v | grep Subsystem
#   -> e.g. "Subsystem: XFX Limited Device [1eae:8811]"
```

Throughout this guide, substitute your actual PCI address and subsystem ID wherever `0000:86:00.0`, `0000:86:00.1`, or
`1eae:8811` appear.

---

## IOMMU verification

Before attempting passthrough, confirm IOMMU is active and the card sits in an isolated IOMMU group:

```bash
# IOMMU is up (AMD)
dmesg | grep -iE "AMD-Vi|iommu" | head -10
#   -> expect "AMD-Vi: Using global IVHD EFR: ..." and
#               "iommu: Default domain type: Passthrough (set via kernel command line)"

# Intel equivalent: "DMAR: IOMMU enabled"

# The card's IOMMU group must not contain unrelated host devices.
# Replace 86:00 with your PCI address.
for dev in /sys/bus/pci/devices/0000:86:00.*; do
    group=$(basename $(readlink $dev/iommu_group))
    echo "Group $group: $(lspci -nns $(basename $dev))"
done
#   -> both functions should land in small groups containing only
#      the card and its own bridges (no host storage / NICs / etc.)
```

If the group contains unrelated devices you would also need to pass through, the board's PCIe topology doesn't give you
clean isolation. You can either:

- move the card to a different slot (usually the lowest-numbered CPU-direct x16 slot has the best isolation), or
- as a last resort, use `pcie_acs_override=downstream,multifunction` in the GRUB cmdline (security trade-off — read the
  caveats in the Proxmox wiki first).

---

## Dumping the card's VBIOS

Done **once**, from the host, with no VM running. The dumped file must match this exact card/SKU (board partner and
silicon revision). Do not use ROMs dumped from other 9070 XT boards.

### Precondition

With the `vfio.conf` above, `amdgpu` claims the card at boot. Confirm this before dumping:

```bash
lspci -nnk -s 86:00.0 | grep "Kernel driver"
#   -> Kernel driver in use: amdgpu
```

If it shows `vfio-pci` instead (because the card's IDs are still in `vfio.conf ids=` or because a VM has been started),
remove those IDs from `vfio.conf`, `update-initramfs -u -k all`, and reboot first.

### Dump

```bash
echo 1 > /sys/bus/pci/devices/0000:86:00.0/rom
cat /sys/bus/pci/devices/0000:86:00.0/rom > /usr/share/kvm/vbios_9070_xfx.bin
echo 0 > /sys/bus/pci/devices/0000:86:00.0/rom
chmod 644 /usr/share/kvm/vbios_9070_xfx.bin

# Sanity checks
ls -l /usr/share/kvm/vbios_9070_xfx.bin
file /usr/share/kvm/vbios_9070_xfx.bin
#   -> BIOS (ia32) ROM Ext. ... PCI AMD/ATI device=0x7550
hexdump -C /usr/share/kvm/vbios_9070_xfx.bin | head -1
#   -> expect leading "55 aa ..."
strings /usr/share/kvm/vbios_9070_xfx.bin | grep -iE "Navi|AMD|ATOMBIOS" | head -5
#   -> expect "NAVI48", "ATOMBIOS", board part string
```

A bad dump looks like all-0xFF or is very small (<32 KB). Re-dump if that happens.

The ROM file must live under `/usr/share/kvm/` (standard Proxmox/QEMU lookup path for `romfile=`).

---

## VM hookscript

Purpose:

- **pre-start**: force both functions out of runtime idle, unbind `amdgpu` from the GPU function, resize BAR 2 to 8 MB.
- **post-stop**: if the card is still healthy, re-post it via `amdgpu` so the next VM start can survive another bus
  reset. A link-health guard prevents `amdgpu` bind on a broken card (which would hang in a `trn=2 ACK` loop and wedge
  `qm cleanup`).

Path: `/var/lib/vz/snippets/rx9070_reset.sh`

```bash
#!/bin/bash
phase="$2"
echo "Phase is $phase"
if [ "$phase" == "pre-start" ]; then
    # Disable runtime PM so both functions stay in D0
    echo "on" > /sys/bus/pci/devices/0000:86:00.0/power/control 2>/dev/null
    echo "on" > /sys/bus/pci/devices/0000:86:00.1/power/control 2>/dev/null
    sleep 1
    # Hand GPU function from amdgpu to vfio-pci.
    # amdgpu must have posted the card (at host boot or previous post-stop)
    # for vfio-pci + romfile to survive the bus reset QEMU issues at VM init.
    echo "0000:86:00.0" > /sys/bus/pci/drivers/amdgpu/unbind 2>/dev/null
    sleep 2
    # Resize BAR2 to 8 MB (3 = log2(8MB)); Linux guest preference
    echo 3 > /sys/bus/pci/devices/0000:86:00.0/resource2_resize 2>/dev/null
    sleep 1
elif [ "$phase" == "post-stop" ]; then
    sleep 5
    # Re-post card via amdgpu so the NEXT VM start can succeed.
    # GUARD: skip if the card looks broken (link dead) — rebinding amdgpu
    # on a broken card hangs in trn=2 ACK loop and wedges qm cleanup.
    link_speed=$(cat /sys/bus/pci/devices/0000:86:00.0/current_link_speed 2>/dev/null)
    if [ -n "$link_speed" ] && [ "$link_speed" != "Unknown" ]; then
        echo "post-stop: card looks healthy (link=$link_speed), rebinding amdgpu"
        echo "0000:86:00.0" > /sys/bus/pci/drivers/vfio-pci/unbind 2>/dev/null
        sleep 2
        echo "0000:86:00.0" > /sys/bus/pci/drivers/amdgpu/bind 2>/dev/null
        sleep 5
    else
        echo "post-stop: card in bad state (link=$link_speed), skipping amdgpu rebind"
    fi
fi
```

```bash
chmod +x /var/lib/vz/snippets/rx9070_reset.sh
```

> Note: the hookscript only manages the GPU function `86:00.0`. The audio function `86:00.1` has no host driver (no
`amdgpu` binding for audio, `snd_hda_intel` is blacklisted). QEMU attaches and releases vfio-pci to it automatically at
> VM start/stop.
>
> Multi-cycle behavior: with the guarded post-stop rebind, `qm start` / `qm shutdown` / `qm start` cycles succeed
> repeatedly within a host session. The only non-recoverable state is when the card ends up in the "broken" state mid-VM (
> very rare under this config); the guard detects that and skips the rebind so the host doesn't hang.

---

## NVIDIA passthrough (companion)

The NVIDIA GeForce RTX 3060 is passed through alongside the AMD card using the standard `vfio-pci` approach. Unlike the
AMD side, NVIDIA requires no VBIOS dump, no hookscript, and no special reset handling — the GA10x family passes cleanly
through QEMU's default reset path.

### Why NVIDIA is simpler than AMD on this guide

| Aspect                           | NVIDIA (RTX 3060 / GA106)                 | AMD (RX 9070 XT / Navi 48)                                              |
|----------------------------------|-------------------------------------------|-------------------------------------------------------------------------|
| Reset method exposed             | `flr` (function-level reset)              | **`bus` only** (secondary bus reset crashes the card's internal switch) |
| Needs VBIOS dump?                | No                                        | **Yes** (`romfile=`)                                                    |
| Needs `amdgpu`/driver pre-init?  | No — can bind `vfio-pci` directly at boot | **Yes** — `amdgpu` must post the card first                             |
| Needs hookscript?                | No                                        | **Yes** — to handle amdgpu ↔ vfio-pci transitions                       |
| Guest-initiated reboot survives? | Yes (clean FLR on reboot)                 | **No** (bus reset crashes switch)                                       |

### Host configuration (NVIDIA)

Already covered in the [Host Linux configuration](#host-linux-configuration) section, but summarized here for clarity:

- **`/etc/modprobe.d/vfio.conf`** contains the NVIDIA device IDs so `vfio-pci` claims them at boot:

  ```
  options vfio-pci ids=1000:0087,8086:15ff,10de:2503,10de:228e
  ```

    - `10de:2503` — GA106 GPU (RTX 3060)
    - `10de:228e` — GA106 HDMI audio controller

- **`/etc/modprobe.d/blacklist.conf`** blacklists both NVIDIA host drivers:

  ```
  blacklist nouveau
  blacklist nvidia
  ```

  This prevents the host from trying to load any NVIDIA driver on boot (the host doesn't need GPU rendering — the card
  exists only for passthrough).

- **IOMMU group**: verify `47:00.0` and `47:00.1` land in their own group (on this board, group 65 for both functions —
  acceptable since both are passed to the same VM):

  ```bash
  for d in /sys/bus/pci/devices/0000:47:00.*; do
      echo "$(basename $d) -> group $(basename $(readlink $d/iommu_group))"
  done
  ```

### VM configuration (NVIDIA)

A single line — no `rombar`, no `romfile`, no hookscript:

```bash
qm set <VMID> -hostpci0 0000:47:00,pcie=1
```

- `0000:47:00` with no function suffix means **both functions** (GPU + HDMI audio) are passed as one PCI multifunction
  device.
- `pcie=1` presents it on a PCIe root port in the guest (Q35 machine).
- No `rombar`/`romfile` needed — NVIDIA's on-card ROM is read cleanly over PCI.
- No hookscript line — the same hookscript used for AMD is fine; it doesn't touch the NVIDIA addresses.

### Guest driver (NVIDIA, Fedora 42 example)

Fedora doesn't ship the proprietary NVIDIA driver. Install from RPM Fusion (nonfree):

```bash
# RPM Fusion should already be enabled (used earlier for ffmpeg + mesa freeworld)
dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda        # + CUDA user-space runtime
dnf install nvidia-settings                              # optional, for GUI
# kernel module builds via akmods — wait ~3-5 min after install, or:
akmods --force
# Reboot the guest once after install so nvidia.ko loads (clean power-off + qm start,
# NOT `reboot` — see Known quirks about guest reboot with the AMD card attached)
```

Alternative: CUDA Toolkit from NVIDIA's own repo (adds `nvidia-container-toolkit`, `cuda-drivers`, etc.) — useful if you
intend to run GPU-accelerated Docker containers. See NVIDIA's installation guide for Fedora.

> Install ordering matters: install the NVIDIA driver **before** the first `reboot` of the guest. If you've been using
> the guest with only the AMD card, shut the guest down (`poweroff`), add `hostpci0` from the Proxmox host, `qm start`,
> then install the NVIDIA driver.

### Verification (NVIDIA-specific)

```bash
# Host side — card bound to vfio-pci
lspci -nnk -s 47:00
#   47:00.0 ... Kernel driver in use: vfio-pci
#   47:00.1 ... Kernel driver in use: vfio-pci

# Guest side
lspci -nnk | grep -iA2 "NVIDIA"
#   01:00.0 ... NVIDIA Corporation GA106 [GeForce RTX 3060]
#       Kernel driver in use: nvidia
#   01:00.1 ... GA106 High Definition Audio Controller
#       Kernel driver in use: snd_hda_intel

nvidia-smi
#   Displays card, driver version, VRAM, temp, power, utilization

# Compute sanity
nvidia-smi -q | grep -E "CUDA Version|Driver Version|Product Name"
nvcc --version 2>/dev/null || echo "(install cuda-toolkit for nvcc)"
```

`nvidia-smi` output from the tested setup:

```
NVIDIA-SMI 580.126.18    Driver Version: 580.126.18    CUDA Version: 13.0
GeForce RTX 3060    59°C   17W / 170W    4 MiB / 12288 MiB    0% utilization
```

### Dual-GPU coexistence in the guest

When both cards are attached, the guest sees them on different PCIe root ports. With Fedora's auto-detection:

```
/dev/dri/card0       — virtual VGA (bochs, used by the guest console)
/dev/dri/card1       — NVIDIA RTX 3060 (nvidia-drm)
/dev/dri/card2       — AMD RX 9070 XT (amdgpu)
/dev/dri/renderD128  — NVIDIA render node
/dev/dri/renderD129  — AMD render node
```

Both kernel modules (`nvidia` and `amdgpu`) coexist without conflict. Userspace apps can target either card explicitly:

- Vulkan: pick the device via the `VK_PHYSICAL_DEVICE_*` extension, environment variable `VK_DRIVER_FILES=`, or
  `vulkaninfo`'s GPU index.
- OpenGL: `DRI_PRIME=1 glxinfo` selects the non-default DRM card.
- CUDA/HIP: `CUDA_VISIBLE_DEVICES=0` and `HIP_VISIBLE_DEVICES=0` (each only sees its own vendor's cards).
- FFmpeg VAAPI: `-vaapi_device /dev/dri/renderD129` (the AMD render node).
- FFmpeg NVENC: `-c:v h264_nvenc` (automatically picks the NVIDIA card).

---

## VM configuration

The important lines (the rest of the VM is unrelated to passthrough):

```
agent: 1
cores: 60
cpu: host,flags=+nested-virt;+ibpb;+virt-ssbd;+pdpe1gb;+aes
machine: q35
memory: 196608
numa: 0
ostype: l26
scsihw: virtio-scsi-single
hookscript: local:snippets/rx9070_reset.sh
hostpci0: 0000:47:00,pcie=1
hostpci1: 0000:86:00,pcie=1,rombar=1,romfile=vbios_9070_xfx.bin
```

Attach the hookscript and both `hostpciN` lines from the CLI:

```bash
qm set <VMID> -hookscript local:snippets/rx9070_reset.sh
qm set <VMID> -hostpci0 0000:47:00,pcie=1                                              # NVIDIA
qm set <VMID> -hostpci1 0000:86:00,pcie=1,rombar=1,romfile=vbios_9070_xfx.bin          # AMD
```

Key options explained:

| Option                                  | Why                                                                                                                                     |
|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| `pcie=1`                                | Present device to guest on a PCIe root port (Q35 machine type).                                                                         |
| `rombar=1` (AMD only)                   | Expose the ROM BAR to the guest.                                                                                                        |
| `romfile=vbios_9070_xfx.bin` (AMD only) | QEMU serves this file as the AMD card's ROM instead of reading it from the physical card over PCI. This is what bypasses the reset bug. |

The NVIDIA passthrough line needs **no** `romfile` or `rombar=1` — NVIDIA cards don't have the Navi 48 reset bug and
handle the standard vfio-pci reset sequence cleanly.

Notes:

- `hostpciN` numbering (0, 1, …) is just a slot number in the VM's PCIe root complex — the order doesn't matter
  functionally. Used here with NVIDIA on `hostpci0` and AMD on `hostpci1` to keep things consistent with VM identity and
  guest-side PCI enumeration (NVIDIA appears at guest `01:00`, AMD at `02:00`).
- If you want a **single-GPU** setup (AMD only, no NVIDIA), just drop the `hostpci0` line and the NVIDIA IDs from
  `vfio.conf`.
- SeaBIOS (Proxmox default) works for a headless Linux guest. OVMF also works but requires a guest bootloader that knows
  UEFI.

---

## Verification

### On the host, after `qm start <VMID>`

```bash
lspci -nnk -s 86:00
#   86:00.0 ... Kernel driver in use: vfio-pci
#   86:00.1 ... Kernel driver in use: vfio-pci

cat /sys/bus/pci/devices/0000:86:00.0/current_link_speed
#   32.0 GT/s PCIe          <-- card's internal switch link; must remain populated

cat /sys/bus/pci/devices/0000:86:00.0/current_link_width
#   16

dmesg | grep -iE "broken device|retraining failed|stuck in D3|Refused to change power state"
#   (no hits = good)
```

### Inside the guest

```bash
# Both GPUs visible
lspci -nnk | grep -iEA2 "Navi 48|NVIDIA"
#   01:00.0 ... NVIDIA Corporation GA106 [GeForce RTX 3060]
#       Kernel driver in use: nvidia
#   01:00.1 ... GA106 High Definition Audio Controller
#       Kernel driver in use: snd_hda_intel
#   02:00.0 ... Navi 48 [Radeon RX 9070/9070 XT]
#       Kernel driver in use: amdgpu
#   02:00.1 ... HDMI/DP Audio Controller
#       Kernel driver in use: snd_hda_intel

# Both kernel modules loaded
lsmod | grep -iE "amdgpu|nvidia" | head -6
#   amdgpu, nvidia_drm, nvidia_modeset, nvidia, nvidia_uvm, ...

dmesg | grep amdgpu | grep -iE "VRAM|DMUB|SMU|Initialized"
#   VRAM: 16304M
#   SMU is initialized successfully!
#   [drm] DMUB hardware initialized
#   Initialized amdgpu X.XX.X for 0000:02:00.0 on minor 2

# DRM device nodes
ls /dev/dri/
#   card0         — virtual VGA (bochs)
#   card1         — NVIDIA (nvidia-drm)
#   card2         — AMD (amdgpu)
#   renderD128    — NVIDIA render
#   renderD129    — AMD render
```

`rocm-smi` shows the AMD card; `nvidia-smi` shows the NVIDIA card. Both can run simultaneously.

---

## Guest software stack (Fedora 42 example)

The VM in this setup runs Fedora 42 as a compute/media node. Packages that enable the full GPU feature set:

### Core compute

```bash
# Vulkan + Mesa
dnf install vulkan-tools vulkan-headers vulkan-validation-layers \
            spirv-tools glslc glslang \
            mesa-libGL mesa-libEGL mesa-libgbm mesa-libOSMesa mesa-libGLU \
            mesa-dri-drivers mesa-vdpau-drivers \
            glx-utils

# ROCm (HIP / HSA / OpenCL / math libs — Fedora-packaged)
dnf install rocminfo rocm-runtime rocm-opencl rocm-clinfo \
            rocm-hip-devel rocm-hip-runtime rocm-comgr \
            rocblas rocsolver hipblas rocm-smi
```

### Hardware video (VAAPI) — requires RPM Fusion

Fedora's stock `mesa-va-drivers` strips the H.264/HEVC encoders for patent reasons. RPM Fusion provides a "freeworld"
build with the encoders restored. On the 9070 XT this unlocks H.264, HEVC (Main + Main10), and AV1 hardware encode:

```bash
# Enable RPM Fusion first (see https://rpmfusion.org)
dnf install libva-utils
dnf swap mesa-va-drivers mesa-va-drivers-freeworld --allowerasing
dnf swap mesa-vdpau-drivers mesa-vdpau-drivers-freeworld --allowerasing
```

### FFmpeg (full codecs + VAAPI)

The Fedora base ships `ffmpeg-free` (limited codec set). Swap for the full RPM Fusion build:

```bash
dnf swap ffmpeg-free ffmpeg --allowerasing
```

### AMD AMF on Linux

Skip. AMF is the Windows / `amdgpu-pro` proprietary encode API. The open-source Linux path is VAAPI (above) or the newer
Vulkan Video encode. `libamfrt64.so` is not part of any Fedora package.

### NVIDIA proprietary driver + CUDA (RPM Fusion)

Fedora's stock repos ship only the `nouveau` driver; for CUDA/NVENC you need NVIDIA's proprietary stack:

```bash
# Proprietary driver + CUDA userspace (RPM Fusion nonfree)
dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda

# Optional:
dnf install nvidia-settings                            # control panel
dnf install xorg-x11-drv-nvidia-cuda-libs              # extra CUDA libs
```

`akmod-nvidia` builds the kernel module against the running kernel via DKMS-like `akmods`. After `dnf install`, wait ~
3–5 minutes for the first build to finish (check `systemctl status akmods.service` or look for
`/lib/modules/$(uname -r)/extra/nvidia/nvidia.ko.xz`), then shut the guest down and `qm start` to load `nvidia.ko`
fresh (don't use `reboot` — see Known quirks).

For **CUDA Toolkit** (nvcc + dev headers) and/or **NVIDIA Container Toolkit** (GPU-accelerated Docker), add NVIDIA's
official CUDA repo:

```bash
# Per https://developer.download.nvidia.com/compute/cuda/repos/
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/fedora41/x86_64/cuda-fedora41.repo
dnf install cuda-toolkit
# Container runtime for Docker:
dnf install nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

(On Fedora 42, the `fedora41` CUDA repo is the closest available and works fine in practice.)

---

## Verification — software stack inside the guest

```bash
# Kernel & GPU visibility
uname -r                                              # 6.19.x
lspci -nnk | grep -iA2 "Navi 48"                     # Kernel driver in use: amdgpu
ls /dev/dri/                                         # card0 + card1 + renderD128

# VRAM
cat /sys/class/drm/card1/device/mem_info_vram_total  # 16 GB raw bytes

# Vulkan
vulkaninfo --summary | grep -A2 "deviceName"
# -> AMD Radeon RX 9070 XT (RADV GFX1201)
vkcube                                               # (needs display; smoke test only)

# OpenCL
clinfo | grep -E "Platform Name|Board name|Device Topology|compute units"
# -> AMD Accelerated Parallel Processing, AMD Radeon RX 9070 XT, 32 CUs

# ROCm / HSA
rocminfo | grep -B1 -A1 "gfx1201"
# -> Agent: AMD Radeon RX 9070 XT, gfx1201
rocm-smi                                             # temp, power, clocks, util

# NVIDIA driver + CUDA + NVENC
nvidia-smi                                           # GPU, driver version, CUDA version
nvidia-smi -q | grep -E "Product Name|Driver Version|CUDA Version"
nvcc --version                                       # CUDA compiler (if cuda-toolkit installed)
ffmpeg -hide_banner -encoders | grep -i nvenc        # h264_nvenc, hevc_nvenc, av1_nvenc
# NVENC smoke test
ffmpeg -f lavfi -i testsrc=duration=1:size=1920x1080:rate=30 -c:v h264_nvenc -f null -

# VAAPI (AMD — encode + decode support)
vainfo --display drm --device /dev/dri/renderD129 | grep -E "Driver version|Profile"
# Note: with NVIDIA also present, the AMD render node is usually renderD129 (not 128).
# Check `ls /dev/dri/` and match to the card you want via `cat /sys/class/drm/card*/device/uevent`.
# -> Driver version: Mesa Gallium driver 25.x for AMD Radeon RX 9070 XT (radeonsi, gfx1201)
# -> H264/HEVC/AV1: both VLD (decode) and EncSlice (encode) entrypoints

# FFmpeg AMD VAAPI HW encode smoke tests  (adjust render node to YOUR AMD card)
AMD_DRI=/dev/dri/renderD129
ffmpeg -vaapi_device $AMD_DRI \
       -f lavfi -i testsrc=duration=1:size=1920x1080:rate=30 \
       -vf format=nv12,hwupload -c:v h264_vaapi -f null -
ffmpeg -vaapi_device $AMD_DRI \
       -f lavfi -i testsrc=duration=1:size=1920x1080:rate=30 \
       -vf format=nv12,hwupload -c:v hevc_vaapi -f null -
ffmpeg -vaapi_device $AMD_DRI \
       -f lavfi -i testsrc=duration=1:size=1920x1080:rate=30 \
       -vf format=nv12,hwupload -c:v av1_vaapi  -f null -

# FFmpeg NVIDIA NVENC HW encode smoke test (picks the NVIDIA card automatically)
ffmpeg -f lavfi -i testsrc=duration=1:size=1920x1080:rate=30 \
       -c:v h264_nvenc -f null -
ffmpeg -f lavfi -i testsrc=duration=1:size=1920x1080:rate=30 \
       -c:v hevc_nvenc -f null -
```

Expected: all three encode pipelines complete with `speed` well above `1x` real-time.

---

## Empirical test results

Setup verified end-to-end with the following workload, repeated across VM lifecycle events:

**Load:** 60-second VAAPI HEVC encode, 3840×2160 @ 60 fps, 80 Mbps target bitrate, output to `/dev/null`.

```bash
ffmpeg -hide_banner -re -f lavfi -i testsrc=size=3840x2160:rate=60 \
       -vaapi_device /dev/dri/renderD128 -vf format=nv12,hwupload \
       -c:v hevc_vaapi -b:v 80M -f null -
```

**Observed (sampled via `rocm-smi`):**

| Phase                                      | Temp  | Power    | MCLK        | SCLK               | GPU%   | Note                                                                                  |
|--------------------------------------------|-------|----------|-------------|--------------------|--------|---------------------------------------------------------------------------------------|
| Idle pre-load                              | 36 °C | 4 W      | 96 MHz      | 0 MHz              | 0 %    | PwrCap 340 W                                                                          |
| Load (peak)                                | 37 °C | **24 W** | **772 MHz** | 0–594 MHz (bursts) | 0–15 % | VCN engaged; shader CUs idle (VAAPI is fixed-function)                                |
| Idle post-load                             | 37 °C | 4 W      | 96 MHz      | 0 MHz              | 0 %    | Returned to baseline                                                                  |
| **After `qm shutdown` + `qm start` cycle** |       |          |             |                    |        |                                                                                       |
| Idle pre-load (cycle 2)                    | 38 °C | 5 W      | 96 MHz      | 0 MHz              | 0 %    | Hookscript post-stop cleanly rebound `amdgpu`, then pre-start handed back to vfio-pci |
| Load (peak, cycle 2)                       | 39 °C | **24 W** | **772 MHz** | 0–41 MHz           | 0–3 %  | Identical behavior to cycle 1                                                         |
| Idle post-load (cycle 2)                   | 39 °C | 5 W      | 96 MHz      | 0 MHz              | 0 %    |                                                                                       |

**Host dmesg during cycle 2 VM start:** no `broken device`, no `retraining failed`, no `stuck in D3`. Link remained at
32 GT/s ×16 throughout.

**Conclusion:** the card is fully functional, identical behavior across VM restart cycles, no degradation. `amdgpu` init
on post-stop takes ~5 s and is visible in `journalctl -u qmeventd`:

```
qmeventd[…]: Phase is post-stop
qmeventd[…]: post-stop: card looks healthy (link=32.0 GT/s PCIe), rebinding amdgpu
kernel: amdgpu 0000:86:00.0: amdgpu: initializing kernel modesetting
```

---

## Why this works

The Navi 48 exposes only `bus` as its reset method (no FLR, no PM reset):

```bash
$ cat /sys/bus/pci/devices/0000:86:00.0/reset_method
bus
```

A secondary bus reset from the motherboard-side bridge crashes the card's internal PCIe switch (
`pcieport 0000:85:00.0: broken device, retraining failed`), leaving the card unreachable until a full host power cycle.

Two things combine to make passthrough survive:

1. **`amdgpu` posts the card at host boot.** The card must go through the full VBIOS / PSP / SMU / DMUB init sequence
   once per host boot. Only `amdgpu` performs this. If vfio-pci binds directly (via `ids=`), the card never reaches a
   correctly-posted state and the bus reset at VM start crashes the internal switch.
2. **`rombar=1,romfile=` lets QEMU present the VBIOS to the guest from a memory image** instead of reading it from the
   physical card over PCI. This avoids the specific ROM-read cycle that triggers a second bus reset during guest
   `amdgpu` init.

The hookscript bridges these two: at VM start, unbind `amdgpu` (card stays initialized in hardware), then QEMU attaches
`vfio-pci` to the already-posted card and hands the romfile to the guest.

---

## Known quirks

- **Do not reboot the guest OS.** The card survives cold VM starts (via amdgpu-posted + romfile) but a guest-initiated
  reboot (or `qm reboot`) re-triggers the bus reset without amdgpu's prep, crashing the card's internal switch. Workflow
  for guest kernel updates:
    1. Shut the guest down cleanly (`poweroff` inside, or `qm shutdown <VMID>` from host) so the hookscript post-stop
       rebinds amdgpu.
    2. Start the VM fresh: `qm start <VMID>`.

  `qm shutdown` → `qm start` cycles work indefinitely thanks to the hookscript's post-stop rebind. Guest reboot is the
  one path that bypasses the post-stop hook and leaves the card wedged → host IPMI power cycle required.
- **Post-stop `amdgpu` rebind is guarded.** If the card was already crashed mid-VM (very rare under this config), the
  hookscript's link-health check skips the rebind to avoid the `trn=2 ACK should not assert` hang. In that case the next
  `qm start` will fail cleanly ("stuck in D3"); recovery is an IPMI power cycle.
- **Do not put `1002:7550` / `1002:ab40` in `vfio.conf ids=`.** If vfio-pci claims the card at boot instead of amdgpu,
  VM start fails with `pcieport XX:00.0: broken device, retraining failed`. The card *must* be posted by amdgpu first.
- **`kvm: vfio: Unable to power on device, stuck in D3`** printed on `qm start` means the card is not in a
  correctly-posted state. Usually caused by vfio-pci claiming the card at boot, or by a failed previous VM run that
  bypassed post-stop. Remedy: verify `amdgpu` owns the card pre-start; if the card is stuck, power-cycle the host.
- **`error writing '1' to '/sys/bus/pci/devices/0000:86:00.0/reset': Inappropriate ioctl for device`** in the `qm start`
  output is benign — the card has no FLR method, Proxmox falls through to the QEMU-internal reset path which succeeds
  with the ROM file in place.
- **Guest's `amdgpu` may log `[drm] Cannot find any crtc or sizes`** and `Runtime PM not available`. These are expected
  for a headless passthrough GPU with no physical display attached and do not affect compute or rendering.

---

## References

- Proxmox forum: _Issues moving from NVIDIA RTX 3080 GPU passthrough to AMD RX 9070
  XT_ — https://forum.proxmox.com/threads/issues-moving-from-nvidia-rtx3080-gpu-passthrough-to-amd-rx-9070-xt.177253/
- Proxmox forum: _AMD GPU passthrough recently stopped
  working_ — https://forum.proxmox.com/threads/amd-gpu-passthrough-recently-stopped-working.165977/
- Proxmox PCI passthrough documentation — https://pve.proxmox.com/wiki/PCI_Passthrough
- ASRockRack ROMED8-2T manual — https://www.asrockrack.com/general/productdetail.asp?Model=ROMED8-2T

---

## License

This documentation is released under the [MIT License](https://opensource.org/licenses/MIT). Configuration files and
scripts provided as-is with no warranty.
