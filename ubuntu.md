<div align="center">

# 🐧 Windows PC → Ubuntu Desktop for Quantus Mining

**Install Ubuntu on a spare disk without losing Windows, and get the NVIDIA driver working**

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/download/desktop)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Boot](https://img.shields.io/badge/Boot-UEFI%20%C2%B7%20GPT-blue?style=flat-square)](https://rufus.ie/en/)
[![Dual Boot](https://img.shields.io/badge/Windows-Preserved-0078D4?style=flat-square&logo=windows&logoColor=white)](https://ubuntu.com/tutorials/install-ubuntu-desktop)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Target:** a Windows 10/11 desktop with an NVIDIA GPU and a spare SSD
> **Result:** bare-metal Ubuntu with a working CUDA driver, Windows untouched
> **Last Updated:** September 2026

---

## Scope

This document does exactly one job: it turns a Windows machine into a working Ubuntu machine with a live NVIDIA driver. **No miner, no wallet, no seed phrase, and no choice about pooling yet.**

**Where this ends:** at [Step 8](#step-8--final-check). From there you continue in the main guide, which starts with two shared steps and only then asks you to pick a route:

```text
ubuntu.md  Step 1 → 8      you are here
      ↓
guide.md   Step 1  create a wallet
guide.md   Step 2  verify the GPU   (a 2-minute re-check of what you built here)
guide.md   Decision Point → Part A (pool) or Part B (own node)
```

Do not skip ahead to Part A or Part B from here. The wallet in guide.md Step 1 is a prerequisite for both.

Renting a GPU instead? → [vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md) · Network overview → [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md)

**Why bother:** the fast CUDA path exists only on Linux. The Windows miner is **4–6× slower** on the same card — ~100 MH/s versus 400–450 MH/s on a mid-range RTX.

---

## Table of Contents

- [What You Need](#what-you-need)
- [Step 1 — Download and Verify the ISO](#step-1--download-and-verify-the-iso)
- [Step 2 — Write the USB with Rufus](#step-2--write-the-usb-with-rufus)
- [Step 3 — BIOS and Boot Menu](#step-3--bios-and-boot-menu)
- [Step 4 — Identify the Target Disk](#step-4--identify-the-target-disk)
- [Step 5 — Install Ubuntu](#step-5--install-ubuntu)
- [Step 6 — Updates and the NVIDIA Driver](#step-6--updates-and-the-nvidia-driver)
- [Step 7 — Disable Suspend](#step-7--disable-suspend)
- [Step 8 — Final Check](#step-8--final-check)
- [Troubleshooting](#troubleshooting)

---

## What You Need

| Item | Requirement |
|---|---|
| PC | Windows 10/11 booting in **UEFI** mode (Legacy/CSM off) |
| GPU | NVIDIA RTX 20/30/40/50 series — AMD does not work with the CUDA miner |
| Disk | A **separate SSD** with nothing valuable on it — the installer erases it |
| USB stick | 8 GB+. Rufus wipes it, so back it up first |
| Network | Wired Ethernet preferred — the installer downloads drivers |

Three ground rules:

1. **Bare metal.** Not WSL, not a VM.
2. **Keep Windows.** One OS per physical disk. Avoid *Install alongside Windows*, which squeezes Ubuntu onto the Windows drive.
3. **No seed phrase on this machine.** Wallet creation happens on your phone — [guide.md → Step 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-1--create-a-wallet).

---

## Step 1 — Download and Verify the ISO

1. From [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop), take the **Intel/AMD 64-bit Desktop** LTS image — not an ARM ISO.
2. RTX 50-series (Blackwell) needs a newer kernel: prefer **26.04.x**. For 30/40-series, 24.04 LTS is fine.
3. Download `SHA256SUMS` from the same page and verify in PowerShell:

```powershell
Get-FileHash .\ubuntu-*-desktop-amd64.iso -Algorithm SHA256
```

> ⚠️ If the hash does not match the `SHA256SUMS` line, delete the ISO and download it again. Never write a corrupt image.

---

## Step 2 — Write the USB with Rufus

Get Rufus from [rufus.ie](https://rufus.ie/en/) — the signature must read **Akeo Consulting**. The portable `.exe` is enough.

| Rufus field | Value |
|---|---|
| Device | **The USB stick** — verify this twice |
| Boot selection | The ISO you just verified |
| Persistent partition size | **0** |
| Partition scheme | **GPT** |
| Target system | **UEFI (non CSM)** |
| File system | Large FAT32 (Rufus picks it) |
| Quick format | on · bad-block check: off |

Press **START**. If asked about ISOHybrid mode, choose **ISO image mode**, not DD. Wait for **READY**, then eject.

> A "Windows User Experience" prompt means you picked a Windows ISO by mistake.

---

## Step 3 — BIOS and Boot Menu

Boot menu is usually **F11 / F12 / F8 / Esc**; BIOS setup is **Del / F2**.

| Setting | What to do |
|---|---|
| Boot entry | Pick the one prefixed **UEFI:** — never `Legacy` or `CSM` |
| Secure Boot | Supported, but if `nvidia-smi` is empty later, **disabling it** is the quickest fix |
| Intel RST / RAID | Ubuntu needs AHCI. If RST is on and Windows is installed, research your board before switching — it can break Windows |

Boot the USB and choose **Try or Install Ubuntu**. Do not launch the installer yet — do Step 4 first.

---

## Step 4 — Identify the Target Disk

A wrong disk choice destroys Windows, and two identical SSDs look the same in the installer's dropdown. Identify the disk from a terminal in the live session (`Ctrl+Alt+T`) **before** starting the installer.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MODEL        # disks and their partitions
sudo lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,ROTA # physical disks, model + serial
```

Typical output:

```text
NAME        SIZE TYPE FSTYPE LABEL    MODEL
sda         1.8T disk        —        <disk model>
├─sda1      100M part vfat   SYSTEM
├─sda2       16M part
└─sda3      1.8T part ntfs   Windows
sdb       465.8G disk        —        <disk model>
nvme0n1   931.5G disk        —        <disk model>
└─nvme0n1p1 931.5G part ntfs Games
```

How to read it:

| What you see | What it is | Decision |
|---|---|---|
| Small `vfat` (~100–500 MB) **+** large `ntfs` | Windows system disk — that `vfat` is the Windows EFI partition | **Never select** |
| `ntfs` partitions, no EFI partition | Data / game / backup drive | **Never select** |
| No partitions, or ones you emptied yourself | Your target | ✅ This one |
| `ROTA` = `1` | A spinning HDD, not an SSD | Avoid |

Cross-check before committing:

```bash
sudo blkid | grep -i ntfs    # which partitions hold Windows filesystems
sudo fdisk -l /dev/sdb       # full partition table of your candidate
```

If `fdisk -l` shows an `EFI System` or `Microsoft reserved` partition, that is a Windows disk — stop and re-read the list.

Write down the exact device, size and model (e.g. `/dev/sdb, 465.8G`). You will match that string on the installer's summary screen.

> ⚠️ Two identical disks? Use the `SERIAL` column, or physically unplug the one you must not touch. Guessing is not an option.

---

## Step 5 — Install Ubuntu

Start the installer, pick language and keyboard, plug in Ethernet, choose **Interactive installation**, and tick **Install third-party software / NVIDIA drivers**.

At the disk step:

- Choose **Erase disk and install Ubuntu** and point it at the device from Step 4 — nothing else.
- **Never** use *Install alongside Windows*.
- Leave off: TPM / full-disk encryption (it can conflict with NVIDIA drivers) and ZFS.

On the summary screen, match the model and size against your note. **If the string does not match exactly, stop.**

Set a username, machine name and strong password. Automatic login is convenient on a dedicated mining box; keep it off on a laptop. Remove the USB and reboot.

> If GRUB landed on the Windows EFI partition, set the Windows drive first in the BIOS boot order and pick Ubuntu from the boot menu. Cosmetic annoyance — never "fix" it by erasing a disk.

---

## Step 6 — Updates and the NVIDIA Driver

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq
```

If the installer did not provide the driver:

```bash
sudo ubuntu-drivers autoinstall
sudo reboot
```

Then confirm it is live:

```bash
nvidia-smi
nvidia-smi -L    # one line per GPU — this count becomes --gpu-devices
```

You want the card name, driver version and a `CUDA Version` column. Low utilisation is expected — nothing is mining yet.

If it is empty or missing: disable Secure Boot and reboot → or run `ubuntu-drivers devices` and install the recommended driver explicitly → on a 50-series card, confirm the driver is 570+.

> You do **not** need `cuda-toolkit`. The miners carry their own CUDA runtime; a working `nvidia-smi` is all that matters.

---

## Step 7 — Disable Suspend

A blank screen is fine — **suspend stops the GPU**. Run these as your desktop user inside a graphical session:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

`idle-delay 300` blanks the display after five minutes; `0` never blanks it.

---

## Step 8 — Final Check

This is the end of this document. Tick every box before you leave it:

- [ ] Ubuntu is on the disk from Step 4 only, and Windows still boots
- [ ] `nvidia-smi` shows card, driver and CUDA version
- [ ] You know how many GPUs `nvidia-smi -L` reports
- [ ] Suspend is disabled
- [ ] `apt` reaches the internet
- [ ] No seed phrase was typed on this machine

✅ All ticked → continue at **[guide.md → Step 1: Create a Wallet](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-1--create-a-wallet)**.

That wallet plus a quick GPU re-check in Step 2 are the shared setup for both routes; the [Decision Point](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#decision-point--pick-your-route) then sends you to **Part A** (pool, easiest at home) or **Part B** (your own node). If you want to read the trade-off before you get there: [The Two Routes](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#the-two-routes--a-or-b).

---

## Troubleshooting

| Problem | What to check |
|---|---|
| USB will not boot | Pick the **UEFI:** entry in the boot menu; rewrite with GPT + UEFI (non CSM) |
| ISO checksum mismatch | Re-download from ubuntu.com |
| Unsure which disk to erase | Re-run Step 4. Identical disks → use `SERIAL` or unplug one |
| Installer refuses the disk | Intel RST/RAID is on, or it is part of a Windows dynamic volume |
| Windows seems gone | Set the Windows drive first in the boot order. **Erase nothing** |
| `nvidia-smi` empty or missing | `sudo ubuntu-drivers autoinstall`, disable Secure Boot, reboot |
| Driver fine but hash rate low later | Not a driver issue — see [the benchmark table](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-a2--benchmark-the-card) |
| Mining stops when idle | Re-apply Step 7 inside a graphical session |
| No Wi-Fi after install | Use Ethernet, or `sudo ubuntu-drivers autoinstall` + reboot |

Official walkthrough: [Install Ubuntu Desktop](https://ubuntu.com/tutorials/install-ubuntu-desktop).

---

## About the Author

Prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
