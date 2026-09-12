<div align="center">

# ⚛️ Quantus Mainnet GPU Mining & Full Node Guide

**Mine QTC on Quantus mainnet — join a pool, or run your own node with an external GPU miner**
*Two complete routes, one shared setup: wallet, CUDA miner, systemd service, node sync, monitoring, troubleshooting.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Miner](https://img.shields.io/badge/Miner%20Protocol-quantus--miner%2F2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/miner-protocol/)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Quantus Mainnet (`--chain mainnet`)
> **Versions:** `quantus-node` v1.0.1+ · `quantus-miner` v4.2.x · pool miner 6.2.0
> **Last Updated:** September 2026

---

## Table of Contents

**Read first**

- [How Mining Works](#how-mining-works)
- [The Two Routes — A or B](#the-two-routes--a-or-b)
- [Hardware Requirements](#hardware-requirements)
- [Ports and Endpoints](#ports-and-endpoints)

**Shared setup — everybody does this**

- [Step 1 — Create a Wallet](#step-1--create-a-wallet)
- [Step 2 — Verify the GPU (Linux)](#step-2--verify-the-gpu-linux)
- [Decision Point — Pick Your Route](#decision-point--pick-your-route)

**Then one route, not both**

- [Part A — Pool Mining (Quanpool PPLNS)](#part-a--pool-mining-quanpool-pplns)
  - [A1 — Install the Pool Miner](#a1--install-the-pool-miner)
  - [A2 — Benchmark the Card](#a2--benchmark-the-card)
  - [A3 — Token and TLS Pin](#a3--token-and-tls-pin)
  - [A4 — First Manual Run](#a4--first-manual-run)
  - [A5 — systemd Service](#a5--systemd-service)
  - [A6 — Verify on the Pool](#a6--verify-on-the-pool)
  - [Part A Finish Line](#part-a-finish-line)
- [Part B — Your Own Node](#part-b--your-own-node)
  - [B1 — Automated Setup Script](#b1--automated-setup-script)
  - [B2 — Manual Install: Binaries](#b2--manual-install-binaries)
  - [B3 — Node Identity and Wormhole Inner Hash](#b3--node-identity-and-wormhole-inner-hash)
  - [B4 — Start the Node](#b4--start-the-node)
  - [B5 — Start the External Miner](#b5--start-the-external-miner)
  - [B6 — Updating a Node/Miner Pair](#b6--updating-a-nodeminer-pair)
  - [Part B Finish Line](#part-b-finish-line)

**Reference — both routes**

- [Monitoring and Everyday Commands](#monitoring-and-everyday-commands)
- [Performance Reference](#performance-reference)
- [Firewall](#firewall)
- [Running Multiple Machines](#running-multiple-machines)
- [Troubleshooting](#troubleshooting)
- [Economics — What to Expect](#economics--what-to-expect)
- [Switching Routes Later](#switching-routes-later)

---

## How Mining Works

Quantus mining always involves **two jobs**, and the split explains everything that follows:

1. **The node** (`quantus-node`) connects to mainnet, keeps a full copy of the chain, assembles block candidates and hands out mining jobs. Started with `--miner-listen-port`, it also becomes a QUIC server on port `9833`.
2. **The miner** does nothing but search for a winning nonce, on GPU or CPU. The miner is always the *client*: it dials the node, never the reverse.

The algorithm is **QPoW** — double Poseidon2 hashing instead of SHA-256. A nonce wins when `Poseidon2(Poseidon2(block_hash ‖ nonce))` lands below the difficulty target. Poseidon2 was chosen because it is cheap to verify inside a ZK circuit, not because SHA-256 is weak.

The node's built-in CPU miner exists for testing — roughly **15 MH/s per thread**. Real throughput comes from an external GPU miner, **500 MH/s – 1.5 GH/s** per modern card. Several miners may connect to one node; the first valid result wins.

**The key question is who runs job 1.** Run the node yourself and you own the whole pipeline. Join a pool and the operator runs it for you, while your miner dials out to `host:9834` over UDP — which is why pool mining works behind CGNAT with no port forwarding, no static IP and no chain sync.

Either way, rewards never land on a regular address. The protocol pays a **wormhole address** derived from a 32-byte preimage called your *inner hash*. Derive it from the same 24 words as your wallet app and the rewards appear there, spendable, with no claim transaction.

### Chain facts worth knowing

| Parameter | Value |
|---|---|
| Algorithm | QPoW — double Poseidon2 |
| Address format | SS58 prefix 189 — addresses start with `qz…` |
| Max supply | 21,000,000, 12 decimals |
| Difficulty adjustment | Every finalized block, clamped per block — no 2016-block epochs |
| Fork choice | Heaviest chain by cumulative work, not longest |
| Finalization | 179 blocks behind the tip (max reorg depth 180) |
| Block reward | `(MaxSupply − CurrentSupply) / EmissionDivisor` — smooth decay, no halvings |

> ⚠️ Always pass `--chain mainnet`. `planck` is the retired testnet: separate chain, separate database, no balance migration. Never copy `chains/planck/` into `chains/mainnet/`, and never pass `--force-authoring` on mainnet — that flag is for bootstrapping a brand-new network.

---

## The Two Routes — A or B

Two mutually exclusive halves. **Read this before installing anything**: the choice changes what you download, what you open on your firewall and how you get paid.

### Part A — Pool Mining

You run **only a miner**. It connects outbound to a pool operator's node, works the jobs that node sends, and submits shares. The pool finds blocks and splits rewards by share count (PPLNS).

- **You install:** one binary, one systemd service.
- **You need:** a `qz…` address, a GPU, outbound UDP.
- **You do not need:** a synced chain, disk space, open inbound ports, a public IP.
- **You get paid:** small amounts, continuously, once past the payout threshold.
- **Time to first hash:** 15–30 minutes.

### Part B — Your Own Node

You run **both jobs on your own hardware**: `quantus-node` syncing mainnet, plus `quantus-miner` pointed at it on `127.0.0.1:9833`.

- **You install:** two matched binaries, node identity, wormhole inner hash.
- **You need:** 100 GB+ SSD, stable bandwidth, patience for the initial sync.
- **You get paid:** the **whole** block reward when your node wins — and nothing in between.
- **You also get:** zero fees, full custody, no trust in an operator, and a node that strengthens the network.
- **Time to first hash:** a few hours, mostly sync time.

### Side by side

| | **Part A — Pool** | **Part B — Own node** |
|---|---|---|
| Processes to run | 1 (miner) | 2 (node + miner) |
| Chain sync | none | full sync required |
| Disk | negligible | 100 GB+ SSD, HDD will not do |
| Works behind CGNAT | yes, by design | yes, outbound peering is enough |
| Inbound ports | none | `30333/TCP` optional |
| Income shape | steady trickle | rare lumps, high variance |
| Fees | pool fee, published on the pool site | none |
| Trust assumption | the operator pays honestly | none |
| Good for | one or two consumer cards at home | a dedicated rig, or a node you want anyway |

### Which one should you pick?

Pick **Part A** if any of these is true — and at home, usually one is:

- You have one or two consumer GPUs rather than a farm.
- You are behind CGNAT, a mobile connection, or a router you do not control.
- You want income within the week instead of waiting for luck.
- You do not want a process that has to stay synced.

Pick **Part B** if:

- You wanted to run a Quantus full node anyway.
- You have enough hash rate that block variance is tolerable.
- You refuse to route rewards through a third party.
- You are on a server with a real IP, an SSD and no bandwidth cap.

> **The decision is not permanent.** Wallet, address and GPU work are identical in both routes — see [Switching Routes Later](#switching-routes-later).

### How this guide is laid out

```text
  Step 1  Create a wallet          ─┐
  Step 2  Verify the GPU            ├─ everybody does these two
                                    │
  ── Decision Point ──────────────┘
         │
         ├── Part A  A1 → A6   ends at "Part A Finish Line"
         │
         └── Part B  B1 → B6   ends at "Part B Finish Line"
                 │
  Reference sections apply to whichever route you finished
```

**Do exactly one part.** Both on one machine means two miners fighting over one GPU; neither performs.

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Operating System | Ubuntu 20.04+, macOS, Windows 10/11 | Ubuntu 24.04 / 26.04 LTS |
| CPU | 2 cores | 4+ cores |
| RAM | 4 GB | 8 GB+ |
| Disk | 100 GB *(Part B only)* | 500 GB+ SSD — SATA is fine, HDD is not |
| Network | 3 Mbps | 10+ Mbps |
| GPU | none (CPU only, very slow) | NVIDIA RTX 20/30/40/50 series |

- Hash rate is decided by the **GPU and the miner build**, not disk size or CPU cache. A large-cache CPU does not accelerate Poseidon2.
- QPoW is **not VRAM-hungry**. 8–12 GB is plenty; a 140 GB datacenter card is not "50× faster" and usually loses on cost per hash to a consumer 4090.
- The node database is RocksDB and does random I/O. Any SSD works; an HDD stalls the sync.
- **Linux ARM64 has no official miner binary** — mine from Linux x86_64 or macOS. AMD GPUs do not work with the CUDA pool miner.
- **Part A needs none of the disk or bandwidth headroom above** — only the GPU and outbound UDP.

---

## Ports and Endpoints

| Port | Purpose | Route | What to do |
|---|---|---|---|
| `30333/TCP` | Node P2P | B | The only port that may face the internet. Optional — outbound peering syncs fine |
| `9833/UDP` | Miner ↔ your own node (QUIC) | B | **Localhost or VPN only.** It binds `0.0.0.0`; only your firewall protects it |
| `9834/UDP` | Pool miner → pool node | A | **Outbound only.** Blocked outbound UDP = endless reconnect loop |
| `9944` | Node RPC | B | Localhost |
| `9615` | Node Prometheus metrics | B | Localhost |
| `9900` | Miner metrics / `hive-stats` | A + B | Localhost |

All official links live in [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md).

---

## Where to Start

| Your situation | Start with |
|---|---|
| Windows PC, NVIDIA card, no Linux yet | **[ubuntu.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.md)** — it ends at Step 2 below |
| Renting a GPU by the hour | **[vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md)** — a self-contained Part A variant |
| Ubuntu or macOS with a working GPU driver | Continue to Step 1 |
| Just want the overview and links | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

# Shared Setup

Both routes start here. Two steps, then you choose.

## Step 1 — Create a Wallet

You need a `qz…` address before anything else. Both routes use the same wallet and the same 24 words.

1. Install the [Quantus Wallet](https://www.quantus.com/wallet/) (iOS / Android — links also on [linktr.ee/quantusnetwork](https://linktr.ee/quantusnetwork)).
2. Create a wallet and write the **24-word phrase on paper**, offline.
3. The address on the main screen starts with `qz…`. That is the only value you ever paste into a pool form or a lookup box.

CLI alternative:

```bash
# from https://github.com/Quantus-Network/quantus-cli/releases
quantus wallet create --name mining
```

> **CRITICAL:** the 24 words are the only recovery path for your rewards. Never type them into a chat window, a pool form or a rented server. Part A needs your **address**; Part B needs the **inner hash derived** from those words.

**Done when:** you can read a `qz…` address off the screen and the 24 words are on paper.

---

## Step 2 — Verify the GPU (Linux)

Confirm the proprietary NVIDIA driver is active before installing any miner. A miner on the wrong driver path silently runs 4–6× slow.

> No Linux yet? Do **[ubuntu.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.md)** first — it hands back here. On a rented GPU the driver is already injected by the host: use **[vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md)**.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq

# only if the driver is missing
sudo ubuntu-drivers autoinstall
sudo reboot
```

After the reboot:

```bash
nvidia-smi
nvidia-smi -L      # one line per GPU
```

You should see the card name, the driver version and a `CUDA Version` column. Low utilisation is expected — no miner is running yet.

> You do **not** need the full `cuda-toolkit`; the prebuilt miners carry their own CUDA runtime. If `nvidia-smi` prints nothing, disable Secure Boot (or enroll the MOK key) and reboot. RTX 50-series (Blackwell) needs a recent kernel and a 570+ driver.

**Done when:** `nvidia-smi -L` lists your card(s) by name.

---

## Decision Point — Pick Your Route

Shared setup is finished. Everything below is **one route or the other**.

| Choose | Go to | You will end up with |
|---|---|---|
| **Part A — Pool** | [A1](#a1--install-the-pool-miner) | One `quanpool-miner` service, hashing to a pool, balance accruing per share |
| **Part B — Own node** | [B1](#b1--automated-setup-script) | A synced `quantus-node` plus a local `quantus-miner`, full block rewards |

Still unsure? Do **Part A**. It is reversible in an hour, it proves your card and address work, and it starts paying while you think.

---

# Part A — Pool Mining (Quanpool PPLNS)

> **Prerequisites:** Step 1 (a `qz…` address) and Step 2 (a working GPU driver).
> **Ends at:** [Part A Finish Line](#part-a-finish-line).
> **You will not touch:** `quantus-node`, chain sync, inbound firewall rules, or an inner hash.

> Quanpool is a **community** pool, not official Quantus infrastructure. Host address, download link and TLS pin are published live on [quanpool.com](https://quanpool.com/) → **Start mining**. Copy them from there every time; the values below are placeholders on purpose.

### A1 — Install the Pool Miner

```bash
sudo mkdir -p /opt/quantus
sudo chown "$USER:$USER" /opt/quantus
cd /opt/quantus

wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version
```

If that version 404s, take the current Linux link from **Start mining** (6.1.0 also works).

> The `gpu-list` subcommand was removed in 6.1+. Enumerate cards with `nvidia-smi -L`.

### A2 — Benchmark the Card

A benchmark runs locally and never contacts the pool. Do it **before** any pool configuration — it answers the most expensive question in this guide: whether you are on the CUDA code path at all.

```bash
cd /opt/quantus
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
```

| Result | Meaning |
|---|---|
| 400 MH/s – 1.5 GH/s depending on the card | Correct — Linux CUDA path |
| ~100 MH/s on a modern RTX card | **Wrong binary or driver** — stock/wgpu path, 4–6× slower. Fix this now, not later |

Treat `--cpu-workers 0` as mandatory on a desktop. CPU mining adds ~15 MH/s per thread — negligible next to the card, while stealing the threads that feed it.

### A3 — Token and TLS Pin

On [quanpool.com](https://quanpool.com/) → **Start mining**, fill in:

| Field | Value |
|---|---|
| Address | your `qz…` address — **never the seed** |
| Worker | optional name, **unique per machine**. `a-z 0-9 . - _`, max 32 chars, no spaces |
| Mode | **Pool (PPLNS)** |
| System | **Linux** |

There is no sign-up and no password: the address *is* the account. Copy the `--node-addr` (prefer the literal `IP:9834` over a hostname) and the 64-hex `--tls-cert-sha256` value into files, so they never reach your shell history:

```bash
cd /opt/quantus

# one single line: qzYOURADDRESS.workername
nano auth-token

# the 64-hex TLS pin from Start mining
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

### A4 — First Manual Run

Run it in the foreground once so you can read the errors, before handing it to systemd.

```bash
cd /opt/quantus
./quanpool-miner serve \
  --node-addr <POOL_HOST>:9834 \
  --auth-token-file /opt/quantus/auth-token \
  --tls-cert-sha256-file /opt/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
```

Replace `<POOL_HOST>` with the address from **Start mining**, and set `--gpu-devices` to the card count from `nvidia-smi -L` (omit the flag to use all of them).

Healthy signs within the first minute:

- `nvidia-smi` shows the miner process and 95–100% GPU utilisation
- the log prints job lines and `SHARE FOUND`
- `curl -s http://127.0.0.1:9900/hive-stats` returns a populated `hs` and an `ar` with zero rejects

> A burst of `SOLUTION LOST` / stale messages in the first seconds is normal while the miner catches up to the current job.

Stop with `Ctrl+C` once it looks healthy — the next step makes it survive reboots.

### A5 — systemd Service

```bash
sudo tee /etc/systemd/system/quanpool-miner.service >/dev/null <<EOF
[Unit]
Description=Quantus pool miner (Quanpool PPLNS)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/opt/quantus
ExecStart=/opt/quantus/quanpool-miner serve --node-addr <POOL_HOST>:9834 --auth-token-file /opt/quantus/auth-token --tls-cert-sha256-file /opt/quantus/tls-cert-sha256 --cpu-workers 0 --gpu-devices 1 --mode pool
Restart=always
RestartSec=8
Nice=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now quanpool-miner
sudo systemctl status quanpool-miner --no-pager
journalctl -u quanpool-miner -f
```

Edit `<POOL_HOST>` in `ExecStart` before starting. `auth-token` is mode `600`, so it must be readable by the `User=` you set here.

A desktop must also stop suspending, or the GPU halts when the screen blanks:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

> A rented container usually has no working `systemd`. Use the image's process supervisor instead — see [vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md).

### A6 — Verify on the Pool

Locally first:

```bash
curl -s http://127.0.0.1:9900/hive-stats
```

`hs` is usually **kH/s** — `430000` ≈ 430 MH/s. `ar` is `[accepted, rejected]` and rejected should stay at zero; `temp`, `fan` and `pool_rtt_ms` are also reported.

Then open [quanpool.com](https://quanpool.com/), paste your `qz…` address into the lookup box and press **Look up**. Your worker appears within a few minutes, with its own hash rate, share count and a growing pending balance.

> Ports `9900`, `9833`, `9944` and `9615` must never be published to the internet. A router "virtual server" rule for them achieves nothing and creates real exposure.

### Part A Finish Line

**Complete when all four are true:**

1. `systemctl status quanpool-miner` reports `active (running)` and survives a reboot.
2. `nvidia-smi` shows 95–100% utilisation at the hash rate from your A2 benchmark.
3. `hive-stats` shows accepted shares climbing and rejected staying at zero.
4. Your worker is visible on the pool lookup page and the pending balance is rising.

Nothing else to install: no node, no sync, no inner hash. Continue to [Monitoring](#monitoring-and-everyday-commands), [Running Multiple Machines](#running-multiple-machines) or [Economics](#economics--what-to-expect). **Skip Part B** unless you later decide you want your own node — then read [Switching Routes Later](#switching-routes-later) first.

---

# Part B — Your Own Node

> **Prerequisites:** Step 1 (the 24 words — you derive the inner hash from them), Step 2 (a working GPU driver), a 100 GB+ SSD and bandwidth you can leave running.
> **Ends at:** [Part B Finish Line](#part-b-finish-line).
> **Two ways through:** B1 is the official script and does everything for you. B2–B5 are the manual equivalent. **Do B1, or do B2–B5 — not both.** B6 applies either way.

No pool fees and full custody, at the cost of a full chain sync and a matched binary pair. Works on Linux, macOS and WSL2.

### B1 — Automated Setup Script

The official script generates the wormhole inner hash and node identity, downloads a matched pair into `~/quantus-mining/bin/`, and writes `~/quantus-mining/mining.conf` with `CHAIN=mainnet`.

```bash
curl -fsSL https://docs.quantus.com/scripts/quantus-mining.sh -o quantus-mining.sh
chmod +x quantus-mining.sh
./quantus-mining.sh setup
./quantus-mining.sh start -d
```

Day-to-day control:

```bash
./quantus-mining.sh start          # node foreground, miner background
./quantus-mining.sh start-node     # one terminal
./quantus-mining.sh start-miner    # second terminal, after the miner server listens
./quantus-mining.sh stop
./quantus-mining.sh config show
./quantus-mining.sh config set GPU_DEVICES 1
./quantus-mining.sh config set CPU_WORKERS 0
```

Pin an explicit pair instead of letting two `latest` tags drift apart:

```bash
./quantus-mining.sh config set NODE_VERSION v1.0.1
./quantus-mining.sh config set MINER_VERSION v4.2.0
./quantus-mining.sh stop
./quantus-mining.sh setup --force
./quantus-mining.sh start -d
```

`--force` refreshes binaries only: it keeps your `INNER_HASH` and wormhole address and does **not** generate a new keypair. The script reads the node's `miner-auth-token` and `miner-tls-cert-sha256` itself, so you never copy those by hand. Docker mode has been removed.

> **If the script covered you, stop here** and jump to the [Part B Finish Line](#part-b-finish-line). B2–B5 build the same result by hand — useful when you want control over paths, versions and service management, or when the script fails on your distro.

### B2 — Manual Install: Binaries

```bash
sudo apt update
sudo apt install -y curl wget tar unzip jq screen ufw ca-certificates
mkdir -p ~/quantus && cd ~/quantus

wget https://github.com/Quantus-Network/chain/releases/download/v1.0.1/quantus-node-v1.0.1-x86_64-unknown-linux-gnu.tar.gz
tar -xzf quantus-node-v1.0.1-x86_64-unknown-linux-gnu.tar.gz
chmod +x quantus-node
./quantus-node --version

wget https://github.com/Quantus-Network/quantus-miner/releases/download/v4.2.0/quantus-miner-linux-x86_64 -O quantus-miner
chmod +x quantus-miner
```

Check [Releases](https://github.com/Quantus-Network/chain/releases) for current tags — official guidance is **node v1.0.1 or newer**, and the two repositories version independently.

Confirm the pair speaks the authenticated protocol **before** going further — both commands must print a match:

```bash
./quantus-node --help | grep miner-auth-token-file
./quantus-miner serve --help | grep auth-token-file
```

| Platform | Node archive | Miner binary |
|---|---|---|
| Linux x86_64 | `quantus-node-<tag>-x86_64-unknown-linux-gnu.tar.gz` | `quantus-miner-linux-x86_64` |
| Linux ARM64 | node exists | **no official miner** |
| macOS Apple Silicon | `…-aarch64-apple-darwin.tar.gz` | `quantus-miner-macos-aarch64` |
| Windows native | `…-x86_64-pc-windows-msvc.zip` | `quantus-miner-windows-x86_64.exe` |

On macOS, clear the Gatekeeper flag first: `xattr -d com.apple.quarantine quantus-node`.

### B3 — Node Identity and Wormhole Inner Hash

```bash
./quantus-node key generate-node-key --file node_key.p2p
./quantus-node key quantus --scheme wormhole --words
```

`--words` prompts for your 24 words **without echoing them**, so the phrase never lands in your shell history. Save the values it prints:

| Value | What it is | What to do with it |
|---|---|---|
| **Address** | your wormhole address — where rewards land | keep for monitoring |
| **Inner Hash** | the 32-byte preimage | pass as `--rewards-inner-hash` |
| **Secret** | the key proving ownership | back up offline, never share |

Using the same 24 words as your wallet app is the recommended path — rewards then show up in the app automatically. To mine to a brand-new wallet instead, run `./quantus-node key quantus --scheme wormhole` and back up the phrase it generates. Reward routing to a wormhole address is **not optional**; it is built into the protocol, which is also why your mining identity is not linked on-chain to your payout address.

### B4 — Start the Node

```bash
screen -S quantus-node
cd ~/quantus
./quantus-node \
  --name <YOUR_NODE_NAME> \
  --validator \
  --miner-listen-port 9833 \
  --chain mainnet \
  --node-key-file node_key.p2p \
  --rewards-inner-hash <YOUR_INNER_HASH> \
  --max-blocks-per-request 64 \
  --sync full
```

`--name` is how your node appears on [telemetry](https://telemetry.quantus.cat/).

On first start with `--miner-listen-port`, the node writes the miner auth material into the chain directory:

| File | Purpose |
|---|---|
| `miner-auth-token` | shared secret the miner sends in `Ready`. Mode `0600`, never logged |
| `miner-tls-cert-sha256` | SHA-256 of the miner QUIC certificate; miners pin this |
| `miner-tls-cert.der` / `miner-tls-key.der` | node TLS material — never copy the private key to a miner |

| Platform | Chain directory |
|---|---|
| Linux | `~/.local/share/quantus-node/chains/mainnet/` |
| macOS | `~/Library/Application Support/quantus-node/chains/mainnet/` |

> ⚠️ **Wait for a full sync before expecting rewards.** Blocks mined before you reach the tip are orphans and earn nothing; the miner pauses by itself when the node has no peers. You are synced when the log switches from `Syncing` to `Idle` at the current height — typically 15 minutes to a few hours. `discarding proposal` during sync is normal. A stall with `Verification failed` and 0 peers means your node version is out of step — check [Releases](https://github.com/Quantus-Network/chain/releases).

Also wait for the log line confirming the miner server is listening. If miner-server startup fails the node exits — there is no fallback to local mining.

### B5 — Start the External Miner

In a second terminal:

```bash
screen -S quantus-miner
cd ~/quantus
CHAIN_DIR="$HOME/.local/share/quantus-node/chains/mainnet"

./quantus-miner serve \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --node-addr 127.0.0.1:9833 \
  --auth-token-file "$CHAIN_DIR/miner-auth-token" \
  --tls-cert-sha256-file "$CHAIN_DIR/miner-tls-cert-sha256"
```

Variants: add `--cuda-gpu` on an NVIDIA card where Vulkan is unavailable; use `--cpu-workers 4 --gpu-devices 0` on a machine without a usable GPU; add `--gpu-throttle-ms 50` (or `5` if you also use the desktop) to cap heat and power draw.

> Always use `--auth-token-file` / `--tls-cert-sha256-file` rather than the inline forms. A wrong token or pin is a **permanent** error: the miner stops instead of reconnect-looping. Re-read the files.
>
> On macOS quote `CHAIN_DIR` — the path contains a space.

### B6 — Updating a Node/Miner Pair

Applies to both the scripted and the manual install:

1. Stop the node first, then the miner.
2. Download a **matched** pair — both must ship `quantus-miner/2`.
3. Keep `.../chains/mainnet/`. Never import a `chains/planck/` directory.
4. Start the node, wait for the miner server to listen, then start the miner.

### Part B Finish Line

**Complete when all five are true:**

1. The node log reads `Idle` at the current chain height, with a stable peer count.
2. Your `--name` is visible on [telemetry.quantus.cat](https://telemetry.quantus.cat/).
3. The node logged that the miner server is listening on `9833`.
4. The miner is connected, `nvidia-smi` shows 95–100% utilisation, and the node log shows `Broadcasting job`.
5. `ufw status` allows `22/tcp` and `30333/tcp` and **nothing else**.

Rewards now arrive as whole blocks, irregularly, at the wormhole address from B3 — check it in the wallet app or the explorer. Continue to [Monitoring](#monitoring-and-everyday-commands), then [Economics](#economics--what-to-expect) so the silence between blocks does not worry you.

---

# Reference

Applies to whichever route you finished.

## Monitoring and Everyday Commands

**Part A — pool**

```bash
sudo systemctl status quanpool-miner --no-pager
sudo systemctl restart quanpool-miner
journalctl -u quanpool-miner -f
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw,fan.speed --format=csv

# re-benchmark, or test outbound UDP (no reply from nc is not proof of failure)
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
nc -zvu <POOL_HOST> 9834
```

The pool's lookup page shows workers, pending balance and payouts. To update the binary: stop the service, download the new file, `chmod +x`, start again.

**Part B — own node**

| What | Where |
|---|---|
| Rewards | the wallet app (same 24 words), or `https://explorer.quantus.com/accounts/<YOUR_QZ_ADDRESS>` |
| Node on telemetry | [telemetry.quantus.cat](https://telemetry.quantus.cat/) — search your `--name` |
| Node metrics / RPC | `http://localhost:9615/metrics` · `http://localhost:9944` |
| Miner metrics | `http://localhost:9900/metrics` |

```bash
./quantus-node --version
./quantus-node key inspect-node-key --file node_key.p2p
tail -f ~/.local/share/quantus-node/chains/mainnet/network/quantus-node.log
```

A healthy node shows a stable peer count, `Idle` at the tip, `Prepared block for proposing` and `Broadcasting job`.

---

## Performance Reference

Published figures for the Linux CUDA pool miner versus the stock miner, GPU only. Always confirm your own card with `benchmark --duration 30`.

| Card | Stock miner | Linux CUDA pool miner | Ratio |
|---|---|---|---|
| RTX 5090 | 342 MH/s | 1.49 GH/s | ~4.4× |
| RTX 4090 | 179 MH/s | 1.12 GH/s | ~6.3× |
| RTX 4070 Ti | 122 MH/s | 574 MH/s | ~4.7× |
| RTX 4070 | — | ~430–460 MH/s | — |
| RTX 5070 | ~100 MH/s | ~410–450 MH/s | ~4.2× |
| RTX 3080 Ti | 104 MH/s | 435 MH/s | ~4.2× |
| RTX 5060 Ti | 75 MH/s | 314 MH/s | ~4.2× |
| CPU, 8 workers | ~120 MH/s total | — | ~15 MH/s per thread |

Rough expected income, ignoring fees and luck:

```text
daily QTC ≈ (your H/s / network H/s) × (86400 / block_seconds) × block_reward
```

Every term moves. Difficulty re-adjusts on each finalized block, so a single consumer card is a small fraction of a network measured in TH/s. Treat any fiat figure as unverified.

**Thermals:** a 24/7 mining card sits near its power limit with fans up. Sustained ~70 °C is normal, well below the ~83–88 °C throttle point; dust and case airflow are the real maintenance items. If the card holds 85 °C continuously, add `--gpu-throttle-ms 5`.

---

## Firewall

**Part A needs no inbound rules at all** — only outbound UDP to the pool port. If your router or ISP blocks outbound UDP/QUIC, the miner loops on reconnect; test from a phone hotspot to confirm.

**Part B:**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 30333/tcp
sudo ufw enable
sudo ufw status verbose
```

Do **not** open `9833`, `9944` or `9615`. For a remote miner, join it to the node over WireGuard or Tailscale and keep the miner port private.

Behind CGNAT (most mobile and many fibre home connections) inbound `30333` never arrives — the rule is harmless but does nothing, and the node still syncs through outbound connections. Port forwarding cannot fix CGNAT; only a public IP or a VPN endpoint can. This is precisely why Part A is the easier home route.

---

## Running Multiple Machines

Several miners can pay into the **same** `qz…` address; PPLNS shares add up.

- Give every machine a **different worker name**. Reusing one name makes two rigs collide and one disappears from the pool.
- You do not need to stop the home rig to add a second one.
- Never copy your seed to a rented machine. A rented box only needs `qzADDRESS.worker` and the TLS pin — see [vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md).
- On the Part B side the equivalent is several miners dialling one node on `9833` over a VPN. The node broadcasts jobs to all of them and the first valid result wins.

---

## Troubleshooting

### Part A — pool route

| Symptom | What to check |
|---|---|
| `timed out` / endless reconnect | Outbound **UDP** to the pool port. Home firewalls and some ISP paths block QUIC — test from a phone hotspot |
| `certificate` / `fingerprint` error | Re-copy the TLS pin from **Start mining**; exactly 64 hex characters |
| Worker never appears on the site | Address in `auth-token` differs from the one you looked up, or the worker name breaks the rules. Allow 2–3 minutes |
| Benchmark ~100 MH/s on a modern card | Not the CUDA build — wrong binary or driver. Check `nvidia-smi` |
| Still pointing at `127.0.0.1:9833` | That is the Part B address; pool mining uses the pool host on `9834` |
| Service `running` but GPU at 0% | Read the log: token, pin or `--node-addr` is wrong |
| Home worker vanished after adding a rig | Duplicate worker name. Rename one and restart both |
| Mining stops when the machine goes idle | Suspend is still enabled — re-apply the `gsettings` commands in A5 |

### Part B — own node route

| Symptom | What to check |
|---|---|
| Miner exits immediately | Auth or version: did you wait for the miner server to listen, do both auth files exist, do both binaries speak `quantus-miner/2` |
| TLS `no application protocol` | Node and miner are an incompatible pair |
| Wrong token or pin | Permanent error by design — re-read the files, do not retry blindly |
| `Verification failed`, 0 peers | Node version is out of step with the network |
| Sync never finishes | Bandwidth below 3 Mbps, or an HDD instead of an SSD |
| Windows sync stalls | Defender is scanning the RocksDB directory — add an exclusion, or use WSL2 |
| Mining but no rewards | Still syncing (orphans), wrong inner hash, or a different seed than your wallet |
| Node exits on the miner port | Bind, TLS or token failure. There is no local-mining fallback |
| Linux ARM64 has no miner | Mine from x86_64 or macOS |

On Windows, exclude the node data directory from Defender before syncing:

```powershell
Add-MpPreference -ExclusionPath "$env:USERPROFILE\.quantus"
```

> Pre-auth releases (node v0.9.0, miner v3.3.1 and earlier) have no miner authentication. Do not use them against current mainnet.

---

## Economics — What to Expect

**Emission.** Block rewards follow `(MaxSupply − CurrentSupply) / EmissionDivisor` — smooth exponential decay of a fixed 21,000,000 supply, with no halving cliffs. Miners receive **50% of total supply** over time and roughly 99% of all supply is emitted within about 40 years. A dev tax allocates 15% of block rewards to the company, vesting over years.

**On-chain fees.** Standard transfers pay a fixed fee that goes to the miner. High-security reversible transfers pay a volume-based fee that is burned, and ZK-aggregated transactions pay a smaller volume fee split between the miner and a burn. Mining income therefore comes overwhelmingly from emission, not from fees.

**Part A — how PPLNS pays.** You are paid **per share, not per block**. No block is ever "yours"; your balance accrues continuously and that is the whole point — it removes variance. The pool charges a flat fee and pays out above a single threshold; both are published on the pool site and both have changed over time, so read them there rather than trusting a number in any guide, including this one. Solo mode inside a pool carries the same fee but pays like a lottery.

**Part B — how solo pays.** Zero fees, full custody, and one whole block reward whenever you win. With a single consumer card against a network measured in TH/s, that can mean long silences. Nothing is wrong; variance is the price of not sharing.

**What moves your income.** Difficulty re-adjusts on every finalized block, so a rising network hash rate lowers your daily QTC even if your rig never changes. Measure with `benchmark`, then compare against the pool's live network hash rate.

> QTC market data is thin, and the unit is labelled `QUAN` in parts of the official docs and tooling. Treat any fiat projection as speculative.

---

## Switching Routes Later

The two routes share the wallet, the address and the GPU work, so moving between them is cheap.

**A → B (pool to own node)**

```bash
sudo systemctl disable --now quanpool-miner
```

Then start at [B1](#b1--automated-setup-script). Derive the inner hash from the **same** 24 words and rewards keep landing in the same wallet app. Leave any pending pool balance alone — it pays out on the normal schedule.

**B → A (own node to pool)**

Stop the miner, then the node. Keep `chains/mainnet/` if you might come back; it saves the resync. Then start at [A1](#a1--install-the-pool-miner) using the same `qz…` address.

**Running both at once is not a strategy.** Two miners on one GPU halve each other. With two cards, give each route its own card via `--gpu-devices` and separate services — otherwise pick one.

---

## Next Steps

| Goal | Document |
|---|---|
| Turn a Windows PC into a mining machine | [ubuntu.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.md) |
| Rent a GPU by the hour | [vast.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.md) |
| Network facts, links and route comparison | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

## About the Author

Prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
