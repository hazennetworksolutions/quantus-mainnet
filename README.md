<div align="center">

# ⚛️ Quantus — GPU Mining & Full Node Setup Guides

**Mine QTC on Quantus mainnet — on your own PC or a rented GPU, in a pool or on your own node**

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

## 📚 Start Here

| If this is you | Read |
|---|---|
| Ubuntu or macOS ready, driver working | **[guide.md](guide.md)** — the main mining guide |
| Windows PC with an NVIDIA card, no Linux yet | **[ubuntu.md](ubuntu.md)** → then [guide.md](guide.md) |
| No hardware — renting a GPU by the hour | **[vast.md](vast.md)** |
| No pool fees, full custody | [guide.md → Part B](guide.md#part-b--your-own-node-official-path) |

> Linux CUDA is **4–6× faster** than the Windows miner on the same card. The Ubuntu detour pays for itself immediately.

---

## 📋 What Quantus Is

A post-quantum Proof-of-Work Layer 1 on Substrate, built as a store of value rather than a smart-contract platform. Post-quantum signatures (ML-DSA / Dilithium) ship from block one, and **QPoW** replaces SHA-256 with double Poseidon2 hashing so mining work can later be verified inside ZK proofs.

Mainnet launched **9 September 2026**. There is no stake requirement and no validator set — anyone with a GPU can mine. Rewards go only to **wormhole addresses** derived from your seed phrase, so mining income is private by default and needs no claim step.

**Three ways to mine:**

| Route | You run | Fees | Best for |
|---|---|---|---|
| Pool (PPLNS) | A miner process only | ~6% | A single home GPU |
| Rented GPU + pool | The same miner, billed hourly | ~6% + rent | No hardware of your own |
| Own node + miner | `quantus-node` + `quantus-miner` | 0% | Custody, no fees, more work |

---

## ⚙️ Network Facts

| Field | Value |
|---|---|
| Chain flag | `--chain mainnet` (`planck` is the retired testnet) |
| Consensus | QPoW — `Poseidon2(Poseidon2(block_hash \|\| nonce))` |
| Token / supply | QTC · 21,000,000 max |
| Address format | `qz…` (SS58 prefix 189) |
| Block time | ~12 seconds |
| Difficulty | EMA per finalized block, clamped ±10% |
| Finalization | 179 blocks behind best (~18 min) |
| Emission | `(MaxSupply − CurrentSupply) / EmissionDivisor` — smooth decay, no halvings |
| Rewards | Wormhole address only, derived from your inner hash |
| Miner protocol | ALPN `quantus-miner/2` — node and miner must match |
| Ports | `30333/TCP` public · `9833/UDP`, `9944`, `9615`, `9900` localhost only |

Hash rate comes from the **GPU and the miner build** — not disk size or CPU cache. QPoW is not VRAM-hungry, so 8–12 GB is plenty. Linux ARM64 has a node but **no official miner binary**. Full requirements: [guide.md](guide.md#hardware-requirements).

---

## 🔐 Security Essentials

- **Your 24 words never leave paper.** A pool needs only your `qz…` address; a node needs only the derived inner hash. Never put the phrase on a rented machine.
- Pass secrets by file (`--auth-token-file`), never inline — shell history is forever.
- Publish `30333/TCP` and nothing else. Auth and TLS pinning do **not** make `9833/UDP` safe to expose.
- For a remote miner, use WireGuard or Tailscale instead of opening the miner port.
- Node and miner must be a **matched pair**. Two independent `latest` tags fail the handshake with *no application protocol*.
- Quanpool is a **community** project, not official Quantus infrastructure. Copy its host and TLS pin from the site each time; treat pending balances as counterparty risk.
- Never point a pool miner and your own node at the **same GPU**.

---

## 🔗 Links

**Official** — [quantus.com](https://quantus.com/) · [docs](https://docs.quantus.com/) · [mining guide](https://docs.quantus.com/guides/mining/) · [QPoW deep dive](https://docs.quantus.com/deep-dives/qpow/) · [miner protocol](https://docs.quantus.com/deep-dives/miner-protocol/) · [setup script](https://docs.quantus.com/scripts/quantus-mining.sh)

**Code** — [chain](https://github.com/Quantus-Network/chain) ([releases](https://github.com/Quantus-Network/chain/releases)) · [quantus-miner](https://github.com/Quantus-Network/quantus-miner) ([releases](https://github.com/Quantus-Network/quantus-miner/releases)) · [quantus-cli](https://github.com/Quantus-Network/quantus-cli/releases)

**Tools** — [wallet](https://www.quantus.com/wallet/) · [explorer](https://explorer.quantus.com) · [telemetry](https://telemetry.quantus.cat/) · [linktr.ee](https://linktr.ee/quantusnetwork)

**Community** — [quanpool.com](https://quanpool.com/) (unofficial pool) · [Discord](https://discord.gg/vPkuc8eu42) · [Telegram](https://t.me/quantusnetwork) · [research](https://research.quantus.com)

---

## About the Author

Prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
