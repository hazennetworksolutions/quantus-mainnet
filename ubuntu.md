<div align="center">

# 🐧 Ubuntu Install for Quantus Mining

**Turn a Windows PC with an NVIDIA card into a Linux mining machine — without losing Windows**
*USB stick, dual boot on a spare SSD, GPU driver, then hand off to the mining guide.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/download/desktop)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Purpose:** prepare the operating system only — no mining software is installed here
> **Ends at:** [guide.md → Step 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-1--create-a-wallet)
> **Last Updated:** September 2026

---

## Why Bother

The same card mines **4–6× faster** under Linux CUDA than with the Windows miner. An RTX 4090 goes from ~179 MH/s to over 1.1 GH/s. The install below takes about an hour; it pays for itself on day one.

This document ends where the mining begins. When you reboot into a working Ubuntu desktop with `nvidia-smi` printing your card, continue with **[guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md)** and pick pool mining (Part A) or your own node (Part B).

**You need:** a Windows PC with an NVIDIA GPU, an 8 GB+ USB stick, and either a **spare SSD** or an empty partition. Renting a GPU instead? Skip all of this and read [vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md).

---

## Pick a Version

| Your GPU | Install |
|---|---|
| RTX 20 / 30 / 40 series | **Ubuntu 24.04 LTS** — the safe default |
| RTX 50 series (Blackwell) | **Ubuntu 26.04 LTS** — newer kernel, driver 570+ |

Always choose **Desktop LTS**. Non-LTS releases lose support in nine months, and Server adds nothing useful for a mining box.

---

## Step 1 — Download and Verify the ISO

Download the Desktop ISO from [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop), then verify it in PowerShell:

```powershell
Get-FileHash -Algorithm SHA256 "$env:USERPROFILE\Downloads\ubuntu-24.04-desktop-amd64.iso"
```

Compare the output with the `SHA256SUMS` file published next to the download. A mismatch means a corrupted file — download again rather than installing it.

---

## Step 2 — Write the USB Stick

Use [Rufus](https://rufus.ie/) (portable build is fine).

| Setting | Value |
|---|---|
| Device | your USB stick — **check twice, it gets erased** |
| Boot selection | the Ubuntu ISO |
| Partition scheme | **GPT** |
| Target system | **UEFI (non CSM)** |
| File system | FAT32, default cluster size |
| Persistent partition size | **0** |

Write in **ISO mode** if Rufus asks. Keep persistence at zero: this stick is an installer, not a live system, and persistence causes confusing boot behaviour.

---

## Step 3 — Boot from USB

1. Reboot and open the firmware menu (`F2`, `F10`, `F12`, `Del` or `Esc`, depending on the vendor).
2. Pick the **UEFI** entry for your USB stick — not the legacy/CSM one.
3. Choose **Try or Install Ubuntu**.

If the USB does not appear, disable Fast Boot in the firmware, and consider disabling Secure Boot now: the proprietary NVIDIA driver needs either Secure Boot off or an enrolled MOK key.

---

## Step 4 — Identify the Target Disk

**This is the step that prevents a wiped Windows drive.** From the live session terminal:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MODEL
sudo lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,ROTA
sudo blkid | grep -i ntfs      # NTFS = Windows, do not touch
sudo fdisk -l
```

Write down the target device node (`/dev/nvme1n1`, `/dev/sdb`, …) by matching **model, serial and size**, not by guessing the letter. The disk holding your Windows NTFS partitions is off limits.

---

## Step 5 — Install

Run **Install Ubuntu** and choose:

- Normal installation
- **Install third-party software for graphics and Wi-Fi hardware** — tick this; it pulls the NVIDIA driver during setup
- Download updates while installing, if your connection allows

At the disk step:

| Situation | Choose |
|---|---|
| Empty spare SSD, Windows on another disk | **Erase disk and install Ubuntu**, then explicitly select that spare disk |
| One disk shared with Windows | **Install alongside Windows**, or use Manual to place `/` on free space |

> ⚠️ "Erase disk" erases the **selected** disk completely. Confirm the device node from Step 4 on the summary screen before continuing. If the summary mentions your Windows disk, stop and go back.

Set a username and password you will actually remember — every command later uses `sudo`. Remove the USB stick when prompted and reboot.

---

## Step 6 — Install the GPU Driver

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

After the reboot:

```bash
nvidia-smi
nvidia-smi -L
```

You should see the card name, the driver version and a `CUDA Version` column.

| Problem | Fix |
|---|---|
| `nvidia-smi` not found or no devices | Secure Boot is blocking the module — disable it in firmware or enroll the MOK key, then reboot |
| RTX 50-series not recognised | Needs driver 570+; install Ubuntu 26.04 or add the graphics-drivers PPA |
| Screen stuck at low resolution | Boot the previous kernel entry and rerun `ubuntu-drivers autoinstall` |

> You do **not** need the full `cuda-toolkit`. The prebuilt Quantus miners ship their own CUDA runtime; the driver is enough.

---

## Step 7 — Stop the Machine Sleeping

A suspended desktop mines nothing. GNOME:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

Also set **Settings → Power → Automatic Suspend: Off**. On a headless box, also mask system sleep:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

---

## Step 8 — Final Checklist

| Check | Command | Expected |
|---|---|---|
| Ubuntu version | `lsb_release -a` | 24.04 or 26.04 LTS |
| GPU visible | `nvidia-smi -L` | your card, by name |
| Windows still bootable | reboot into the firmware menu | Windows entry present |
| Disk space | `df -h /` | 100 GB+ free if you plan Part B |
| Network | `ping -c 3 quantus.com` | replies |
| No suspend | `gsettings get … sleep-inactive-ac-type` | `'nothing'` |

All six green? The machine is ready.

---

## Next Step

Continue with **[guide.md → Step 1 — Create a Wallet](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-1--create-a-wallet)**. Step 2 there repeats the `nvidia-smi` check, so you can move through it quickly, then choose:

- **Part A — pool mining:** one binary, one service, income within days.
- **Part B — your own node:** full sync, zero fees, whole block rewards.

Undecided? Part A is the reversible choice.

---

## About the Author

Prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
