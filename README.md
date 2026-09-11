<div align="center">

# ⚛️ Quantus — GPU Mining & Full Node Setup Guides

**Mine QTC on Quantus mainnet — on your own PC or a rented GPU, in a pool or on your own node**

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Consensus](https://img.shields.io/badge/Consensus-QPoW%20%C2%B7%20Poseidon2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/qpow)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Quantus Mainnet (`--chain mainnet`)
> **Versions:** `quantus-node` v1.0.1+ · `quantus-miner` v4.1.x · pool miner 6.2.0
> **Last Updated:** September 2026

---

## 📚 Start Here

Three documents, one decision. Read the decision first — it saves you installing the wrong thing.

| If this is you | Read |
|---|---|
| Ubuntu or macOS ready, driver working | **[guide.md](guide.md)** — the main mining guide |
| Windows PC with an NVIDIA card, no Linux yet | **[ubuntu.md](ubuntu.md)** → then [guide.md](guide.md) |
| No hardware — renting a GPU by the hour | **[vast.md](vast.md)** |
| Not sure whether to pool or run a node | [guide.md → The Two Routes](guide.md#the-two-routes--a-or-b) |

> Linux CUDA is **4–6× faster** than the Windows miner on the same card. The Ubuntu detour pays for itself immediately.

---

## 🧭 The Two Routes, in One Paragraph

Mining needs two jobs done: a **node** that follows the chain and builds block candidates, and a **miner** that grinds nonces. In **Part A** a pool operator runs the node and you run only the miner — no sync, no open ports, income arrives as a steady trickle of shares. In **Part B** you run both yourself — a full sync, zero fees, full custody, and the whole block reward whenever you win. Same wallet, same GPU work, so the choice is reversible. The full comparison and a decision checklist live in [guide.md](guide.md#the-two-routes--a-or-b).

| Route | You run | Fees | Income shape | Best for |
|---|---|---|---|---|
| Pool (PPLNS) | miner only | pool fee, published on the pool site | steady trickle | a single home GPU |
| Rented GPU + pool | the same miner, billed hourly | pool fee + rent | steady trickle | no hardware of your own |
| Own node + miner | `quantus-node` + `quantus-miner` | none | rare lumps | custody, no fees, more work |

---

## 📋 What Quantus Is

A post-quantum Proof-of-Work Layer 1 on Substrate, built as a store of value rather than a smart-contract platform. Post-quantum signatures (ML-DSA / Dilithium) ship from block one, and **QPoW** replaces SHA-256 with double Poseidon2 hashing so that mining work can be verified inside ZK proofs — the reason for Poseidon2 is circuit efficiency, not extra quantum resistance.

Mainnet launched **9 September 2026**. There is no stake requirement and no validator set — anyone with a GPU can mine. Rewards go only to **wormhole addresses** derived from your seed phrase, so mining income is private by default and needs no claim step.

---

## ⚙️ Network Facts

| Field | Value |
|---|---|
| Chain flag | `--chain mainnet` (`planck` is the retired testnet) |
| Consensus | QPoW — `Poseidon2(Poseidon2(block_hash \|\| nonce))` |
| Token / supply | QTC · 21,000,000 max · 12 decimals |
| Address format | `qz…` (SS58 prefix 189) |
| Block time | ~12 seconds target |
| Difficulty | Re-adjusts on every finalized block, clamped per block — no 2016-block epochs |
| Fork choice | Heaviest chain by cumulative work, not longest |
| Finalization | 179 blocks behind best (max reorg depth 180, ~18 min) |
| Emission | `(MaxSupply − CurrentSupply) / EmissionDivisor` — smooth decay, no halvings |
| Distribution | Miners 50% of supply over time · 15% dev tax on block rewards · ~99% emitted in ~40 years |
| Rewards | Wormhole address only, derived from your inner hash |
| Miner protocol | ALPN `quantus-miner/2` — node and miner must match |
| Ports | `30333/TCP` public · `9833/UDP`, `9944`, `9615`, `9900` localhost only |

Hash rate comes from the **GPU and the miner build** — not disk size or CPU cache. QPoW is not VRAM-hungry, so 8–12 GB is plenty. GPU mining runs roughly 500 MH/s–1.5 GH/s per modern card against ~15 MH/s per CPU thread. Linux ARM64 has a node but **no official miner binary**. Full requirements: [guide.md](guide.md#hardware-requirements).

> Some official docs and tools still label the unit `QUAN`. It is the same 21M-supply native token; do not confuse either ticker with unrelated coins.

---

## 🔐 Security Essentials

- **Your 24 words never leave paper.** Part A needs only your `qz…` address; Part B needs only the derived inner hash. Never put the phrase on a rented machine.
- Pass secrets by file (`--auth-token-file`), never inline — shell history is forever.
- Publish `30333/TCP` and nothing else. Auth and TLS pinning do **not** make `9833/UDP` safe to expose.
- For a remote miner, use WireGuard or Tailscale instead of opening the miner port.
- Node and miner must be a **matched pair**. Two independent `latest` tags fail the handshake with *no application protocol*.
- Quanpool is a **community** project, not official Quantus infrastructure. Copy its host and TLS pin from the site each time; treat pending balances as counterparty risk.
- Never point a pool miner and your own node at the **same GPU**.

---

## 🔗 Links

**Official** — [quantus.com](https://quantus.com/) · [docs](https://docs.quantus.com/) · [QPoW deep dive](https://docs.quantus.com/deep-dives/qpow) · [miner protocol](https://docs.quantus.com/deep-dives/miner-protocol/) · [setup script](https://docs.quantus.com/scripts/quantus-mining.sh)

**Code** — [chain](https://github.com/Quantus-Network/chain) ([releases](https://github.com/Quantus-Network/chain/releases)) · [quantus-miner](https://github.com/Quantus-Network/quantus-miner) ([releases](https://github.com/Quantus-Network/quantus-miner/releases)) · [quantus-cli](https://github.com/Quantus-Network/quantus-cli/releases)

**Tools** — [wallet](https://www.quantus.com/wallet/) · [explorer](https://explorer.quantus.com) · [telemetry](https://telemetry.quantus.cat/) · [linktr.ee](https://linktr.ee/quantusnetwork)

**Community** — [quanpool.com](https://quanpool.com/) (unofficial pool) · [Discord](https://discord.gg/vPkuc8eu42) · [Telegram](https://t.me/quantusnetwork) · [research](https://research.quantus.com)

---

## About the Author

Prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
