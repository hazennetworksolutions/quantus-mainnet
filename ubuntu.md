<div align="center">

# 🐧 Windows PC → Ubuntu Desktop for Quantus Mining

**Install Ubuntu Desktop on a spare disk without losing Windows, and get the NVIDIA driver working**
*ISO verification, Rufus USB, UEFI boot, identifying the right disk with `lsblk`, proprietary driver, sleep settings — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/download/desktop)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/guides/mining/)
[![Boot](https://img.shields.io/badge/Boot-UEFI%20%C2%B7%20GPT-blue?style=flat-square)](https://rufus.ie/en/)
[![Dual Boot](https://img.shields.io/badge/Windows-Preserved-0078D4?style=flat-square&logo=windows&logoColor=white)](https://ubuntu.com/tutorials/install-ubuntu-desktop)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Target:** a Windows 10/11 desktop with an NVIDIA GPU and a spare SSD
> **Result:** bare-metal Ubuntu Desktop with a working CUDA driver, Windows untouched
> **Last Updated:** September 2026

---

## Scope

This document does **one** job: it turns a Windows machine into a working Ubuntu Desktop machine with a live NVIDIA driver. It installs no miner and touches no wallet.

| Stage | Document |
|---|---|
| Install Ubuntu and the GPU driver | **This guide** |
| Install the miner, configure the pool or your own node | → **[guide.md](guide.md)** |
| Rent a GPU by the hour instead of using your own PC | → [vast.md](vast.md) |
| Network overview and links | [readme.md](readme.md) |

When you reach the end of Step 8 here, continue at [guide.md → Part A](guide.md#part-a--pool-mining-quanpool-pplns).

---

## Table of Contents

- [Why Ubuntu and Not Windows](#why-ubuntu-and-not-windows)
- [What You Need](#what-you-need)
- [Step 1 — Download and Verify the ISO](#step-1--download-and-verify-the-iso)
- [Step 2 — Write the USB with Rufus](#step-2--write-the-usb-with-rufus)
- [Step 3 — BIOS and Boot Menu](#step-3--bios-and-boot-menu)
- [Step 4 — Identify the Target Disk](#step-4--identify-the-target-disk)
- [Step 5 — Install Ubuntu](#step-5--install-ubuntu)
- [Step 6 — Updates and the NVIDIA Driver](#step-6--updates-and-the-nvidia-driver)
- [Step 7 — Disable Suspend](#step-7--disable-suspend)
- [Step 8 — Confirm the Machine Is Ready](#step-8--confirm-the-machine-is-ready)
- [Thermals and Longevity](#thermals-and-longevity)
- [Home Network Notes](#home-network-notes)
- [Troubleshooting](#troubleshooting)

---

## Why Ubuntu and Not Windows

The fast CUDA code path for Quantus mining exists on Linux. On the same card, the Windows stock miner runs roughly **4–6× slower** — around 100 MH/s where Linux CUDA delivers 400–450 MH/s on a mid-range RTX card. That single factor is the entire reason for this guide.

Three ground rules before you touch anything:

1. **Bare metal, not WSL and not a virtual machine.** Install Ubuntu on a real disk with UEFI. GPU mining through WSL2 is not the path here.
2. **You do not have to delete Windows.** The cleanest layout is Windows on its own drive and Ubuntu on a separate, empty SSD. Avoid *Install alongside Windows*, which squeezes Ubuntu onto the Windows disk.
3. **Your 24-word phrase is never typed into this machine.** Wallet creation happens on your phone — see [guide.md → Step 1](guide.md#step-1--create-a-wallet).

---

## What You Need

| Item | Requirement |
|---|---|
| PC | Windows 10/11 booting in **UEFI** mode (Legacy/CSM off) |
| GPU | NVIDIA RTX 20/30/40/50 series. AMD cards do not work with the CUDA miner |
| Disk | A **separate SSD** for Ubuntu with nothing valuable on it — the installer erases it |
| USB stick | 8 GB or larger. Rufus erases it, so back up its contents first |
| Network | Wired Ethernet preferred; the installer downloads driver packages |

Anything above the official floor (4+ cores, 8 GB+ RAM, an SSD) is enough. Hash rate comes from the **GPU**, not from disk capacity.

---

## Step 1 — Download and Verify the ISO

1. Go to [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) and take the **Intel/AMD 64-bit Desktop** LTS image. Do not download an ARM ISO.
2. Prefer the newest LTS for a recent GPU: RTX 50-series (Blackwell) needs a newer kernel and driver, so **26.04.x** causes less friction than 24.04. For 30/40-series cards, 24.04 LTS is fine.
3. Download the `SHA256SUMS` file published next to the ISO, then verify in PowerShell from the folder holding the ISO:

```powershell
Get-FileHash .\ubuntu-*-desktop-amd64.iso -Algorithm SHA256
```

Compare the output with the matching ISO line in `SHA256SUMS`.

> ⚠️ If the checksum does not match, **delete the ISO and download it again.** Do not write a corrupt image to USB.

---

## Step 2 — Write the USB with Rufus

1. Get Rufus from [rufus.ie](https://rufus.ie/en/) or the official `pbatard/rufus` releases page. The signature must read **Akeo Consulting**. No app stores, no mirror sites.
2. The portable `.exe` is enough; installation is not required.
3. Insert the USB stick. In Rufus, confirm **Device** is that USB stick — not your Windows disk, not the SSD you reserved for Ubuntu, not a data drive.
4. **Boot selection:** the `ubuntu-*-desktop-amd64.iso` you just verified.
5. Use these settings:

| Rufus field | Value |
|---|---|
| Persistent partition size | **0** (this is an installer USB, not a live system) |
| Partition scheme | **GPT** |
| Target system | **UEFI (non CSM)** |
| File system | Large FAT32 (Rufus picks this for a multi-GB ISO) |
| Cluster size | default |
| Quick format | enabled |
| Check device for bad blocks | disabled |

6. Press **START**. If Rufus asks about ISOHybrid mode, choose **ISO image mode**, not DD. Accept the erase warning only while the selected device is still the USB stick.
7. When the status returns to **READY**, close Rufus and eject the USB.

> If a Windows setup or "Windows User Experience" prompt appears during this process, you selected a Windows ISO by mistake.
>
> Do not store your notes or project files on this USB — Rufus wipes it. Use a second stick, or copy them from the Windows NTFS partition after Ubuntu is running.

---

## Step 3 — BIOS and Boot Menu

Keys vary by vendor: the boot menu is usually **F11 / F12 / F8 / Esc**, and BIOS setup is **Del / F2**.

1. Leave the USB inserted and restart.
2. In the boot menu choose the entry prefixed **UEFI:** for your USB device. Do **not** pick a `Legacy` or `CSM` entry — an Ubuntu installed in legacy mode will conflict with your UEFI Windows.
3. **Secure Boot:** current Ubuntu LTS supports it. If `nvidia-smi` later comes up empty, setting Secure Boot to **Disabled** is the lowest-friction fix, because it removes the kernel module signing / MOK enrollment problem entirely.
4. **Intel RST / RAID:** Ubuntu will not install onto an RST volume; disks must be in AHCI mode. Switching RST off blindly while Windows is installed can break Windows — if AHCI is already active, just continue; if not, research your specific board before changing it.

Boot the USB and choose **Try or Install Ubuntu**. Do not start the installer yet — do Step 4 first.

---

## Step 4 — Identify the Target Disk

A wrong disk choice destroys Windows, and the installer's own list is easy to misread — two identical SSDs look the same in a dropdown. So identify the disk from a terminal in the live session **before** launching the installer, and write down its device name.

Open a terminal in the live session (`Ctrl+Alt+T`) and list every disk with its partitions:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MODEL
```

Then list the physical disks only, with model and serial, so you can tell two same-size drives apart:

```bash
sudo lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,ROTA
```

A typical machine prints something along these lines:

```text
NAME   SIZE TYPE FSTYPE LABEL   MODEL
sda    1.8T disk        —       <disk model>
├─sda1  100M part vfat   SYSTEM
├─sda2   16M part
└─sda3  1.8T part ntfs   Windows
sdb  465.8G disk        —       <disk model>
nvme0n1 931.5G disk     —       <disk model>
└─nvme0n1p1 931.5G part ntfs  Games
```

Read it with these rules:

| What you see on a disk | What it is | Decision |
|---|---|---|
| A small `vfat` partition (~100–500 MB) plus a large `ntfs` partition | The Windows system disk — that `vfat` partition is the Windows EFI partition | **Never select it** |
| One or more `ntfs` partitions with no EFI partition | A data, game or backup drive | **Never select it** |
| No partitions at all, or partitions you know you emptied yourself | The disk you reserved for Ubuntu | This is your target |
| `ROTA` column shows `1` | A spinning hard disk, not an SSD | Avoid — usable for Ubuntu, but poor for a node |

Cross-check before you commit. Any of these three confirms the picture:

```bash
# which partitions carry a Windows filesystem
sudo blkid | grep -i ntfs

# is there a Windows installation on a mounted partition
ls /mnt 2>/dev/null; sudo mount | grep -i ntfs

# full partition table of one specific disk, e.g. the candidate target
sudo fdisk -l /dev/sdb
```

On the candidate target, `fdisk -l` should report either no partition table at all or only partitions you recognise as disposable. If it lists an `EFI System` partition or a `Microsoft reserved partition`, you are looking at a Windows disk — stop and re-read the list.

Write down the device name and its size and model, for example `/dev/sdb, 465.8G`. You will match that exact string on the installer's summary screen.

> ⚠️ If two disks are the same size and the same model, use the `SERIAL` column to tell them apart, or unplug the drive you must not touch before installing. Guessing is not an option here.

---

## Step 5 — Install Ubuntu

Start the installer from the live desktop, pick your language and keyboard layout, and plug in Ethernet.

Choose **Install Ubuntu** → *Interactive installation*. Tick **Install third-party software / NVIDIA drivers** and the media codecs box.

When you reach the disk step:

- Select **Erase disk and install Ubuntu**, and point it at the device you identified in Step 4 — nothing else.
- **Do not use "Install alongside Windows."** The goal is one operating system per physical disk.
- Leave these off for a mining install: TPM / experimental full-disk encryption (it can conflict with NVIDIA drivers), ZFS, and any option that formats the Windows or data drives.

On the summary screen, match the model and size of the drive to be erased against what you wrote down. If the string does not match exactly, stop — do not press Install.

Set your username, machine name and a strong password. Automatic login is convenient on a dedicated mining box (the miner and the desktop come up without you unlocking anything); keep it off on a laptop.

Remove the USB and reboot. From now on, the boot menu offers the Ubuntu disk for Linux and the Windows drive for Windows.

> If GRUB was written to the Windows EFI partition, set the Windows drive first in the BIOS boot order and select Ubuntu from the boot menu when you want it. This is a cosmetic annoyance — never "fix" it by erasing the Windows disk.

---

## Step 6 — Updates and the NVIDIA Driver

Open a terminal (`Ctrl+Alt+T`):

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq
```

If the installer did not already provide the NVIDIA driver:

```bash
sudo ubuntu-drivers autoinstall
sudo reboot
```

After the reboot, confirm the driver is live:

```bash
nvidia-smi
nvidia-smi -L
```

You should see the card name, the driver version and a `CUDA Version` column. Low utilisation is expected — nothing is mining yet. Note how many lines `nvidia-smi -L` prints; that is the number you will pass as `--gpu-devices`.

If the output is empty or the command does not exist:

- Disable Secure Boot in the BIOS and reboot.
- Run `ubuntu-drivers devices` and install the recommended driver explicitly, then reboot.
- On a 50-series card, make sure the kernel and driver are recent enough (570+).

> You do **not** need the full `cuda-toolkit` package. The prebuilt miners carry their own CUDA runtime. All that is required is the proprietary driver and a working `nvidia-smi`.

---

## Step 7 — Disable Suspend

A miner keeps running with a blank screen or an active screensaver. **Suspend stops the GPU**, so that is what has to go:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

`idle-delay 300` blanks the display after five minutes; use `0` to never blank it. Run these as your own desktop user, inside a graphical session.

---

## Step 8 — Confirm the Machine Is Ready

- [ ] Ubuntu is installed **only** on the disk you identified in Step 4, and Windows still boots
- [ ] `nvidia-smi` shows the card, the driver version and a CUDA version
- [ ] `nvidia-smi -L` prints one line per GPU and you know the count
- [ ] Suspend is disabled
- [ ] Ethernet works and `apt` can reach the internet
- [ ] No seed phrase has been typed on this machine

All ticked? Continue at **[guide.md → Part A — Pool Mining](guide.md#part-a--pool-mining-quanpool-pplns)** to install the miner, or **[Part B](guide.md#part-b--your-own-node-official-path)** to run your own node.

---

## Thermals and Longevity

Under Linux CUDA the card generally sits near its power limit with fans spun up. Sustained ~70 °C is normal mining behaviour on most consumer cards and stays clear of the ~83–88 °C throttle point.

- Dust removal and case airflow are the maintenance that actually matters.
- Fans run continuously and are usually the first mechanical part to fail.
- Overclocking and undervolting are optional. If the card holds 85 °C continuously, add `--gpu-throttle-ms 5` to the miner — you lose a little hash rate and gain headroom. The same flag keeps the desktop responsive while mining.

Monitor with:

```bash
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw,fan.speed --format=csv
```

---

## Home Network Notes

Pool mining speaks **UDP/QUIC** outbound to the pool port. Opening TCP 443 is irrelevant, and no inbound rule is needed at all.

| Port | Purpose | What to do at home |
|---|---|---|
| `9834/UDP` | Pool connection | **Outbound only.** If it is blocked, the miner loops on reconnect |
| `30333/TCP` | Node P2P | Only relevant if you run your own node |
| `9900` | Local miner metrics | Localhost only — never forward it |

- If your WAN address is in `100.64.0.0/10` you are behind **CGNAT**: forwarding ports at the router brings no inbound traffic. Pool mining does not need any.
- Some transit paths with DDoS scrubbing drop outbound QUIC, which looks like "Ethernet is broken but the phone hotspot works". Test with a hotspot to isolate it; report it to the pool, as it is usually transient.
- A static IP is not required. After a router reboot the miner simply reconnects.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| USB will not boot | Use the boot menu key and pick the **UEFI:** USB entry; rewrite with Rufus using GPT + UEFI (non CSM) |
| ISO checksum mismatch | Re-download from ubuntu.com; never install from a corrupt image |
| Unsure which disk to erase | Go back to Step 4 and re-run `lsblk` / `blkid`. If two disks are identical, use `SERIAL` or unplug the one you must keep |
| Installer will not use the disk | Intel RST/RAID is enabled, or the disk is part of a Windows dynamic volume |
| Windows seems gone | Set the Windows drive first in the BIOS boot order, or pick it from the boot menu. Do not erase anything |
| `nvidia-smi` missing or empty | `sudo ubuntu-drivers autoinstall`, disable Secure Boot, reboot |
| `nvidia-smi` works but the miner is slow later | Not a driver problem — see the benchmark table in [guide.md](guide.md#step-a2--benchmark-the-card) |
| Machine sleeps and mining stops | Re-apply the Step 7 settings inside a graphical session |
| No Wi-Fi after install | Prefer Ethernet; some Wi-Fi chips need `sudo ubuntu-drivers autoinstall` plus a reboot |

Official installer walkthrough: [Install Ubuntu Desktop](https://ubuntu.com/tutorials/install-ubuntu-desktop).

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
