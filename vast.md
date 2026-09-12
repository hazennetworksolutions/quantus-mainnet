<div align="center">

# ☁️ Quantus Mining on a Rented GPU (Vast.ai)

**Rent an NVIDIA GPU by the hour and run the Quantus pool miner on it — no hardware, no node, no seed phrase on the box**
*Instance selection, SSH keys, miner install, supervisor persistence, cost control and teardown — step by step.*

[![Vast.ai](https://img.shields.io/badge/Marketplace-Vast.ai-1A73E8?style=flat-square)](https://vast.ai)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/guides/mining/)
[![Mode](https://img.shields.io/badge/Mode-Pool%20PPLNS-orange?style=flat-square)](https://quanpool.com/)
[![Persistence](https://img.shields.io/badge/Service-supervisor-yellow?style=flat-square)](http://supervisord.org/)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Target:** a rented NVIDIA container on a GPU marketplace, billed per hour
> **Result:** `quanpool-miner` under supervisor, surviving reboots, no wallet material on the host
> **Last Updated:** September 2026

---

## Scope

| Stage | Document |
|---|---|
| Rent a GPU and mine on it | **This guide** |
| Mining concepts, the A/B decision, own-node route | → [guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#the-two-routes--a-or-b) |
| Use your own Windows PC instead | → [ubuntu.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.md) |
| Network overview and links | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

This sets up a plain NVIDIA Ubuntu image, `quanpool-miner`, and a supervisor program so the miner restarts by itself. It does **not** set up your own node, put your seed phrase anywhere, or change a rig you already run at home.

**How this relates to the main guide.** guide.md splits mining into two routes: **Part A** (pool only) and **Part B** (your own node). This document is a **self-contained Part A variant for rented hardware** — it replaces guide.md Step 2 and Steps A1–A6 with the container equivalents, because a rented box has no persistent identity, no static address and no reason to sync a chain.

```text
guide.md  Step 1   create a wallet          ← the one thing you still do there
      ↓
vast.md   Step 1 → 11                       ← everything else happens here
      ↓
vast.md   Cost Control and Teardown         ← where this route ends
```

**Starts here:** with a `qz…` address from [guide.md → Step 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-1--create-a-wallet). That is the only prerequisite.
**Ends at:** [Step 11](#step-11--verify-on-the-pool), when your worker shows up on the pool page — followed by [Cost Control and Teardown](#cost-control-and-teardown), which is not optional reading on metered hardware.
**Never touched here:** `quantus-node`, chain sync, inbound firewall rules, wormhole inner hash. If you want that route instead, you want your own machine and [Part B](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#part-b--your-own-node-official-path).

> ⚠️ **Nothing secret goes on a rented machine.** The box needs exactly two values: `qzYOURADDRESS.workername` and the pool's TLS pin. Your 24 words, wallet file and node inner hash stay off it. Assume the host operator can see every file and every process.

---

## Table of Contents

- [What a GPU Marketplace Is](#what-a-gpu-marketplace-is)
- [Step 1 — Account and Credit](#step-1--account-and-credit)
- [Step 2 — Choose the Right Card](#step-2--choose-the-right-card)
- [Step 3 — Pick an Image, Not a Template](#step-3--pick-an-image-not-a-template)
- [Step 4 — Search Filters and Renting](#step-4--search-filters-and-renting)
- [Step 5 — SSH Key and First Connection](#step-5--ssh-key-and-first-connection)
- [Step 6 — First Checks Inside the Container](#step-6--first-checks-inside-the-container)
- [Step 7 — Install the Miner](#step-7--install-the-miner)
- [Step 8 — Token and TLS Pin](#step-8--token-and-tls-pin)
- [Step 9 — First Manual Run](#step-9--first-manual-run)
- [Step 10 — Make It Persistent with supervisor](#step-10--make-it-persistent-with-supervisor)
- [Step 11 — Verify on the Pool](#step-11--verify-on-the-pool)
- [Cost Control and Teardown](#cost-control-and-teardown)
- [Security Checklist](#security-checklist)
- [Troubleshooting](#troubleshooting)

---

## What a GPU Marketplace Is

Vast.ai is a marketplace where independent hosts rent out GPUs by the hour. You pay from a prepaid balance, a container starts on someone else's machine, and you connect over SSH. Most instances are **containers, not virtual machines**: you appear to be `root`, but you cannot load kernel modules or run Docker inside.

Three consequences shape this whole guide:

1. **Billing is continuous.** A running instance consumes credit whether the GPU is busy or idle. Destroying it is the only thing that stops the meter.
2. **`systemd` usually does not work.** Long-running processes go under the image's **supervisor**.
3. **Storage is disposable.** Without an attached volume, **Destroy deletes everything** — miner, token file, logs.

Rented mining only makes sense as pool mining: a rented box has no persistent identity, no static address and no reason to sync a chain, so [Part B of guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#part-b--your-own-node-official-path) is not the route here.

---

## Step 1 — Account and Credit

1. Create an account at [vast.ai](https://vast.ai) (console also at [cloud.vast.ai](https://cloud.vast.ai)).
2. Under **Billing**, add credit. Your runway is `balance ÷ hourly price`.
3. Two console pages matter: **Search** (offers you can rent) and **Instances** (boxes you are paying for).

Enable two-factor authentication and use a password you do not reuse — a compromised marketplace account is a direct financial loss.

---

## Step 2 — Choose the Right Card

Quantus QPoW (Poseidon2) is **compute-bound, not VRAM-bound**: 8–12 GB is plenty. This is the most expensive misunderstanding on a rented GPU — an 80–140 GB datacenter card is not "tens of times faster" here, and costs several times more per hour. Consumer cards usually win decisively on cost per hash.

Approximate Linux CUDA throughput, for sizing only — confirm your own instance with `benchmark`:

| Card | Approximate hash rate | Note |
|---|---|---|
| RTX 4090 | ~1.1 GH/s | Usually the best cost per hash |
| RTX 5090 | above a 4090 | Check the hourly price before assuming it wins |
| RTX 4070 / 5070 | ~410–460 MH/s | Two roughly match one 4090 |
| RTX 3080 / 3080 Ti | ~430 MH/s | 10–12 GB is sufficient |
| H100 / H200 / B200 | a 4090 to a few times that | VRAM is wasted; hourly cost rarely justifies it |

> **"Rent a monster for an hour and find my own block"** does not work. Against a network measured in TH/s, a few hours of one box is a lottery ticket. PPLNS pays you for shares instead, so the same rented hash rate earns steadily.

Unverified hosts at very low prices are sometimes genuinely cheap and sometimes unusable — an image stuck in **Loading**, an HDD, 40 Mbps of bandwidth. On a first attempt prefer **Verified** hosts with NVMe, 200+ Mbps and a recent CUDA driver. If an instance misbehaves, destroy it and take another offer; you are out a few cents.

---

## Step 3 — Pick an Image, Not a Template

Do **not** select an LLM, ComfyUI or CUDA-devel template. They pull tens of gigabytes of layers you will never use, and you pay while they download.

You want a **plain NVIDIA Ubuntu base image** with the driver injected by the host — typically the marketplace's own base or Jupyter CUDA image. The miner is a single ~13 MB download you add yourself.

| Setting | Choice |
|---|---|
| Template | None / the plain base image |
| Instance disk | 16–32 GB is plenty |
| Volume | Not required — but without one, Destroy erases everything |
| GPU count | Whatever the offer provides; pass the same number as `--gpu-devices` |

A Jupyter interface may be running. You do not have to use it, and you should not expose anything else through it.

---

## Step 4 — Search Filters and Renting

Narrow the Search list before renting:

- **GPU model:** the consumer cards from Step 2.
- **Number of GPUs:** start with **1×**. A two-card box costs about twice as much and delivers about twice the hash rate — there is no discount for learning on a big box.
- **Disk:** at least ~16 GB.
- **Verified / NVMe / bandwidth:** as above.

Read the price as **$/hour**: `$0.17/hr × 24 ≈ $4.10/day`. Prices far below market for a given card deserve suspicion rather than excitement.

Press **Rent**, then watch **Instances**. **Loading** means the image is still being pulled; if it is still loading after 10–15 minutes, destroy it and pick another offer. A ready instance shows status **running** with the GPU near 0% — nothing is mining yet.

---

## Step 5 — SSH Key and First Connection

The marketplace authenticates with an **SSH public key**, not a password. Generate a dedicated key — do not reuse the one for your personal servers:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/vast_quantus -N '' -C 'vast-quantus'
chmod 600 ~/.ssh/vast_quantus
cat ~/.ssh/vast_quantus.pub
```

Copy the printed `ssh-ed25519 AAAA…` line — that is the **public** half and is safe to paste into the console. The extensionless file (`vast_quantus`) is the private key and never leaves your machine.

In **Instances**, open the SSH / key panel, paste the public key into the *add key* field, and confirm it appears in the list. Give it a few seconds to propagate.

The panel also shows a connection line shaped like `ssh -p <PORT> root@<HOST_IP> -L 8080:localhost:8080`. The port and address differ for every instance, and the `-L 8080` tunnel is for Jupyter, not mining. Connect with your dedicated key:

```bash
ssh -p <PORT> -i ~/.ssh/vast_quantus \
  -o IdentitiesOnly=yes \
  -o StrictHostKeyChecking=accept-new \
  root@<HOST_IP>
```

On `Permission denied (publickey)`: wait a few seconds, confirm the key is listed on *that* instance, and check the path after `-i`. If the host's firewall blocks direct SSH, use the console's proxy SSH address. From Windows, the same command works in Windows Terminal.

---

## Step 6 — First Checks Inside the Container

```bash
nvidia-smi -L
nvidia-smi
```

Confirm the card count, driver version and CUDA version. `quanpool-miner` carries its own CUDA runtime, so any reasonably recent host driver works on Ada and Ampere; a 50-series (Blackwell) card needs CUDA 12.8+ on the host.

The image may print a welcome banner pointing at its own documentation — worth reading, because it tells you which supervisor layout the image uses. Treat it as a manual, not as instructions to run blindly.

One detail decides where you install: a path under `/workspace` survives a stop/start of the same container, but **not** a Destroy unless you attached a volume.

---

## Step 7 — Install the Miner

Confirm the current version and download link on [quanpool.com](https://quanpool.com/) → **Start mining**. At the time of writing the Linux CUDA build is **6.2.0**.

```bash
mkdir -p /workspace/quantus
cd /workspace/quantus
wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version

# optional local benchmark — never contacts the pool, but you are billed for the time
./quanpool-miner benchmark --cpu-workers 0 --duration 20
```

Compare the result against Step 2. Roughly 100 MH/s on a modern card means you are not on the CUDA path — see the table in [guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-a2--benchmark-the-card).

> The `gpu-list` subcommand was removed in 6.1+. Count cards with `nvidia-smi -L`.

---

## Step 8 — Token and TLS Pin

Use the **same `qz…` address** as your other machines and a **different worker name**. PPLNS shares from all your workers add up; duplicate names collide and one disappears from the pool.

Worker name rules: `a-z 0-9 . - _`, max 32 characters, no spaces. Avoid an obvious name like `pc1` on a shared host.

```bash
cd /workspace/quantus

# one single line: qzYOURADDRESS.uniqueworker
nano auth-token

# the 64-hex TLS pin from Start mining
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

The pin is the same for everyone on the pool, but copy it from the site rather than an old note — it changes when the pool rotates its certificate. Never pass these values inline with `--auth-token`; the file flags keep them out of the shell history.

Set `--gpu-devices` to the number of lines from `nvidia-smi -L`. Omitting the flag makes the miner try every card.

---

## Step 9 — First Manual Run

Take the live pool address from **Start mining** — it has the shape `IP:9834`:

```bash
cd /workspace/quantus
./quanpool-miner serve \
  --node-addr <POOL_HOST>:9834 \
  --auth-token-file /workspace/quantus/auth-token \
  --tls-cert-sha256-file /workspace/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
```

Healthy signs: GPU utilisation near 98%, `SHARE FOUND` in the log, and a populated response with zero rejects from `curl -s http://127.0.0.1:9900/hive-stats`.

Round-trip time from a rented datacenter can be higher than from home (300–400 ms is not unusual). That does not reduce your hash rate — the pool adjusts share difficulty. A continuous `timed out` loop is different: that host blocks outbound UDP/QUIC, so destroy the instance and rent in another region.

Stop with `Ctrl+C`, then move it under supervisor — otherwise the miner dies with your SSH session.

---

## Step 10 — Make It Persistent with supervisor

The base image's pattern is a script in `/opt/supervisor-scripts/` plus a matching config in `/etc/supervisor/conf.d/`.

```bash
cat > /opt/supervisor-scripts/quanpool-miner.sh << 'EOF'
#!/bin/bash
utils=/opt/supervisor-scripts/utils
. "${utils}/logging.sh"
. "${utils}/environment.sh"
cd /workspace/quantus
exec /workspace/quantus/quanpool-miner serve \
  --node-addr <POOL_HOST>:9834 \
  --auth-token-file /workspace/quantus/auth-token \
  --tls-cert-sha256-file /workspace/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
EOF
chmod 755 /opt/supervisor-scripts/quanpool-miner.sh
```

Set `<POOL_HOST>` to the live address and `--gpu-devices` to your card count before continuing.

> Do not source the image's portal-exit helper in this script. It can decide that a program missing from the portal list should be skipped, which silently stops your miner from starting.

```bash
cat > /etc/supervisor/conf.d/quanpool-miner.conf << 'EOF'
[program:quanpool-miner]
environment=PROC_NAME="%(program_name)s"
command=/opt/supervisor-scripts/quanpool-miner.sh
autostart=true
autorestart=true
startsecs=8
stopasgroup=true
killasgroup=true
stopsignal=TERM
stopwaitsecs=15
stdout_logfile=/var/log/portal/quanpool-miner.log
redirect_stderr=true
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=3
EOF

supervisorctl reread
supervisorctl update
supervisorctl status quanpool-miner
```

The state goes `STARTING` → `RUNNING` after a few seconds; CUDA warm-up takes 10–20 seconds. Day-to-day:

```bash
supervisorctl status quanpool-miner
supervisorctl restart quanpool-miner
tail -f /var/log/portal/quanpool-miner.log
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw --format=csv
```

Leave the image's own management programs (portal, tunnel manager, reverse proxy) running — they are how the console reaches the box.

> **Never publish port `9900`.** Metrics are for you, over SSH, on localhost. Opening a port on a rented box exposes it to the whole internet.

---

## Step 11 — Verify on the Pool

Open [quanpool.com](https://quanpool.com/), paste your `qz…` address and press **Look up**. The new worker appears within a few minutes, alongside any rigs you already run.

`hs` is reported in **kH/s** — `440000` ≈ 440 MH/s. A two-card instance shows two GPU rows, and rejects should stay at zero. A burst of `SOLUTION LOST` / stale messages in the first seconds is normal; after a few minutes you should see steady `SHARE FOUND` lines.

### Finish Line

The rented route is complete when all five are true:

1. `supervisorctl status quanpool-miner` reports `RUNNING`.
2. `nvidia-smi` shows 95–100% utilisation at roughly the hash rate you expected from Step 2.
3. `hive-stats` shows accepted shares climbing and rejects at zero.
4. The worker is visible on the pool lookup page under your `qz…` address, with a name no other machine uses.
5. You know the hourly price and have decided when you will Destroy the instance.

There is nothing further to install. What remains is money management, below.

---

## Cost Control and Teardown

Your spend is the hourly price multiplied by the time the instance exists.

| Action | What happens |
|---|---|
| Stop the miner (`supervisorctl stop`) | GPU goes idle, **you are still billed** |
| **Stop** the instance | Container stops; billing and disk retention depend on the host's policy |
| **Destroy** the instance | Instance and disk deleted, billing ends. Miner, token and logs are gone |
| **Reboot** | With `autostart=true`, supervisor brings the miner back |

When you are finished, **Destroy**. A forgotten instance quietly eats your balance, and stopping the miner alone does not stop the bill.

Renting the same offer again is a fresh install unless you attached a volume — run this guide from Step 5. Reusing the worker name is fine once the old instance is gone.

---

## Security Checklist

- [ ] No seed phrase, wallet file or node inner hash anywhere on the instance
- [ ] A dedicated SSH key for rented boxes, not your personal server key
- [ ] `auth-token` and `tls-cert-sha256` are mode `600`, passed by file and never inline
- [ ] Port `9900` and the miner port are not exposed publicly
- [ ] The worker name differs from every other machine you run
- [ ] You know the hourly price, and you will Destroy when finished

---

## Troubleshooting

| Symptom | What to check |
|---|---|
| Instance stuck on **Loading** for 15+ minutes | Slow host storage or a huge image. Destroy, take a Verified NVMe offer |
| `Permission denied (publickey)` | Key added to *that* instance, correct path after `-i`, wait a few seconds |
| Direct SSH never connects | Host firewall — use the console's proxy SSH address |
| `nvidia-smi` missing, or no GPU listed | Wrong image, or the GPU was not passed through. Recreate elsewhere |
| CUDA or PTX error | Host driver too old, or a 50-series card with an older build. Try the current miner or another offer |
| Endless `timed out` / reconnecting | That host blocks outbound UDP on the pool port. Destroy and rent in another region |
| `certificate` / `fingerprint` error | Re-copy the 64-hex TLS pin from **Start mining** |
| Worker never appears on the site | Address in `auth-token` vs the one you looked up, worker name rules, allow 2–3 minutes |
| supervisor `RUNNING` but GPU at 0% | Read the log: token, pin or `--node-addr` is wrong. `supervisorctl tail quanpool-miner` |
| Around 100 MH/s on a strong card | Not the CUDA path — wrong binary, or wrong GPU flags |
| Home worker disappeared | You reused the home worker name. Rename this one and restart both |
| Miner dies when SSH closes | Still running in the foreground — finish Step 10 |

---

## Next Steps

| Goal | Document |
|---|---|
| Mining concepts, pool details, own-node route | [guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md) |
| Convert a Windows PC into a mining machine | [ubuntu.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.md) |
| Network facts and links | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
