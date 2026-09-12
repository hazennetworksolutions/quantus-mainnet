<div align="center">

# ☁️ Quantus Mining on a Rented GPU

**Mine QTC on hourly GPU rental — no hardware, no chain sync, no seed phrase on the box**
*Instance selection, SSH, CUDA miner, supervisor autostart, cost control, teardown.*

[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Pool](https://img.shields.io/badge/Mode-Pool%20PPLNS-orange?style=flat-square)](https://quanpool.com/)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Route:** pool mining only — the rented-GPU variant of [guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md) Part A
> **Versions:** pool miner 6.2.0
> **Last Updated:** September 2026

---

## What This Is

Renting a GPU by the hour (Vast.ai and similar marketplaces) lets you mine without owning hardware. You get a container with the NVIDIA driver already injected, you install one binary, and you pay per hour for as long as it runs.

**Pool mining only.** Running your own node on a rented box means paying by the hour to sync 100 GB of chain data that disappears when the instance is destroyed. If you want your own node, use hardware you keep — [guide.md Part B](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#part-b--your-own-node).

**Three rules for rented machines:**

1. Your 24-word phrase **never** goes on the box. The miner only needs your public `qz…` address.
2. Rent is charged whether or not the miner is running. Benchmark first, then decide.
3. Only **Destroy** stops billing. Stopping an instance still costs money for stored data.

**Prerequisites:** a `qz…` address ([guide.md Step 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md#step-1--create-a-wallet)) and the host address plus TLS pin from [quanpool.com](https://quanpool.com/) → **Start mining**.

---

## Step 1 — Create an SSH Key

On your own machine:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vast_quantus -C "quantus-mining"
cat ~/.ssh/vast_quantus.pub
```

Paste the public key into the marketplace's SSH-keys page. Never upload the private key anywhere.

---

## Step 2 — Pick an Instance

| Criterion | What to look for |
|---|---|
| GPU | RTX 4090 / 5090 for throughput; 3080 Ti / 4070 for cost per hash |
| Reliability | 99%+ — low-reliability hosts vanish mid-run |
| Image | any CUDA / PyTorch image with the driver preinstalled |
| Disk | 20 GB is plenty — you are not storing a chain |
| Internet | outbound UDP must work; avoid heavily filtered hosts |
| Price | compare $/hour against the card's benchmark, not its name |

A datacenter card with 80–140 GB of VRAM is **not** proportionally faster: QPoW barely uses VRAM, so a consumer 4090 usually wins on cost per hash.

Interruptible/spot instances are cheaper but can be paused at any moment. For mining that is acceptable — with supervisor autostart, work resumes when the instance does.

---

## Step 3 — Connect

```bash
ssh -i ~/.ssh/vast_quantus -p <PORT> root@<HOST>
nvidia-smi
nvidia-smi -L
```

If `nvidia-smi` fails, the host is broken — destroy the instance and rent another; do not troubleshoot someone else's driver stack by the hour.

> Do not source the portal's exit/cleanup helper scripts in your shell. They can terminate your session in some images.

---

## Step 4 — Install the Miner

Use `/workspace` when the image provides it — it is the persistent volume.

```bash
mkdir -p /workspace/quantus && cd /workspace/quantus
apt-get update -qq && apt-get install -y -qq wget curl ca-certificates netcat-openbsd jq

wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version
```

Take the current download link from **Start mining** if that version is gone.

---

## Step 5 — Benchmark Before Committing

This is the whole point of doing it in this order: the benchmark tells you whether the rental is worth its hourly price, and it never contacts the pool.

```bash
cd /workspace/quantus
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
```

| Result | Action |
|---|---|
| 400 MH/s – 1.5 GH/s depending on the card | Continue |
| ~100 MH/s on a modern RTX card | Wrong driver path on this host — destroy and rent elsewhere |
| Far below the card's published figure | The GPU is shared or throttled — destroy it |

---

## Step 6 — Credentials

From [quanpool.com](https://quanpool.com/) → **Start mining**, using your `qz…` address and a worker name unique to this instance:

```bash
cd /workspace/quantus

# single line: qzYOURADDRESS.vast01
nano auth-token

# the 64-hex TLS pin
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

> Never paste your seed phrase. The address is public information and is all the pool needs to credit you. Give every rented instance its own worker name, or two rigs will collide and one will vanish from the pool.

---

## Step 7 — First Manual Run

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

Healthy within a minute: GPU utilisation at 95–100%, `SHARE FOUND` lines in the log, and `hive-stats` reporting a hash rate with zero rejects. A burst of stale-job messages at startup is normal.

If it loops on `timed out`, outbound UDP is blocked on this host — destroy it and rent another. `Ctrl+C` once it looks healthy.

---

## Step 8 — Autostart with supervisor

Containers usually have no working `systemd`, so the systemd unit from guide.md does not apply. Most marketplace images ship **supervisor** instead.

```bash
mkdir -p /opt/supervisor-scripts

cat > /opt/supervisor-scripts/quanpool-miner.sh <<'EOF'
#!/usr/bin/env bash
cd /workspace/quantus || exit 1
exec ./quanpool-miner serve \
  --node-addr <POOL_HOST>:9834 \
  --auth-token-file /workspace/quantus/auth-token \
  --tls-cert-sha256-file /workspace/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
EOF

chmod +x /opt/supervisor-scripts/quanpool-miner.sh

cat > /etc/supervisor/conf.d/quanpool-miner.conf <<'EOF'
[program:quanpool-miner]
command=/opt/supervisor-scripts/quanpool-miner.sh
autostart=true
autorestart=true
startsecs=8
stdout_logfile=/var/log/portal/quanpool-miner.log
redirect_stderr=true
EOF

supervisorctl reread
supervisorctl update
supervisorctl status quanpool-miner
```

Edit `<POOL_HOST>` in the script before starting. `autorestart=true` brings the miner back after a crash or a resumed spot instance.

```bash
supervisorctl restart quanpool-miner
supervisorctl stop quanpool-miner
tail -f /var/log/portal/quanpool-miner.log
```

> No supervisor in the image? Run the miner inside `tmux` or `screen` and accept that a container restart needs a manual start.

---

## Step 9 — Verify

```bash
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw --format=csv
```

`hs` is usually kH/s (`430000` ≈ 430 MH/s) and `ar` is `[accepted, rejected]`; rejected should stay at zero. Then look your `qz…` address up on [quanpool.com](https://quanpool.com/) — the worker appears within a few minutes.

---

## Step 10 — Cost Control

Rent accrues by the hour regardless of hash rate, so the arithmetic is simple: **hourly rent versus the QTC that hash rate earns.** Check it in the first hour, not the first week.

- Note the measured hash rate from Step 5 and compare it against the pool's live network hash rate.
- Difficulty re-adjusts on every finalized block, so yesterday's estimate can be wrong today.
- Set a spending limit or credit alert in the marketplace account.
- Interruptible instances are cheaper; with autostart they suit mining well.
- Storage is billed even while an instance is stopped.

Rented GPU mining is only profitable when the card is cheap and the network hash rate is low. Treat every projection as unverified and re-check weekly.

---

## Step 11 — Teardown

1. `supervisorctl stop quanpool-miner`
2. Confirm your pending balance on the pool lookup page — it stays with your address, not the instance.
3. **Destroy** the instance in the marketplace UI. Stopping is not enough; only Destroy ends billing.
4. Remove the SSH key from the marketplace account if you are done for good.

Unclaimed pool balance is unaffected by destroying the instance: it is tied to your `qz…` address and pays out on the pool's normal schedule.

---

## Security Checklist

| Rule | Why |
|---|---|
| Seed phrase never touches the box | The host operator can read the filesystem |
| Only the `qz…` address and TLS pin live on it | Both are public or instance-specific |
| Secrets in `chmod 600` files, never inline | Shell history and process lists are readable |
| No inbound ports opened | Pool mining is outbound-only |
| `9900` stays on localhost | Miner metrics are not for the internet |
| One worker name per instance | Duplicate names make rigs collide |
| Destroy when finished | Stopped instances still bill for storage |

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `timed out` / endless reconnect | Outbound UDP blocked on the host — destroy and rent another |
| `certificate` / `fingerprint` error | Re-copy the 64-hex TLS pin from **Start mining** |
| Benchmark ~100 MH/s on a modern card | Host driver path is wrong — do not pay to debug it |
| `nvidia-smi` missing or no devices | Broken host image — destroy it |
| Worker not on the pool page | Address or worker name wrong in `auth-token`; allow 2–3 minutes |
| Miner gone after a restart | supervisor config missing or `autostart=false` |
| `systemctl` not found | Expected in containers — use supervisor |
| Session dies unexpectedly | Do not source the portal exit helper; use `tmux` |

---

## Next Steps

| Goal | Document |
|---|---|
| Full pool and own-node reference | [guide.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.md) |
| Move to your own hardware | [ubuntu.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.md) |
| Network facts and links | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

## About the Author

Prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
