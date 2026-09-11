<div align="center">

# ⚛️ Quantus Mainnet GPU Mining & Full Node Guide

**A complete guide to mining QTC on Quantus mainnet — pool (PPLNS) or your own node + external miner**
*Wallet and wormhole address, NVIDIA CUDA miner, systemd service, official node sync, monitoring and troubleshooting — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Miner](https://img.shields.io/badge/Miner%20Protocol-quantus--miner%2F2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/miner-protocol/)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/guides/mining/)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Quantus Mainnet (`--chain mainnet`)
> **Versions:** `quantus-node` v1.0.1+ · `quantus-miner` v4.1.x · `quanpool-miner` 6.2.0
> **Last Updated:** September 2026

---

## Where to Start

This is the main mining guide. Two companion documents cover getting a machine ready; come back here once you have one.

| Your situation | Start with |
|---|---|
| A Windows PC with an NVIDIA card and no Linux yet | **[ubuntu.md](ubuntu.md)** — install Ubuntu on a spare disk, then return to [Part A](#part-a--pool-mining-quanpool-pplns) |
| Renting a GPU by the hour instead of owning one | **[vast.md](vast.md)** — rented-container setup end to end |
| Ubuntu or macOS already running with a working GPU driver | You are in the right place — continue below |
| Just looking for an overview of the network | [readme.md](readme.md) |

---

## Table of Contents

- [How Mining Works](#how-mining-works)
- [Hardware Requirements](#hardware-requirements)
- [Ports and Endpoints](#ports-and-endpoints)
- [Step 1 — Create a Wallet](#step-1--create-a-wallet)
- [Step 2 — Verify the GPU (Linux)](#step-2--verify-the-gpu-linux)
- [Part A — Pool Mining (Quanpool PPLNS)](#part-a--pool-mining-quanpool-pplns)
  - [Step A1 — Install the Pool Miner](#step-a1--install-the-pool-miner)
  - [Step A2 — Benchmark the Card](#step-a2--benchmark-the-card)
  - [Step A3 — Get Your Token and TLS Pin](#step-a3--get-your-token-and-tls-pin)
  - [Step A4 — First Manual Run](#step-a4--first-manual-run)
  - [Step A5 — Create the systemd Service](#step-a5--create-the-systemd-service)
  - [Step A6 — Verify on the Pool](#step-a6--verify-on-the-pool)
- [Part B — Your Own Node (Official Path)](#part-b--your-own-node-official-path)
  - [Step B1 — Automated Setup Script](#step-b1--automated-setup-script)
  - [Step B2 — Manual Install: Binaries](#step-b2--manual-install-binaries)
  - [Step B3 — Node Identity and Wormhole Inner Hash](#step-b3--node-identity-and-wormhole-inner-hash)
  - [Step B4 — Start the Node](#step-b4--start-the-node)
  - [Step B5 — Start the External Miner](#step-b5--start-the-external-miner)
  - [Step B6 — Updating a Node/Miner Pair](#step-b6--updating-a-nodeminer-pair)
- [Monitoring and Everyday Commands](#monitoring-and-everyday-commands)
- [Performance Reference](#performance-reference)
- [Firewall](#firewall)
- [Running Multiple Machines](#running-multiple-machines)
- [Troubleshooting](#troubleshooting)
- [Economics — What to Expect](#economics--what-to-expect)

---

## How Mining Works

Quantus mining always has two logical parts:

1. **A node** (`quantus-node`) connects to mainnet, builds block candidates and hands out jobs. With `--miner-listen-port` it becomes a QUIC **server** on port `9833`.
2. **A miner** searches for a nonce on CPU, GPU or both. The miner is the **client**: it connects to the node, never the other way round.

The node's built-in CPU miner exists only for testing (~15 MH/s per thread). Real throughput comes from an external GPU miner, and multiple miners may connect to one node — the first valid result wins.

A **pool** removes part 1 from your responsibility: the pool operator runs the node, and your miner connects outbound over UDP to `host:9834`. That is why pool mining works behind CGNAT with no port forwarding, no static IP and no chain sync.

Rewards are never sent to a regular address. The protocol pays mining rewards to a **wormhole address** derived from a 32-byte preimage (your *inner hash*). Derive that preimage from the same 24 words as your wallet app and rewards appear in the app, spendable directly — there is no claim transaction.

> ⚠️ Use `--chain mainnet`. `planck` is the retired public testnet: separate chain, separate database, no balance migration. Never copy a `chains/planck/` directory into `chains/mainnet/`, and never pass `--force-authoring` when joining mainnet.

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Operating System | Ubuntu 20.04+, macOS, Windows 10/11 | Ubuntu 24.04 / 26.04 LTS |
| CPU | 2 cores | 4+ cores |
| RAM | 4 GB | 8 GB+ (16 GB comfortable) |
| Disk | 100 GB | 500 GB+ SSD — SATA is fine, HDD is not |
| Network | 3 Mbps | 10+ Mbps |
| GPU | none (CPU only, very slow) | NVIDIA RTX 20/30/40/50 series |

Notes that matter in practice:

- Hash rate is decided by the **GPU and the miner build**, not by disk size or CPU cache. A large-cache CPU does not accelerate Poseidon2.
- QPoW is **not VRAM-hungry**. 8–12 GB is plenty; a 140 GB datacenter card is not "50× faster" for this algorithm, and usually loses on cost per hash to a consumer 4090.
- The node database is RocksDB and does random I/O. Any SSD is fine; an HDD will stall the sync.
- **Linux ARM64 has no official miner binary.** Mine from Linux x86_64 or macOS.
- AMD GPUs do not work with the CUDA pool miner. The official miner supports Metal / Vulkan / DirectX depending on platform.
- Pool mining (Part A) needs none of the disk or bandwidth headroom above — only the GPU and outbound UDP.

---

## Ports and Endpoints

| Port | Purpose | What to do |
|---|---|---|
| `30333/TCP` | Node P2P | The only port that may face the internet. Optional — outbound peering syncs fine, and behind CGNAT inbound never arrives anyway |
| `9833/UDP` | Miner ↔ your own node (QUIC) | **Keep on localhost or a VPN.** The miner port binds `0.0.0.0`; only your firewall protects it |
| `9834/UDP` | Pool miner → pool node | **Outbound only.** If your network blocks outbound UDP the miner loops on reconnect |
| `9944` | Node RPC | Localhost |
| `9615` | Node Prometheus metrics | Localhost |
| `9900` | Miner metrics / `hive-stats` | Localhost |

| Resource | URL |
|---|---|
| Official mining guide | https://docs.quantus.com/guides/mining/ |
| Node releases | https://github.com/Quantus-Network/chain/releases |
| Miner releases | https://github.com/Quantus-Network/quantus-miner/releases |
| Setup script | https://docs.quantus.com/scripts/quantus-mining.sh |
| Wallet | https://www.quantus.com/wallet/ |
| Explorer | https://explorer.quantus.com |
| Telemetry | https://telemetry.quantus.cat/ |
| Community pool (unofficial) | https://quanpool.com/ |

---

## Step 1 — Create a Wallet

You need a `qz…` address before anything else. Both routes use the same wallet.

1. Install the [Quantus Wallet](https://www.quantus.com/wallet/) (iOS / Android — links also on [linktr.ee/quantusnetwork](https://linktr.ee/quantusnetwork)).
2. Create a wallet and write the **24-word phrase on paper**, offline.
3. The address on the main screen starts with `qz…`. That is the only value you ever paste into a pool or a lookup form.

CLI alternative, if you prefer a terminal:

```bash
# from https://github.com/Quantus-Network/quantus-cli/releases
quantus wallet create --name mining
```

> **CRITICAL:** the 24 words are the only recovery path for your mining rewards. Never type them into a chat window, a pool form, a `wget` command or a rented server. A pool needs your address; a node needs the derived inner hash. Neither needs your seed.

---

## Step 2 — Verify the GPU (Linux)

Confirm the proprietary NVIDIA driver is active before installing any miner.

> Coming from Windows with no Linux installed yet? Do **[ubuntu.md](ubuntu.md)** first — it ends exactly here. On a rented GPU, the driver is already injected by the host: see **[vast.md](vast.md)**.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq
```

If the driver is missing:

```bash
sudo ubuntu-drivers autoinstall
sudo reboot
```

After the reboot:

```bash
nvidia-smi
nvidia-smi -L      # one line per GPU
```

You should see the card name, the driver version and a `CUDA Version` column. Low GPU utilisation is expected — no miner is running yet.

> You do **not** need the full `cuda-toolkit` package. The prebuilt miners carry their own CUDA runtime; all that is required is a working proprietary driver. If `nvidia-smi` prints nothing, disable Secure Boot in the BIOS (or enroll the MOK key) and reboot. RTX 50-series (Blackwell) needs a recent kernel and a 570+ driver.

---

## Part A — Pool Mining (Quanpool PPLNS)

This is the low-friction route: no node, no sync, no inbound ports, works behind CGNAT. You run one process that connects outbound to the pool and gets paid per share.

> Quanpool is a **community** pool, not official Quantus infrastructure. The host address, the download link and the TLS certificate pin are all published live on [quanpool.com](https://quanpool.com/) → **Start mining**. Copy them from there each time; the values below are placeholders on purpose.

### Step A1 — Install the Pool Miner

```bash
sudo mkdir -p /opt/quantus
sudo chown "$USER:$USER" /opt/quantus
cd /opt/quantus

wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version
```

If that version 404s, take the current Linux download link from **Start mining**. Version 6.1.0 also works.

> The `gpu-list` subcommand was removed in 6.1+. Enumerate cards with `nvidia-smi -L` instead.

### Step A2 — Benchmark the Card

A benchmark runs locally and does not connect to the pool. Do this before anything else — it tells you immediately whether you are on the CUDA code path.

```bash
cd /opt/quantus
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
```

| Result | Meaning |
|---|---|
| 400 MH/s – 1.5 GH/s depending on the card | Correct — Linux CUDA path |
| ~100 MH/s on a modern RTX card | **Wrong binary or driver.** You are on the stock/wgpu path, roughly 4–6× slower |

`--cpu-workers 0` should be treated as mandatory on a desktop: the GPU does the work, the CPU stays responsive. CPU mining adds ~15 MH/s per thread — negligible next to the card, while stealing the threads that feed it.

### Step A3 — Get Your Token and TLS Pin

On [quanpool.com](https://quanpool.com/) → **Start mining**:

| Field | Value |
|---|---|
| Address | your `qz…` address — **never the seed** |
| Worker | optional name, **unique per machine**. Allowed: `a-z 0-9 . - _`, max 32 chars, no spaces |
| Mode | **Pool (PPLNS)** |
| System | **Linux** |

Copy the `--node-addr`, the `--tls-cert-sha256` value (64 hex characters) and the auth token format. Prefer the literal `IP:9834` shown on the site over a hostname.

Write both secrets to files rather than pasting them on a command line:

```bash
cd /opt/quantus

# one single line: qzYOURADDRESS.workername
nano auth-token

# the 64-hex TLS pin from Start mining
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

The token is simply your address, a dot, and your worker name:

```text
qzYOURADDRESS.rig1
```

### Step A4 — First Manual Run

Run it in the foreground once, so you can read the errors, before handing it to systemd.

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

Replace `<POOL_HOST>` with the address from **Start mining**, and set `--gpu-devices` to the number of cards from `nvidia-smi -L` (omit the flag to use all of them).

Healthy signs within the first minute:

- `nvidia-smi` shows the miner process and GPU utilisation near 95–100%
- the log prints job lines and `SHARE FOUND`
- `curl -s http://127.0.0.1:9900/hive-stats` returns a populated `hs` value and `ar` with a zero reject count

> A burst of `SOLUTION LOST` / stale messages in the first seconds is normal while the miner syncs to the current job. It should settle within a few minutes.

Stop with `Ctrl+C`.

### Step A5 — Create the systemd Service

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
```

Edit `<POOL_HOST>` in `ExecStart` to the live pool address before starting. The `auth-token` file is mode `600`, so it must be readable by the `User=` you set here.

Follow the log:

```bash
journalctl -u quanpool-miner -f
```

A desktop machine must also stop suspending, or the GPU stops when the screen blanks:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

> On a rented container there is usually no working `systemd`. Use the image's process supervisor instead — see [vast.md](vast.md).

### Step A6 — Verify on the Pool

```bash
curl -s http://127.0.0.1:9900/hive-stats
```

The `hs` field is usually in **kH/s** — `430000` means about 430 MH/s. `ar` is `[accepted, rejected]`; rejected should stay at zero. `temp`, `fan` and `pool_rtt_ms` are also reported.

Then open [quanpool.com](https://quanpool.com/), paste your `qz…` address into the lookup box and press **Look up**. Your worker appears within a few minutes with a live hash rate.

> Ports `9900`, `9833`, `9944` and `9615` must never be published to the internet. A router "virtual server" rule for them achieves nothing useful and creates real exposure.

---

## Part B — Your Own Node (Official Path)

No pool fees and full custody, at the cost of a full chain sync and a matched binary pair. Works on Linux, macOS and WSL2.

### Step B1 — Automated Setup Script

The official script generates the wormhole inner hash and node identity, downloads a matched pair into `~/quantus-mining/bin/`, and writes `~/quantus-mining/mining.conf` with `CHAIN=mainnet`.

```bash
curl -fsSL https://docs.quantus.com/scripts/quantus-mining.sh -o quantus-mining.sh
chmod +x quantus-mining.sh
./quantus-mining.sh setup
./quantus-mining.sh start -d
```

Day-to-day control:

```bash
./quantus-mining.sh start          # node in the foreground, miner in the background
./quantus-mining.sh start-node     # one terminal
./quantus-mining.sh start-miner    # second terminal, after the miner server is listening
./quantus-mining.sh stop
./quantus-mining.sh config show
./quantus-mining.sh config set GPU_DEVICES 1
./quantus-mining.sh config set CPU_WORKERS 0
```

Pin an explicit pair instead of letting two independent `latest` tags drift apart:

```bash
./quantus-mining.sh config set NODE_VERSION v1.0.1
./quantus-mining.sh config set MINER_VERSION v4.1.1
./quantus-mining.sh stop
./quantus-mining.sh setup --force
./quantus-mining.sh start -d
```

`--force` refreshes binaries only. It keeps your existing `INNER_HASH` and wormhole address and does **not** generate a new keypair. The script reads the node's `miner-auth-token` and `miner-tls-cert-sha256` itself once the node is up, so you never copy those by hand. Docker mode has been removed (`setup --mode docker` is rejected).

### Step B2 — Manual Install: Binaries

```bash
sudo apt update
sudo apt install -y curl wget tar unzip jq screen ufw ca-certificates
mkdir -p ~/quantus && cd ~/quantus
```

```bash
wget https://github.com/Quantus-Network/chain/releases/download/v1.0.1/quantus-node-v1.0.1-x86_64-unknown-linux-gnu.tar.gz
tar -xzf quantus-node-v1.0.1-x86_64-unknown-linux-gnu.tar.gz
chmod +x quantus-node
./quantus-node --version

wget https://github.com/Quantus-Network/quantus-miner/releases/download/v4.1.1/quantus-miner-linux-x86_64 -O quantus-miner
chmod +x quantus-miner
```

Confirm the pair speaks the authenticated protocol **before** going further:

```bash
./quantus-node --help | grep miner-auth-token-file
./quantus-miner serve --help | grep auth-token-file
```

Both must print a match. If only one does, the versions are mismatched.

| Platform | Node archive | Miner binary |
|---|---|---|
| Linux x86_64 | `quantus-node-<tag>-x86_64-unknown-linux-gnu.tar.gz` | `quantus-miner-linux-x86_64` |
| Linux ARM64 | node exists | **no official miner** |
| macOS Apple Silicon | `…-aarch64-apple-darwin.tar.gz` | `quantus-miner-macos-aarch64` |
| Windows native | `…-x86_64-pc-windows-msvc.zip` | `quantus-miner-windows-x86_64.exe` |

On macOS, clear the Gatekeeper flag first: `xattr -d com.apple.quarantine quantus-node`.

### Step B3 — Node Identity and Wormhole Inner Hash

```bash
./quantus-node key generate-node-key --file node_key.p2p
./quantus-node key quantus --scheme wormhole --words
```

`--words` prompts for your 24 words **without echoing them**, so the phrase never lands in your shell history. Save the two values it prints:

| Value | What it is | What to do with it |
|---|---|---|
| **Address** | your wormhole address — where rewards land | keep for monitoring |
| **Inner Hash** | the 32-byte preimage | pass as `--rewards-inner-hash` |

Using the same 24 words as your wallet app is the recommended path: rewards then show up in the app automatically. To mine to a brand-new wallet instead, run `./quantus-node key quantus --scheme wormhole` and back up the phrase it generates.

### Step B4 — Start the Node

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

`--name` is how your node appears on [telemetry](https://telemetry.quantus.cat/). Never add `--force-authoring` — that flag is only for bootstrapping a brand-new network.

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

> ⚠️ **Wait for a full sync before expecting rewards.** Blocks mined before you reach the chain tip are orphans and earn nothing; the miner pauses by itself when the node has no peers. The node is synced when the log switches from `Syncing` to `Idle` at the current height — typically 15 minutes to a few hours. `discarding proposal` during sync is normal. A stall with `Verification failed` and a drop to 0 peers means your node version is out of step with the network; check [Releases](https://github.com/Quantus-Network/chain/releases).

Also wait for the log line confirming the miner server is listening. If miner-server startup fails the node exits — there is no fallback to local mining.

### Step B5 — Start the External Miner

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

On an NVIDIA card where Vulkan is unavailable, add `--cuda-gpu`:

```bash
./quantus-miner serve \
  --cuda-gpu \
  --gpu-devices 1 \
  --cpu-workers 0 \
  --node-addr 127.0.0.1:9833 \
  --auth-token-file "$CHAIN_DIR/miner-auth-token" \
  --tls-cert-sha256-file "$CHAIN_DIR/miner-tls-cert-sha256"
```

A conservative CPU-only variant for a machine without a usable GPU is `--cpu-workers 4 --gpu-devices 0`.

> Always use `--auth-token-file` / `--tls-cert-sha256-file` rather than the inline `--auth-token` / `--tls-cert-sha256` forms, so the secret stays out of your shell history. A wrong token or pin is a **permanent** error: the miner does not reconnect-loop, it stops. Re-read the files.
>
> On macOS quote `CHAIN_DIR` — the path contains a space.

To cap heat and power draw, add `--gpu-throttle-ms 50` (or `5` if you also use the desktop while mining).

### Step B6 — Updating a Node/Miner Pair

1. Stop the node first, then the miner.
2. Download a **matched** pair — both must ship `quantus-miner/2`.
3. Keep `.../chains/mainnet/`. Never import a `chains/planck/` directory.
4. Start the node, wait for the miner server to listen, then start the miner.

---

## Monitoring and Everyday Commands

**Pool route**

```bash
sudo systemctl status quanpool-miner --no-pager
sudo systemctl restart quanpool-miner
journalctl -u quanpool-miner -f
journalctl -u quanpool-miner -n 80 --no-pager
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw,fan.speed --format=csv

# re-run a benchmark, or test whether outbound UDP to the pool is permitted
# (no reply from nc is not proof of failure)
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
nc -zvu <POOL_HOST> 9834
```

Plus the pool's own lookup page for workers, pending balance and paid amounts. To update the binary: stop the service, download the new file, `chmod +x`, start the service. If the current version works, an available update is not urgent.

**Own node route**

| What | Where |
|---|---|
| Rewards | the wallet app, using the same 24 words — or `https://explorer.quantus.com/accounts/<YOUR_QZ_ADDRESS>` |
| Node on telemetry | [telemetry.quantus.cat](https://telemetry.quantus.cat/) — search your `--name` |
| Node metrics | `http://localhost:9615/metrics` |
| Node RPC | `http://localhost:9944` |
| Miner metrics | `http://localhost:9900/metrics` |

```bash
./quantus-node --version
./quantus-node key inspect-node-key --file node_key.p2p
tail -f ~/.local/share/quantus-node/chains/mainnet/network/quantus-node.log
```

A healthy node shows a stable peer count, `Idle` at the tip, `Prepared block for proposing` and `Broadcasting job`.

---

## Performance Reference

Measured and published figures for the Linux CUDA pool miner versus the stock miner, on GPU only. Always confirm your own card with `benchmark --duration 30`.

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

Every term on the right moves. Difficulty re-adjusts on each finalized block, so a single consumer card is a small fraction of a network measured in TH/s. Treat any fiat figure as unverified.

For 24/7 operation a mining card sits near its power limit with fans up. Sustained ~70 °C is normal and well below the ~83–88 °C throttle point; dust and case airflow are the real maintenance items. If the card holds 85 °C continuously, add `--gpu-throttle-ms 5`.

---

## Firewall

Pool mining needs **no inbound rules at all** — only outbound UDP to the pool port.

For your own node:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 30333/tcp
sudo ufw enable
sudo ufw status verbose
```

Do **not** open `9833`, `9944` or `9615`. If you need a remote miner, join it to the node over WireGuard or Tailscale and keep the miner port private. Behind CGNAT, inbound `30333` never arrives — the rule is harmless but does nothing, and the node still syncs through outbound connections.

---

## Running Multiple Machines

Several miners can pay into the **same** `qz…` address; PPLNS shares simply add up.

- Give every machine a **different worker name**. Reusing one name across two rigs makes them collide and one disappears from the pool.
- You do not need to stop the home rig to add a second one.
- Never copy your seed phrase to a rented machine. A rented box only needs `qzADDRESS.worker` and the TLS pin — full procedure in [vast.md](vast.md).

---

## Troubleshooting

### Pool route

| Symptom | What to check |
|---|---|
| `timed out` / endless reconnect | Outbound **UDP** to the pool port. Home firewalls, egress filters and some ISP scrubbing paths block QUIC. Test from a phone hotspot to isolate it |
| `certificate` / `fingerprint` error | Re-copy the TLS pin from **Start mining**; it must be exactly 64 hex characters |
| Worker never appears on the site | The address in `auth-token` differs from the one you looked up, or the worker name breaks the character rules. Allow 2–3 minutes |
| Benchmark ~100 MH/s on a modern card | Not the CUDA build — wrong binary or a driver problem. Check `nvidia-smi` |
| Still pointing at `127.0.0.1:9833` | That is the own-node address; pool mining uses the pool host on `9834` |
| Service `running` but GPU at 0% | Read the log: token, pin or `--node-addr` is wrong |
| Home worker vanished after adding a rig | Duplicate worker name. Rename one and restart both |
| Mining stops when the machine goes idle | Suspend is still enabled — re-apply the `gsettings` commands in Step A5 |

### Own node route

| Symptom | What to check |
|---|---|
| Miner exits immediately | Auth or version. Did you wait for the miner server to listen, do both auth files exist, do both binaries speak `quantus-miner/2` |
| TLS `no application protocol` | Node and miner are an incompatible pair |
| Wrong token or pin | Permanent error by design — re-read the files, do not retry blindly |
| `Verification failed`, 0 peers | Node version is out of step with the network |
| Sync never finishes | Bandwidth below the 3 Mbps minimum, or an HDD instead of an SSD |
| Windows sync stalls | Defender is scanning the RocksDB directory — add an exclusion for the chain path, or use WSL2 |
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

- **PPLNS pays by share, not by block.** No block is ever "yours"; your balance accrues continuously. That is the point — it removes variance.
- **Fees:** the community pool charges 1%, and its CUDA miner adds 5% on top (6% total). The stock miner in pool mode carries only the 1%. Solo mode in a pool has the same fees but pays like a lottery.
- **Payout threshold** is published on the pool site and has changed over time. Check it in your lookup rather than trusting a number written in any guide.
- **Emission decays smoothly** — `(MaxSupply − CurrentSupply) / EmissionDivisor`, with no halving cliffs. Miners receive 50% of supply over time; ~99% is emitted within roughly 40 years.
- **Fee model:** standard transfers pay a miner tip; wormhole / high-security transactions pay a volume fee split between the miner and a burn.
- **Rising network hash rate lowers your daily QTC** even if your rig never changes.
- QTC market data is thin and easy to confuse with unrelated tickers. Any fiat projection is speculative.
- **Running your own node is a different bet:** 0% fees and full custody, but your income arrives in rare lumps and you carry the sync and uptime work yourself.

---

## Next Steps

| Goal | Document |
|---|---|
| Turn a Windows PC into a mining machine | [ubuntu.md](ubuntu.md) |
| Rent a GPU by the hour instead of buying one | [vast.md](vast.md) |
| Network facts, links and route comparison | [readme.md](readme.md) |

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
