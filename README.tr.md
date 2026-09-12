<div align="center">

# ⚛️ Quantus — GPU Madencilik & Tam Node Rehberleri

**Quantus mainnet'te QTC kaz — kendi PC'nde ya da kiralık GPU'da, havuzda ya da kendi node'unda**

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Consensus](https://img.shields.io/badge/Konsensus-QPoW%20%C2%B7%20Poseidon2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/qpow)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Yazar:** HazenNetworkSolutions
> **Ağ:** Quantus Mainnet (`--chain mainnet`)
> **Sürümler:** `quantus-node` v1.0.1+ · `quantus-miner` v4.2.x · havuz madencisi 6.2.0
> **Son Güncelleme:** Eylül 2026

---

## Buradan Başla

Dört doküman. Merakına göre değil, elindeki donanıma göre seç — yanlış şeyi kurmaktan kurtarır.

| Durumun buysa | Şunu oku |
|---|---|
| Ubuntu ya da macOS hazır, GPU sürücüsü çalışıyor | **[guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md)** — ana madencilik rehberi |
| NVIDIA kartlı Windows PC, henüz Linux yok | **[ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md)** → sonra guide.tr.md |
| Donanım yok — saatlik GPU kiralıyorsun | **[vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md)** |
| Havuz mu, kendi node mu, karar veremedin | [guide.tr.md → Hangi Yol](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#hangi-yol--a-mı-b-mi) |

> Aynı kartta Linux CUDA, Windows madencisinden **4–6 kat hızlı**. Ubuntu'ya sapmak kendini anında amorti ediyor.

---

## İki Yol

Madencilik için iki iş gerekir: zinciri takip edip blok adayı üreten bir **node**, ve nonce öğüten bir **madenci**. **Yol A**'da node'u havuz işletir, sen yalnızca madenciyi çalıştırırsın. **Yol B**'de ikisini de kendin çalıştırırsın — tam senkron, komisyon yok, kazandığın blokta ödülün tamamı. Cüzdan aynı, GPU'nun yaptığı iş aynı; karar geri alınabilir.

| Yol | Ne çalıştırıyorsun | Komisyon | Gelirin şekli | Kime uygun |
|---|---|---|---|---|
| Havuz (PPLNS) | sadece madenci | havuz komisyonu, havuz sitesinde yayınlı | düzenli akış | tek bir ev GPU'su |
| Kiralık GPU + havuz | aynı madenci, saatlik faturalı | havuz komisyonu + kira | düzenli akış | kendi donanımı olmayan |
| Kendi node + madenci | `quantus-node` + `quantus-miner` | yok | seyrek ama toplu | kontrol isteyen, komisyon istemeyen |

---

## Quantus Nedir

Substrate üzerine kurulu, akıllı kontrat platformu değil **değer saklama aracı** olarak tasarlanmış post-kuantum bir Proof-of-Work Layer 1. Post-kuantum imzalar (ML-DSA / Dilithium) ilk bloktan itibaren devrede ve **QPoW**, SHA-256'nın yerine çift Poseidon2 hash'i koyuyor; böylece madencilik işi ZK ispatlarının içinde doğrulanabiliyor. Poseidon2'nin sebebi devre verimliliği, ekstra kuantum direnci değil.

Mainnet **9 Eylül 2026**'da açıldı. Stake şartı ve validator kümesi yok — GPU'su olan herkes kazabilir. Ödüller yalnızca seed ifadenden türeyen **wormhole adreslerine** gider, yani madencilik geliri varsayılan olarak gizlidir ve ayrı bir talep adımı gerektirmez.

---

## Ağ Bilgileri

| Alan | Değer |
|---|---|
| Zincir parametresi | `--chain mainnet` (`planck` kapanan testnet) |
| Konsensus | QPoW — `Poseidon2(Poseidon2(block_hash \|\| nonce))` |
| Token / arz | QTC · 21.000.000 maksimum · 12 ondalık |
| Adres formatı | `qz…` (SS58 öneki 189) |
| Blok süresi | ~12 saniye hedef |
| Zorluk | Her finalize bloğunda yeniden ayarlanır, blok başına sınırlı — 2016 bloklu dönemler yok |
| Çatal seçimi | En uzun değil, kümülatif işe göre **en ağır** zincir |
| Finalizasyon | En iyi bloğun 179 blok gerisi (maks. reorg derinliği 180, ~18 dk) |
| Emisyon | `(MaxSupply − CurrentSupply) / EmissionDivisor` — yumuşak azalma, halving yok |
| Dağıtım | Madenciler zamanla arzın %50'si · blok ödüllerinde %15 geliştirme vergisi · ~40 yılda ~%99 emisyon |
| Ödüller | Yalnızca wormhole adresi, inner hash'inden türetilir |
| Madenci protokolü | ALPN `quantus-miner/2` — node ile madenci uyuşmak zorunda |
| Portlar | `30333/TCP` herkese açık · `9833/UDP`, `9944`, `9615`, `9900` sadece localhost |

Hash hızı **GPU'dan ve madenci derlemesinden** gelir — disk boyutundan ya da CPU önbelleğinden değil. QPoW VRAM'e aç değil, 8–12 GB fazlasıyla yeter. Modern kart başına kabaca 500 MH/s–1,5 GH/s, CPU'da iş parçacığı başına ~15 MH/s bekle. Linux ARM64'te node var ama **resmî madenci binary'si yok**. Tam gereksinimler: [guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#donanım-gereksinimleri).

> Bazı resmî doküman ve araçlar birimi hâlâ `QUAN` diye etiketliyor. Aynı 21M arzlı yerel token; iki tickerı da alakasız coinlerle karıştırma.

---

## Güvenliğin Temeli

- **24 kelimen kâğıttan çıkmaz.** Yol A için sadece `qz…` adresin, Yol B için sadece türetilen inner hash gerekir. İfadeyi kiralık makineye asla yazma.
- Sırları dosyayla geçir (`--auth-token-file`), satır içinde asla — shell geçmişi kalıcıdır.
- Sadece `30333/TCP` yayınla, başka hiçbir şey. Auth ve TLS pinleme, `9833/UDP`'yi dışa açmayı **güvenli yapmaz**.
- Uzaktaki bir madenci için madenci portunu açmak yerine WireGuard veya Tailscale kullan.
- Node ile madenci **eşleşmiş bir çift** olmalı. İki ayrı `latest` etiketi el sıkışmayı *no application protocol* hatasıyla düşürür.
- Quanpool bir **topluluk** projesi, resmî Quantus altyapısı değil. Host'unu ve TLS pinini her seferinde siteden kopyala; bekleyen bakiyeyi karşı taraf riski olarak gör.
- Havuz madencisiyle kendi node'unu **aynı GPU'ya** asla aynı anda bağlama.

---

## Bağlantılar

**Resmî** — [quantus.com](https://quantus.com/) · [dokümanlar](https://docs.quantus.com/) · [QPoW derinlemesine](https://docs.quantus.com/deep-dives/qpow) · [madenci protokolü](https://docs.quantus.com/deep-dives/miner-protocol/) · [kurulum betiği](https://docs.quantus.com/scripts/quantus-mining.sh)

**Kod** — [chain](https://github.com/Quantus-Network/chain) ([sürümler](https://github.com/Quantus-Network/chain/releases)) · [quantus-miner](https://github.com/Quantus-Network/quantus-miner) ([sürümler](https://github.com/Quantus-Network/quantus-miner/releases)) · [quantus-cli](https://github.com/Quantus-Network/quantus-cli/releases)

**Araçlar** — [cüzdan](https://www.quantus.com/wallet/) · [explorer](https://explorer.quantus.com) · [telemetri](https://telemetry.quantus.cat/) · [linktr.ee](https://linktr.ee/quantusnetwork)

**Topluluk** — [quanpool.com](https://quanpool.com/) (resmî olmayan havuz) · [Discord](https://discord.gg/vPkuc8eu42) · [Telegram](https://t.me/quantusnetwork) · [araştırma](https://research.quantus.com)

---

## Yazar Hakkında

**HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
