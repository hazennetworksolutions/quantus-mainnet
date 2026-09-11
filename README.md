<div align="center">

# ⚛️ Quantus — GPU Mining & Full Node Setup Guides

**Everything needed to mine QTC on Quantus mainnet — on your own PC or on a rented GPU, in a pool or on your own node**
*Wallet and wormhole address, Ubuntu Desktop + NVIDIA CUDA, pool miner under systemd, rented GPU under supervisor, official node + external miner.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Consensus](https://img.shields.io/badge/Consensus-QPoW%20%C2%B7%20Poseidon2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/qpow/)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/guides/mining/)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Quantus Mainnet (`--chain mainnet`)
> **Versions:** `quantus-node` v1.0.1+ · `quantus-miner` v4.1.x · `quanpool-miner` 6.2.0
> **Last Updated:** September 2026

---

## 📚 Guides

| Guide | What it covers | Language |
|---|---|---|
| **[guide.md](guide.md)** | The main guide — mining concepts, pool (PPLNS) setup, your own node + external miner, monitoring, firewall, troubleshooting, economics | 🇬🇧 English |
| **[ubuntu.md](ubuntu.md)** | Installing Ubuntu Desktop on a spare disk of a Windows PC and getting the NVIDIA driver working | 🇬🇧 English |
| **[vast.md](vast.md)** | Renting an NVIDIA GPU by the hour and running the pool miner on it under supervisor | 🇬🇧 English |

### Which one do I need?

| Your situation | Path |
|---|---|
| Windows PC with an NVIDIA card, no Linux yet | [ubuntu.md](ubuntu.md) → then [guide.md](guide.md) |
| Ubuntu or macOS already running with a working driver | [guide.md](guide.md) directly |
| No suitable hardware — you want to rent a GPU | [vast.md](vast.md) (it points back to [guide.md](guide.md) for the concepts) |
| You want no pool fees and full custody | [guide.md → Part B](guide.md#part-b--your-own-node-official-path) |

> Linux CUDA is roughly **4–6× faster** than the Windows stock miner on the same card. That is why the Ubuntu detour is worth it before you start mining.

---

## 📋 Overview

Quantus is a post-quantum Proof-of-Work Layer 1 built on Substrate, designed as a store of value rather than a smart-contract platform. Post-quantum signatures (ML-DSA / Dilithium) are in the protocol from block one, and the mining algorithm — **QPoW** — replaces SHA-256 with **double Poseidon2** hashing so that mining work can later be verified inside ZK proofs. Mainnet went live on **9 September 2026**; the previous public testnet (`planck`) is retired and shares no history, database, or balances with mainnet.

There is no stake requirement and no validator set: anyone with a CPU or GPU can mine. Rewards are paid exclusively to **wormhole addresses** derived from your seed phrase, which makes mining income privacy-preserving by default and requires no separate claim step when you use the same 24 words in the Quantus wallet app.

Three practical routes exist for an operator, and this documentation set covers all of them:

1. **Pool mining (Quanpool, PPLNS)** — you run only a miner process. No node, no sync, no public IP, no inbound ports. Works behind CGNAT with outbound UDP only. Recommended for a single home GPU.
2. **Rented GPU, pool mining** — the same miner on someone else's hardware, billed per hour, persisted with the image's supervisor. No hardware purchase, and nothing secret ever lands on the box.
3. **Your own node + external miner (official path)** — you run `quantus-node` as a validator plus `quantus-miner`, pay no pool fee, and keep custody of everything. More operational work: a full sync and a matched node/miner pair are mandatory.

---

## 🔗 Official Links

- Website: [quantus.com](https://quantus.com/) · Documentation: [docs.quantus.com](https://docs.quantus.com/)
- Official mining guide: [docs.quantus.com/guides/mining](https://docs.quantus.com/guides/mining/)
- QPoW deep dive: [docs.quantus.com/deep-dives/qpow](https://docs.quantus.com/deep-dives/qpow/) · Miner protocol: [miner-protocol](https://docs.quantus.com/deep-dives/miner-protocol/)
- Node source: [Quantus-Network/chain](https://github.com/Quantus-Network/chain) · [Releases](https://github.com/Quantus-Network/chain/releases)
- Miner source: [Quantus-Network/quantus-miner](https://github.com/Quantus-Network/quantus-miner) · [Releases](https://github.com/Quantus-Network/quantus-miner/releases)
- CLI: [quantus-cli releases](https://github.com/Quantus-Network/quantus-cli/releases) · Setup script: [quantus-mining.sh](https://docs.quantus.com/scripts/quantus-mining.sh)
- Wallet: [quantus.com/wallet](https://www.quantus.com/wallet/) · [linktr.ee/quantusnetwork](https://linktr.ee/quantusnetwork)
- Explorer: [explorer.quantus.com](https://explorer.quantus.com) · Telemetry: [telemetry.quantus.cat](https://telemetry.quantus.cat/)
- Community pool (unofficial): [quanpool.com](https://quanpool.com/) · [Discord](https://discord.gg/vPkuc8eu42)
- Support: [Telegram](https://t.me/quantusnetwork) · [GitHub Issues](https://github.com/Quantus-Network/chain/issues) · [research.quantus.com](https://research.quantus.com)

---

## ⚙️ Network Facts

| Field | Value |
|---|---|
| Chain flag | `--chain mainnet` (`planck` is the retired testnet) |
| Consensus | QPoW — `Poseidon2(Poseidon2(block_hash \|\| nonce))` |
| Node binary | `quantus-node` (Substrate, v1.0.1+) |
| Miner binary | `quantus-miner` (official) or `quanpool-miner` (pool, Linux CUDA) |
| Miner wire protocol | ALPN `quantus-miner/2` — node and miner must match |
| Token | QTC (documented as QUAN in some deep dives, 12 decimals) |
| Max supply | 21,000,000 |
| Address format | `qz…` (SS58 prefix 189) |
| Target block time | ~12 seconds |
| Difficulty adjustment | EMA per finalized block, clamped ±10% |
| Finalization | 179 blocks behind best (~18 minutes) |
| Max reorg depth | 180 blocks |
| Emission | `(MaxSupply − CurrentSupply) / EmissionDivisor` — smooth decay, no halving cliffs |
| Reward destination | Wormhole address only, derived from your inner hash |
| P2P port | `30333/TCP` (the only port that may face the internet) |
| Miner port (own node) | `9833/UDP` — localhost or VPN only |
| RPC / metrics | `9944` / `9615` (node), `9900` (miner) — localhost only |

Full hardware requirements and per-card performance figures are in [guide.md](guide.md#hardware-requirements). In short: the **GPU and the miner build** decide your hash rate — disk size and CPU cache do not. QPoW is not VRAM-hungry, so 8–12 GB is plenty. Linux ARM64 has a node build but **no official miner binary**.

---

## 🔐 Security Essentials

- Your **24-word phrase never leaves paper or an offline backup.** A pool only ever needs your `qz…` address; a node only needs the derived inner hash. Never put it on a rented machine.
- Treat `miner-auth-token` like a password: pass it with `--auth-token-file`, never inline, so it stays out of shell history.
- Expose only `30333/TCP`. Auth and TLS pinning do **not** make `9833/UDP` safe to publish, and `9900` is for localhost only.
- For remote miners use WireGuard or Tailscale instead of opening the miner port.
- Node and miner must ship the same miner protocol (`quantus-miner/2`); mixing independent `latest` tags fails the TLS handshake with *no application protocol*.
- Quanpool is a community project, not official Quantus infrastructure. Copy its host address and TLS pin live from the site each time, and treat pending balances as counterparty risk until paid.
- Never run a pool miner and your own node against **the same GPU** — they compete for the device and both lose.

---

## About the Author

This documentation set was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
