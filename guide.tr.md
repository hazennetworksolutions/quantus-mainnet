<div align="center">

# ⚛️ Quantus Mainnet GPU Madencilik & Tam Node Rehberi

**Quantus mainnet'te QTC kaz — ya bir havuza katıl, ya kendi node'unu harici GPU madencisiyle çalıştır**
*İki eksiksiz yol, tek ortak kurulum. Cüzdan, CUDA madencisi, systemd servisi, node senkronu, takip ve sorun giderme.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Madenci](https://img.shields.io/badge/Madenci%20Protokolü-quantus--miner%2F2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/miner-protocol/)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Hazırlayan:** HazenNetworkSolutions
> **Ağ:** Quantus Mainnet (`--chain mainnet`)
> **Sürümler:** `quantus-node` v1.0.1+ · `quantus-miner` v4.2.x · havuz madencisi 6.2.0
> **Son Güncelleme:** Eylül 2026

---

## İçindekiler

**Önce bunları oku**

- [Madencilik Nasıl Çalışır](#madencilik-nasıl-çalışır)
- [Hangi Yol — A mı, B mi](#hangi-yol--a-mı-b-mi)
- [Donanım Gereksinimleri](#donanım-gereksinimleri)
- [Portlar ve Uç Noktalar](#portlar-ve-uç-noktalar)

**Ortak kurulum — herkes bunu yapar**

- [Adım 1 — Cüzdan Oluştur](#adım-1--cüzdan-oluştur)
- [Adım 2 — GPU'yu Doğrula (Linux)](#adım-2--gpuyu-doğrula-linux)
- [Karar Noktası — Yolunu Seç](#karar-noktası--yolunu-seç)

**Sonra tek bir yol, ikisi birden değil**

- [Yol A — Havuz Madenciliği (Quanpool PPLNS)](#yol-a--havuz-madenciliği-quanpool-pplns)
  - [Adım A1 — Havuz Madencisini Kur](#adım-a1--havuz-madencisini-kur)
  - [Adım A2 — Kartı Benchmark Et](#adım-a2--kartı-benchmark-et)
  - [Adım A3 — Token ve TLS Pinini Al](#adım-a3--token-ve-tls-pinini-al)
  - [Adım A4 — Manuel Çalıştırma](#adım-a4--manuel-çalıştırma)
  - [Adım A5 — systemd Servisi Oluştur](#adım-a5--systemd-servisi-oluştur)
  - [Adım A6 — Havuzda Doğrula](#adım-a6--havuzda-doğrula)
  - [Yol A Bitiş Çizgisi](#yol-a-bitiş-çizgisi)
- [Yol B — Kendi Node'un (Resmî Yol)](#yol-b--kendi-nodeun-resmî-yol)
  - [Adım B1 — Otomatik Kurulum Betiği](#adım-b1--otomatik-kurulum-betiği)
  - [Adım B2 — Manuel Kurulum: Binary'ler](#adım-b2--manuel-kurulum-binaryler)
  - [Adım B3 — Node Kimliği ve Wormhole Inner Hash](#adım-b3--node-kimliği-ve-wormhole-inner-hash)
  - [Adım B4 — Node'u Başlat](#adım-b4--nodeu-başlat)
  - [Adım B5 — Harici Madenciyi Başlat](#adım-b5--harici-madenciyi-başlat)
  - [Adım B6 — Node/Madenci Çiftini Güncelleme](#adım-b6--nodemadenci-çiftini-güncelleme)
  - [Yol B Bitiş Çizgisi](#yol-b-bitiş-çizgisi)

**Referans — iki yol için de geçerli**

- [Günlük Komutlar ve Takip](#günlük-komutlar-ve-takip)
- [Performans Referansı](#performans-referansı)
- [Güvenlik Duvarı](#güvenlik-duvarı)
- [Birden Fazla Makine](#birden-fazla-makine)
- [Sorun Giderme](#sorun-giderme)
- [Ekonomi — Ne Beklemeli](#ekonomi--ne-beklemeli)
- [Sonradan Yol Değiştirme](#sonradan-yol-değiştirme)

---

## Madencilik Nasıl Çalışır

Quantus madenciliğinde her zaman **iki iş** vardır ve bu ayrımı anladığın an rehberin kalanı kendiliğinden yerine oturur:

1. **Node** (`quantus-node`) mainnet'e bağlanır, zincirin tam kopyasını tutar, blok adaylarını hazırlar ve madencilik işlerini dağıtır. `--miner-listen-port` ile başlatıldığında ayrıca `9833` portunda bir QUIC sunucusu olur.
2. **Madenci** tek bir şey yapar: kazanan nonce'u arar, GPU ya da CPU üzerinde. Madenci her zaman *istemcidir*; node'a o bağlanır, tersi olmaz.

Algoritma **QPoW** — SHA-256 yerine çift Poseidon2 hash'i. Bir nonce, `Poseidon2(Poseidon2(blok_hash ‖ nonce))` sonucu zorluk hedefinin altına düştüğünde kazanır. Poseidon2'nin seçilme nedeni ZK devresi içinde doğrulanmasının ucuz olması; SHA-256'nın zayıf olması değil.

Node'un dahili CPU madencisi test amaçlıdır — çekirdek başına kabaca **15 MH/s**. Gerçek hız harici GPU madencisinden gelir: modern bir kartta **500 MH/s – 1,5 GH/s** aralığı. Aynı node'a birden fazla madenci bağlanabilir; geçerli sonucu ilk bulan kazanır.

**Asıl soru şu: 1. işi kim çalıştırıyor?** Node'u kendin çalıştırırsan tüm hattın sahibi olursun. Havuza katılırsan node'u operatör çalıştırır, senin madencin sadece UDP üzerinden `host:9834` adresine dışa doğru bağlanır — havuz madenciliğinin CGNAT arkasında, port yönlendirmesi olmadan, sabit IP olmadan ve zincir senkronu olmadan çalışmasının sebebi tam olarak budur.

Her iki durumda da ödüller normal bir adrese düşmez. Protokol, *inner hash* denilen 32 baytlık bir ön görüntüden türetilen bir **wormhole adresine** ödeme yapar. Bunu cüzdan uygulamandaki 24 kelimenin aynısından türetirsen ödüller doğrudan uygulamada, harcanabilir şekilde görünür; talep işlemi yoktur.

### Bilmeye değer zincir bilgileri

| Parametre | Değer |
|---|---|
| Algoritma | QPoW — çift Poseidon2 |
| Adres biçimi | SS58 öneki 189 — adresler `qz…` ile başlar |
| Maksimum arz | 21.000.000, 12 ondalık |
| Zorluk ayarı | Her sonlanan blokta, blok başına sınırlı — 2016 bloklu dönem yok |
| Çatal seçimi | En uzun değil, kümülatif işi en ağır zincir |
| Sonlanma | Uçtan 179 blok geride (maks. reorg derinliği 180) |
| Blok ödülü | `(MaxSupply − CurrentSupply) / EmissionDivisor` — yumuşak azalma, halving yok |

> ⚠️ Her zaman `--chain mainnet` kullan. `planck` kapanmış test ağıdır: ayrı zincir, ayrı veritabanı, bakiye taşıması yok. `chains/planck/` klasörünü asla `chains/mainnet/` içine kopyalama ve mainnet'te asla `--force-authoring` verme — o parametre sıfırdan yeni bir ağ başlatmak içindir.

---

## Hangi Yol — A mı, B mi

Bu rehberin birbirini dışlayan iki yarısı var. **Hiçbir şey kurmadan önce burayı oku**, çünkü bu seçim neyi indireceğini, güvenlik duvarında neyi açacağını ve nasıl ödeme alacağını baştan belirliyor.

### Yol A — Havuz Madenciliği

Sadece **madenciyi** çalıştırırsın. Madenci, bir havuz operatörünün node'una dışa doğru bağlanır, o node'un gönderdiği işleri çalışır ve pay (share) gönderir. Blokları havuz bulur, ödülü katkı paylarına göre böler (PPLNS).

- **Kuracağın:** tek bir binary, tek bir systemd servisi.
- **Gereken:** bir `qz…` adresi, bir GPU, dışa açık UDP.
- **Gerekmeyen:** senkron zincir, disk alanı, içe açık port, genel IP.
- **Ödeme şekli:** küçük miktarlar, sürekli akan; ödeme eşiğini geçtikten sonra.
- **İlk hash'e kadar:** kabaca 15–30 dakika.

### Yol B — Kendi Node'un

Her iki işi de **kendi donanımında** çalıştırırsın: mainnet'i senkronlayan `quantus-node` ve ona `127.0.0.1:9833` üzerinden bağlanan `quantus-miner`.

- **Kuracağın:** eşleşen iki binary, node kimliği, wormhole inner hash.
- **Gereken:** 100 GB+ SSD, istikrarlı bant genişliği, ilk senkron için sabır.
- **Ödeme şekli:** node'un blok bulduğunda **ödülün tamamı** — arada ise hiç.
- **Ayrıca:** sıfır havuz komisyonu, tam kendi kontrolün, operatöre güven gerekmez ve ağı güçlendiren bir node.
- **İlk hash'e kadar:** birkaç saat, çoğu senkron süresi.

### Yan yana

| | **Yol A — Havuz** | **Yol B — Kendi node'un** |
|---|---|---|
| Çalışan süreç | 1 (madenci) | 2 (node + madenci) |
| Zincir senkronu | yok | tam senkron şart |
| Disk | önemsiz | 100 GB+ SSD, HDD olmaz |
| CGNAT arkasında | evet, tasarımı gereği | evet, dışa bağlantı yeterli |
| İçe açık port | yok | `30333/TCP` opsiyonel |
| Gelir biçimi | düzenli sızıntı | seyrek büyük parçalar |
| Komisyon | havuz komisyonu, havuz sitesinde yayınlanır | yok |
| Güven varsayımı | operatörün dürüst ödemesi | yok |
| Kime uygun | evde bir iki tüketici kartı | özel bir rig, ya da zaten isteyeceğin bir node |

### Hangisini seçmelisin?

Şunlardan biri doğruysa **Yol A**'yı seç — evdeki çoğu kişi için en az biri doğrudur:

- Çiftlik değil, bir ya da iki tüketici GPU'n var.
- CGNAT, mobil bağlantı ya da yönetmediğin bir modem arkasındasın.
- Şansın dönmesini beklemek yerine hafta içinde gelir görmek istiyorsun.
- Sürekli senkron kalması gereken bir süreç istemiyorsun.

**Yol B**'yi seç:

- Zaten bir Quantus tam node'u çalıştırmak istiyorduysan.
- Blok varyansını kaldırabilecek kadar hash gücün varsa.
- Ödülleri üçüncü bir taraf üzerinden geçirmeye kesinlikle karşıysan.
- Gerçek IP'si, SSD'si ve kotasız bağlantısı olan bir sunucudaysan.

> **Bu karar kalıcı değil.** Cüzdan, adres ve GPU tarafı iki yolda da birebir aynı; sonradan geçmek bir akşamını alır — bkz. [Sonradan Yol Değiştirme](#sonradan-yol-değiştirme).

### Bu rehberin yerleşimi

```text
  Adım 1  Cüzdan oluştur          ─┐
  Adım 2  GPU'yu doğrula           ├─ bu ikisini herkes yapar
                                   │
  ── Karar Noktası ───────────────┘
         │
         ├── Yol A  A1 → A6   "Yol A Bitiş Çizgisi"nde biter
         │
         └── Yol B  B1 → B6   "Yol B Bitiş Çizgisi"nde biter
                 │
  Referans bölümleri, bitirdiğin yol için geçerlidir
```

**Sadece bir bölümü yap.** Aynı makinede ikisini birden çalıştırmak, tek GPU için kavga eden iki madenci demektir; ikisi de verimsiz olur.

---

## Donanım Gereksinimleri

| Bileşen | Asgari | Önerilen |
|---|---|---|
| İşletim sistemi | Ubuntu 20.04+, macOS, Windows 10/11 | Ubuntu 24.04 / 26.04 LTS |
| CPU | 2 çekirdek | 4+ çekirdek |
| RAM | 4 GB | 8 GB+ |
| Disk | 100 GB *(sadece Yol B)* | 500 GB+ SSD — SATA olur, HDD olmaz |
| Ağ | 3 Mbps | 10+ Mbps |
| GPU | yok (sadece CPU, çok yavaş) | NVIDIA RTX 20/30/40/50 serisi |

- Hash hızını **GPU ve madenci sürümü** belirler — disk boyutu ya da CPU önbelleği değil. Büyük önbellekli bir CPU Poseidon2'yi hızlandırmaz.
- QPoW **VRAM'e aç değildir**. 8–12 GB fazlasıyla yeter; 140 GB'lık bir veri merkezi kartı "50 kat hızlı" değildir ve hash başına maliyette genelde bir 4090'a yenilir.
- Node veritabanı RocksDB'dir ve rastgele I/O yapar. Her SSD iş görür; HDD senkronu tıkar.
- **Linux ARM64 için resmî madenci binary'si yok** — Linux x86_64 ya da macOS'tan kaz. AMD GPU'lar CUDA havuz madencisiyle çalışmaz.
- **Yol A yukarıdaki disk ve bant genişliği payına hiç ihtiyaç duymaz** — sadece GPU ve dışa UDP yeter. O satırlar Yol B için var.

---

## Portlar ve Uç Noktalar

| Port | Amaç | Yol | Ne yapmalı |
|---|---|---|---|
| `30333/TCP` | Node P2P | B | İnternete bakabilecek tek port. Opsiyonel — dışa bağlantıyla da senkron olur |
| `9833/UDP` | Madenci ↔ kendi node'un (QUIC) | B | **Sadece localhost ya da VPN.** `0.0.0.0`'a bağlanır; onu sadece güvenlik duvarın korur |
| `9834/UDP` | Havuz madencisi → havuz node'u | A | **Sadece dışa.** Dışa UDP kapalıysa sonsuz yeniden bağlanma döngüsü |
| `9944` | Node RPC | B | Localhost |
| `9615` | Node Prometheus metrikleri | B | Localhost |
| `9900` | Madenci metrikleri / `hive-stats` | A + B | Localhost |

Tüm resmî bağlantılar (dokümanlar, sürümler, cüzdan, explorer, telemetri, havuz) [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) içinde toplandı.

---

## Nereden Başlamalı

| Durumun | Başlangıç |
|---|---|
| Windows PC, NVIDIA kart, henüz Linux yok | **[ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md)** — aşağıdaki Adım 2'de biter |
| Saatlik GPU kiralıyorsun | **[vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md)** — kendi içinde tam bir Yol A varyantı |
| Ubuntu ya da macOS, sürücü çalışıyor | Adım 1'e geç |
| Sadece genel bakış ve bağlantılar | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

# Ortak Kurulum

İki yol da burada başlar. İki adım, sonra seçim.

## Adım 1 — Cüzdan Oluştur

Her şeyden önce bir `qz…` adresine ihtiyacın var. İki yol da aynı cüzdanı ve aynı 24 kelimeyi kullanır.

1. [Quantus Wallet](https://www.quantus.com/wallet/) uygulamasını kur (iOS / Android — bağlantılar ayrıca [linktr.ee/quantusnetwork](https://linktr.ee/quantusnetwork) üzerinde).
2. Cüzdanı oluştur ve **24 kelimelik ifadeyi kâğıda**, çevrimdışı yaz.
3. Ana ekrandaki adres `qz…` ile başlar. Bir havuz formuna ya da sorgu kutusuna yapıştıracağın tek değer budur.

CLI alternatifi:

```bash
# https://github.com/Quantus-Network/quantus-cli/releases adresinden
quantus wallet create --name mining
```

> **KRİTİK:** 24 kelime, ödüllerin tek kurtarma yoludur. Onları asla bir sohbet penceresine, havuz formuna ya da kiralık sunucuya yazma. Yol A **adresini** ister; Yol B o kelimelerden **türetilen inner hash'i** ister. Hiçbir yol, kelimelerin makineni terk etmesini gerektirmez.

**Bittiği an:** ekranda bir `qz…` adresi okuyabiliyorsan ve 24 kelime kâğıtta duruyorsa.

---

## Adım 2 — GPU'yu Doğrula (Linux)

Herhangi bir madenci kurmadan önce NVIDIA'nın kapalı kaynak sürücüsünün aktif olduğunu doğrula. Yanlış sürücü yolundaki bir madenci sessizce 4–6 kat yavaş çalışır.

> Henüz Linux yok mu? Önce **[ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md)**'yi bitir — seni buraya geri gönderir. Kiralık GPU'da sürücüyü zaten sağlayıcı enjekte eder: bu rehber yerine **[vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md)** kullan.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq

# sadece sürücü eksikse
sudo ubuntu-drivers autoinstall
sudo reboot
```

Yeniden başlattıktan sonra:

```bash
nvidia-smi
nvidia-smi -L      # her GPU için bir satır
```

Kart adını, sürücü sürümünü ve bir `CUDA Version` kolonunu görmelisin. Kullanımın düşük olması normal — henüz madenci çalışmıyor.

> Tam `cuda-toolkit`'e **ihtiyacın yok**; hazır madenciler kendi CUDA çalışma zamanını taşır. `nvidia-smi` hiçbir şey yazmıyorsa Secure Boot'u kapat (ya da MOK anahtarını kaydet) ve yeniden başlat. RTX 50 serisi (Blackwell) güncel bir çekirdek ve 570+ sürücü ister.

**Bittiği an:** `nvidia-smi -L` kartlarını adıyla listeliyorsa.

---

## Karar Noktası — Yolunu Seç

Ortak kurulum bitti. Bu çizgiden sonrası **ya bir yol ya diğeri**.

| Seçim | Git | Sonunda elde edeceğin |
|---|---|---|
| **Yol A — Havuz** | [Adım A1](#adım-a1--havuz-madencisini-kur) | Havuza hash basan tek bir `quanpool-miner` servisi, pay başına artan bakiye |
| **Yol B — Kendi node'un** | [Adım B1](#adım-b1--otomatik-kurulum-betiği) | Senkron bir `quantus-node` artı yerel `quantus-miner`, wormhole adresine tam blok ödülleri |

Hâlâ kararsız mısın? **Yol A**'yı yap. Bir saatte geri alınabilir, kartının ve adresinin çalıştığını kanıtlar ve sen node isteyip istemediğini düşünürken para kazanmaya başlar.

---

# Yol A — Havuz Madenciliği (Quanpool PPLNS)

> **Burada başlar.** Ön koşullar: Adım 1 (bir `qz…` adresi) ve Adım 2 (çalışan GPU sürücüsü).
> **Bittiği yer:** [Yol A Bitiş Çizgisi](#yol-a-bitiş-çizgisi) — aşağıdaki altı adımdan sonra, işçin havuz sitesinde göründüğünde.
> **Hiç dokunmayacağın şeyler:** `quantus-node`, zincir senkronu, içe açık güvenlik duvarı kuralları, inner hash. Bunların hiçbiri bu yola ait değil.

Sürtünmesi en az olan yol: node yok, senkron yok, içe açık port yok, CGNAT arkasında çalışır. Tek bir süreç havuza dışa doğru bağlanır ve pay başına ödeme alırsın.

> Quanpool bir **topluluk** havuzudur, resmî Quantus altyapısı değildir. Sunucu adresi, indirme bağlantısı ve TLS pin'i [quanpool.com](https://quanpool.com/) → **Start mining** sayfasında canlı yayınlanır. Her seferinde oradan kopyala; aşağıdaki değerler bilerek yer tutucudur.

### Adım A1 — Havuz Madencisini Kur

```bash
sudo mkdir -p /opt/quantus
sudo chown "$USER:$USER" /opt/quantus
cd /opt/quantus

wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version
```

O sürüm 404 verirse güncel Linux bağlantısını **Start mining**'den al (6.1.0 da çalışır).

> `gpu-list` alt komutu 6.1+ sürümlerinde kaldırıldı. Kartları `nvidia-smi -L` ile say.

### Adım A2 — Kartı Benchmark Et

Benchmark yerelde çalışır, havuza hiç bağlanmaz. Bunu **havuz ayarlarından önce** yap — bu rehberdeki en pahalı sorunun cevabını verir: acaba gerçekten CUDA kod yolunda mısın?

```bash
cd /opt/quantus
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
```

| Sonuç | Anlamı |
|---|---|
| Karta göre 400 MH/s – 1,5 GH/s | Doğru — Linux CUDA yolu |
| Modern bir RTX kartta ~100 MH/s | **Yanlış binary ya da sürücü** — stock/wgpu yolu, 4–6 kat yavaş. Bunu sonra değil, şimdi düzelt |

Masaüstünde `--cpu-workers 0` zorunlu kabul et. CPU madenciliği çekirdek başına ~15 MH/s ekler — kartın yanında önemsiz, ama kartı besleyen çekirdekleri çalar.

### Adım A3 — Token ve TLS Pinini Al

[quanpool.com](https://quanpool.com/) → **Start mining** sayfasında şunları doldur:

| Alan | Değer |
|---|---|
| Address | `qz…` adresin — **asla seed değil** |
| Worker | opsiyonel ad, **her makinede farklı**. `a-z 0-9 . - _`, en fazla 32 karakter, boşluk yok |
| Mode | **Pool (PPLNS)** |
| System | **Linux** |

Kayıt ve şifre yoktur: adres *zaten* hesabın. `--node-addr` değerini (alan adı yerine düz `IP:9834` tercih et) ve 64 haneli onaltılık `--tls-cert-sha256` değerini kopyala. İkisini de komut satırına yapıştırmak yerine dosyaya yaz, böylece kabuk geçmişine düşmezler:

```bash
cd /opt/quantus

# tek satır: qzADRESIN.iscininadı
nano auth-token

# Start mining'deki 64 haneli TLS pin
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

Token sadece adresin, bir nokta ve işçi adından oluşur:

```text
qzADRESIN.rig1
```

### Adım A4 — Manuel Çalıştırma

systemd'ye devretmeden önce bir kez ön planda çalıştır ki hataları okuyabilesin.

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

`<POOL_HOST>` yerine **Start mining**'deki adresi yaz ve `--gpu-devices` değerini `nvidia-smi -L` çıktısındaki kart sayısına ayarla (hepsini kullanmak için parametreyi hiç vermeyebilirsin).

İlk dakikada sağlıklı belirtiler:

- `nvidia-smi` madenci sürecini ve %95–100 GPU kullanımını gösterir
- log iş satırlarını ve `SHARE FOUND` yazar
- `curl -s http://127.0.0.1:9900/hive-stats` dolu bir `hs` ve sıfır reddetmeli bir `ar` döndürür

> İlk saniyelerde bir grup `SOLUTION LOST` / stale mesajı normaldir; madenci güncel işe yetişiyordur.

Sağlıklı göründüğünde `Ctrl+C` ile durdur. Terminalde bırakma — sonraki adım onu yeniden başlatmalara dayanıklı hale getiriyor.

### Adım A5 — systemd Servisi Oluştur

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

Başlatmadan önce `ExecStart` içindeki `<POOL_HOST>`'u düzelt. `auth-token` dosyası `600` modunda, dolayısıyla burada verdiğin `User=` tarafından okunabilir olmalı.

Masaüstü kurulumda uyku modunu da kapatmak şart, aksi halde ekran kararınca GPU durur:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

> Kiralık bir konteynerde genelde çalışan bir `systemd` bulunmaz. Onun yerine imajın süreç yöneticisini kullan — bkz. [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md).

### Adım A6 — Havuzda Doğrula

Önce yerelde:

```bash
curl -s http://127.0.0.1:9900/hive-stats
```

`hs` genelde **kH/s** cinsindendir — `430000` ≈ 430 MH/s. `ar` değeri `[kabul, red]` demektir ve red sıfırda kalmalı; `temp`, `fan` ve `pool_rtt_ms` da raporlanır.

Sonra [quanpool.com](https://quanpool.com/) adresini aç, `qz…` adresini sorgu kutusuna yapıştır ve **Look up**'a bas. İşçin birkaç dakika içinde kendi hash hızı, pay sayısı ve büyüyen bir bekleyen bakiyeyle listelenir.

> `9900`, `9833`, `9944` ve `9615` portları asla internete açılmamalı. Modemde bunlar için "virtual server" kuralı hiçbir işe yaramaz, sadece gerçek bir risk yaratır.

### Yol A Bitiş Çizgisi

**Şu dördü doğruysa Yol A tamamlandı:**

1. `systemctl status quanpool-miner` `active (running)` diyor ve yeniden başlatmadan sağ çıkıyor.
2. `nvidia-smi` kartı %95–100 kullanımda, Adım A2 benchmark'ındaki hızda gösteriyor.
3. `hive-stats` kabul edilen payları artarken, reddedilenleri sıfırda gösteriyor.
4. İşçin havuz sorgu sayfasında görünüyor ve bekleyen bakiye yükseliyor.

**Yolun sonu burası.** Kuracak başka bir şey yok: node yok, senkron yok, inner hash yok. Buradan sonra günlük beş on komut için [Günlük Komutlar ve Takip](#günlük-komutlar-ve-takip), ikinci bir rig eklemek için [Birden Fazla Makine](#birden-fazla-makine), sayıların anlamını çözmek için [Ekonomi](#ekonomi--ne-beklemeli) bölümüne geç.

**Yol B'yi tamamen atla** — ta ki bir gün kendi node'unu istemeye karar verene kadar; o zaman da önce [Sonradan Yol Değiştirme](#sonradan-yol-değiştirme)'yi oku.

---

# Yol B — Kendi Node'un (Resmî Yol)

> **Burada başlar.** Ön koşullar: Adım 1 (24 kelime — inner hash'i onlardan türeteceksin) ve Adım 2 (çalışan GPU sürücüsü). Artı 100 GB+ SSD ve açık bırakmaya razı olduğun bir bağlantı.
> **Bittiği yer:** [Yol B Bitiş Çizgisi](#yol-b-bitiş-çizgisi) — node'un zincir ucuna yetiştiğinde ve yerel bir madenci onu beslediğinde.
> **İki geçiş yolu var:** Adım B1 resmî betiktir ve her şeyi senin yerine yapar. Adım B2–B5 aynı işin elle hâlidir. **Ya B1'i yap, ya B2–B5'i — ikisini birden değil.** B6 her durumda geçerli.

Havuz komisyonu yok ve varlık tamamen sende; karşılığında tam bir zincir senkronu ve eşleşen bir binary çifti. Linux, macOS ve WSL2'de çalışır.

### Adım B1 — Otomatik Kurulum Betiği

Resmî betik wormhole inner hash'ini ve node kimliğini üretir, eşleşen bir çifti `~/quantus-mining/bin/` içine indirir ve `~/quantus-mining/mining.conf` dosyasını `CHAIN=mainnet` ile yazar. İşini görürse doğrudan [Yol B Bitiş Çizgisi](#yol-b-bitiş-çizgisi)'ne atlayıp senkronu bekleyebilirsin.

```bash
curl -fsSL https://docs.quantus.com/scripts/quantus-mining.sh -o quantus-mining.sh
chmod +x quantus-mining.sh
./quantus-mining.sh setup
./quantus-mining.sh start -d
```

Günlük kontrol:

```bash
./quantus-mining.sh start          # node ön planda, madenci arkada
./quantus-mining.sh start-node     # bir terminal
./quantus-mining.sh start-miner    # ikinci terminal, madenci sunucusu dinlemeye başladıktan sonra
./quantus-mining.sh stop
./quantus-mining.sh config show
./quantus-mining.sh config set GPU_DEVICES 1
./quantus-mining.sh config set CPU_WORKERS 0
```

İki `latest` etiketinin birbirinden ayrı düşmesine izin vermek yerine sürümleri sabitle:

```bash
./quantus-mining.sh config set NODE_VERSION v1.0.1
./quantus-mining.sh config set MINER_VERSION v4.2.0
./quantus-mining.sh stop
./quantus-mining.sh setup --force
./quantus-mining.sh start -d
```

`--force` yalnızca binary'leri yeniler: `INNER_HASH` ve wormhole adresin korunur, **yeni anahtar çifti üretilmez**. Betik node'un `miner-auth-token` ve `miner-tls-cert-sha256` dosyalarını kendisi okur, yani bunları elle kopyalamazsın. Docker modu kaldırıldı.

> **Betik işini gördüyse burada dur** ve [Yol B Bitiş Çizgisi](#yol-b-bitiş-çizgisi)'ne geç. Adım B2–B5 aynı sonucu elle kurar — dizinler, sürümler ve servis yönetimi üzerinde kontrol istediğinde ya da betik dağıtımında patladığında işine yarar.

### Adım B2 — Manuel Kurulum: Binary'ler

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

Güncel etiketler için [Releases](https://github.com/Quantus-Network/chain/releases) sayfasına bak — resmî tavsiye **node v1.0.1 ya da üstü** ve iki depo birbirinden bağımsız sürümleniyor.

Devam etmeden **önce** çiftin kimlik doğrulamalı protokolü konuştuğunu teyit et — iki komut da bir eşleşme yazdırmalı:

```bash
./quantus-node --help | grep miner-auth-token-file
./quantus-miner serve --help | grep auth-token-file
```

| Platform | Node arşivi | Madenci binary'si |
|---|---|---|
| Linux x86_64 | `quantus-node-<etiket>-x86_64-unknown-linux-gnu.tar.gz` | `quantus-miner-linux-x86_64` |
| Linux ARM64 | node var | **resmî madenci yok** |
| macOS Apple Silicon | `…-aarch64-apple-darwin.tar.gz` | `quantus-miner-macos-aarch64` |
| Windows (yerel) | `…-x86_64-pc-windows-msvc.zip` | `quantus-miner-windows-x86_64.exe` |

macOS'ta önce Gatekeeper işaretini kaldır: `xattr -d com.apple.quarantine quantus-node`.

### Adım B3 — Node Kimliği ve Wormhole Inner Hash

```bash
./quantus-node key generate-node-key --file node_key.p2p
./quantus-node key quantus --scheme wormhole --words
```

`--words` 24 kelimeyi **ekrana yazdırmadan** sorar, böylece ifade kabuk geçmişine hiç düşmez. Yazdırdığı değerleri sakla:

| Değer | Nedir | Ne yapılır |
|---|---|---|
| **Address** | wormhole adresin — ödüllerin düştüğü yer | takip için sakla |
| **Inner Hash** | 32 baytlık ön görüntü | `--rewards-inner-hash` olarak ver |
| **Secret** | sahipliği kanıtlayan anahtar | çevrimdışı yedekle, asla paylaşma |

Cüzdan uygulamandaki 24 kelimenin aynısını kullanmak önerilen yoldur — ödüller o zaman uygulamada kendiliğinden görünür. Sıfırdan bir cüzdana kazmak istersen `./quantus-node key quantus --scheme wormhole` çalıştır ve ürettiği ifadeyi yedekle. Ödüllerin wormhole adresine yönlenmesi **opsiyonel değildir**, protokole gömülüdür; aynı sebeple madencilik kimliğin zincir üzerinde ödeme adresinle ilişkilendirilemez.

### Adım B4 — Node'u Başlat

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

`--name`, node'unun [telemetri](https://telemetry.quantus.cat/) sayfasında görünen adıdır.

`--miner-listen-port` ile ilk açılışta node, madenci kimlik doğrulama malzemesini zincir dizinine yazar:

| Dosya | Amaç |
|---|---|
| `miner-auth-token` | madencinin `Ready` içinde gönderdiği paylaşılan sır. Mod `0600`, loglanmaz |
| `miner-tls-cert-sha256` | madenci QUIC sertifikasının SHA-256'sı; madenciler bunu sabitler |
| `miner-tls-cert.der` / `miner-tls-key.der` | node TLS malzemesi — özel anahtarı asla bir madenciye kopyalama |

| Platform | Zincir dizini |
|---|---|
| Linux | `~/.local/share/quantus-node/chains/mainnet/` |
| macOS | `~/Library/Application Support/quantus-node/chains/mainnet/` |

> ⚠️ **Ödül beklemeden önce tam senkronu bekle.** Zincir ucuna varmadan kazılan bloklar öksüz kalır ve hiçbir şey kazandırmaz; node'un peer'i yokken madenci kendiliğinden bekler. Log `Syncing`'den güncel yükseklikte `Idle`'a döndüğünde senkronsun — tipik olarak 15 dakika ile birkaç saat arası. Senkron sırasında `discarding proposal` normaldir. `Verification failed` ve 0 peer ile takılma, node sürümünün ağdan koptuğu anlamına gelir — [Releases](https://github.com/Quantus-Network/chain/releases)'a bak.

Ayrıca madenci sunucusunun dinlemeye başladığını doğrulayan log satırını bekle. Madenci sunucusu açılışta patlarsa node kapanır — yerel madenciliğe geri dönüş yoktur.

### Adım B5 — Harici Madenciyi Başlat

İkinci bir terminalde:

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

Varyasyonlar: Vulkan'ın olmadığı bir NVIDIA kartta `--cuda-gpu` ekle; kullanılabilir GPU'su olmayan makinede `--cpu-workers 4 --gpu-devices 0` kullan; ısıyı ve güç çekişini sınırlamak için `--gpu-throttle-ms 50` (masaüstünü de kullanıyorsan `5`) ekle.

> Sırların kabuk geçmişinde kalmaması için satır içi biçim yerine her zaman `--auth-token-file` / `--tls-cert-sha256-file` kullan. Yanlış token ya da pin **kalıcı** bir hatadır: madenci yeniden bağlanma döngüsüne girmez, durur. Dosyaları yeniden oku.
>
> macOS'ta `CHAIN_DIR`'i tırnak içinde kullan — yolda boşluk var.

### Adım B6 — Node/Madenci Çiftini Güncelleme

Hem betikli hem elle kurulum için geçerli:

1. Önce node'u, sonra madenciyi durdur.
2. **Eşleşen** bir çift indir — ikisi de `quantus-miner/2` konuşmalı.
3. `.../chains/mainnet/` dizinini koru. Asla bir `chains/planck/` dizinini içe alma.
4. Node'u başlat, madenci sunucusu dinlemeye geçsin, sonra madenciyi başlat.

### Yol B Bitiş Çizgisi

**Şu beşi doğruysa Yol B tamamlandı:**

1. Node logu güncel zincir yüksekliğinde `Idle` diyor ve peer sayısı stabil.
2. `--name` değerin [telemetry.quantus.cat](https://telemetry.quantus.cat/) üzerinde görünüyor.
3. Node, madenci sunucusunun `9833` portunda dinlediğini loglamış.
4. Madenci bağlı, `nvidia-smi` %95–100 kullanım gösteriyor ve node tarafında `Broadcasting job` satırları akıyor.
5. `ufw status` sadece `22/tcp` ve `30333/tcp` izniyle duruyor, **başka hiçbir şey yok** — `9833`, `9944` ve `9615` kapalı.

**Yolun sonu burası.** Ödüller artık düzensiz aralıklarla, tam bloklar hâlinde, Adım B3'teki wormhole adresine gelir — cüzdan uygulamasından ya da explorer'dan kontrol et. Devamı için [Günlük Komutlar ve Takip](#günlük-komutlar-ve-takip), sonra da bloklar arası sessizlik seni telaşlandırmasın diye [Ekonomi](#ekonomi--ne-beklemeli).

---

# Referans

Hangi yolu bitirdiysen onun için geçerli.

## Günlük Komutlar ve Takip

**Yol A — havuz**

```bash
sudo systemctl status quanpool-miner --no-pager
sudo systemctl restart quanpool-miner
journalctl -u quanpool-miner -f
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw,fan.speed --format=csv

# yeniden benchmark, ya da dışa UDP testi (nc'den yanıt gelmemesi tek başına kanıt değildir)
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
nc -zvu <POOL_HOST> 9834
```

Havuzun sorgu sayfası işçileri, bekleyen bakiyeyi ve ödemeleri gösterir. Binary'yi güncellemek için: servisi durdur, yeni dosyayı indir, `chmod +x`, tekrar başlat. Mevcut sürüm çalışıyorsa çıkan güncelleme acil değildir.

**Yol B — kendi node'un**

| Ne | Nerede |
|---|---|
| Ödüller | cüzdan uygulaması (aynı 24 kelime), ya da `https://explorer.quantus.com/accounts/<YOUR_QZ_ADDRESS>` |
| Telemetride node | [telemetry.quantus.cat](https://telemetry.quantus.cat/) — `--name` değerini ara |
| Node metrikleri / RPC | `http://localhost:9615/metrics` · `http://localhost:9944` |
| Madenci metrikleri | `http://localhost:9900/metrics` |

```bash
./quantus-node --version
./quantus-node key inspect-node-key --file node_key.p2p
tail -f ~/.local/share/quantus-node/chains/mainnet/network/quantus-node.log
```

Sağlıklı bir node'da peer sayısı stabildir, zincir ucunda `Idle` görürsün, `Prepared block for proposing` ve `Broadcasting job` satırları akar.

---

## Performans Referansı

Linux CUDA havuz madencisinin stock madenciye karşı yayınlanmış değerleri, sadece GPU. Kendi kartını her zaman `benchmark --duration 30` ile teyit et.

| Kart | Stock madenci | Linux CUDA havuz madencisi | Oran |
|---|---|---|---|
| RTX 5090 | 342 MH/s | 1,49 GH/s | ~4,4× |
| RTX 4090 | 179 MH/s | 1,12 GH/s | ~6,3× |
| RTX 4070 Ti | 122 MH/s | 574 MH/s | ~4,7× |
| RTX 4070 | — | ~430–460 MH/s | — |
| RTX 5070 | ~100 MH/s | ~410–450 MH/s | ~4,2× |
| RTX 3080 Ti | 104 MH/s | 435 MH/s | ~4,2× |
| RTX 5060 Ti | 75 MH/s | 314 MH/s | ~4,2× |
| CPU, 8 worker | toplam ~120 MH/s | — | çekirdek başına ~15 MH/s |

Komisyon ve şansı yok sayan kaba gelir tahmini:

```text
günlük QTC ≈ (senin H/s / ağın H/s) × (86400 / blok_saniyesi) × blok_ödülü
```

Her terim oynar. Zorluk her sonlanan blokta yeniden ayarlanır, dolayısıyla tek bir tüketici kartı TH/s ile ölçülen bir ağın küçük bir kesridir. Herhangi bir para birimi rakamını doğrulanmamış kabul et.

**Isı:** 7/24 kazan bir kart güç limitine yakın, fanları yukarıda çalışır. Sürekli ~70 °C normaldir ve ~83–88 °C'lik throttle noktasının çok altındadır; gerçek bakım kalemleri toz ve kasa hava akışıdır. Kart sürekli 85 °C'de duruyorsa `--gpu-throttle-ms 5` ekle.

---

## Güvenlik Duvarı

**Yol A hiçbir içe kural gerektirmez** — sadece havuz portuna dışa UDP. Modemin ya da operatörün dışa UDP/QUIC'i engelliyorsa madenci yeniden bağlanma döngüsüne girer; teyit için telefon hotspot'undan dene.

**Yol B:**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 30333/tcp
sudo ufw enable
sudo ufw status verbose
```

`9833`, `9944` ya da `9615` portlarını **açma**. Uzaktaki bir madenciyi node'a WireGuard veya Tailscale ile bağla, madenci portunu özel tut.

CGNAT arkasında (çoğu mobil ve pek çok ev fiber bağlantısı) içe gelen `30333` hiç ulaşmaz — kural zararsızdır ama bir işe yaramaz, node yine dışa bağlantılarla senkron olur. Modemde port yönlendirme CGNAT'ı çözemez; bunu ancak genel bir IP ya da bir VPN uç noktası çözer. Yol A'nın evde daha kolay olmasının sebebi tam olarak budur.

---

## Birden Fazla Makine

Birden fazla madenci **aynı** `qz…` adresine ödeme yapabilir; PPLNS payları toplanır.

- Her makineye **farklı bir işçi adı** ver. Aynı adı kullanmak iki rig'i çakıştırır ve biri havuzdan kaybolur.
- İkinci bir makine eklemek için evdeki rig'i durdurmana gerek yok.
- Seed'ini asla kiralık bir makineye kopyalama. Kiralık kutuya sadece `qzADRES.isci` ve TLS pin gerekir — bkz. [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md).
- Yol B tarafındaki karşılığı, VPN üzerinden tek node'un `9833` portuna bağlanan birkaç madencidir. Node işleri hepsine yayınlar, geçerli sonucu ilk bulan kazanır.

---

## Sorun Giderme

### Yol A — havuz

| Belirti | Neye bakmalı |
|---|---|
| `timed out` / bitmeyen yeniden bağlanma | Havuz portuna dışa **UDP**. Ev güvenlik duvarları ve bazı operatör yolları QUIC'i keser — telefon hotspot'undan dene |
| `certificate` / `fingerprint` hatası | TLS pin'i **Start mining**'den yeniden kopyala; tam 64 onaltılık karakter |
| İşçi sitede hiç görünmüyor | `auth-token` içindeki adres sorguladığından farklı, ya da işçi adı kurallara uymuyor. 2–3 dakika tanı |
| Modern kartta benchmark ~100 MH/s | CUDA sürümü değil — yanlış binary ya da sürücü. `nvidia-smi`'yi kontrol et |
| Hâlâ `127.0.0.1:9833`'ü gösteriyor | O Yol B adresi; havuz madenciliği havuz sunucusunun `9834` portunu kullanır |
| Servis `running` ama GPU %0 | Logu oku: token, pin ya da `--node-addr` yanlış |
| Rig ekledikten sonra evdeki işçi kayboldu | Aynı işçi adı. Birini yeniden adlandır ve ikisini de başlat |
| Makine boşa çıkınca madencilik duruyor | Uyku hâlâ açık — Adım A5'teki `gsettings` komutlarını tekrar uygula |

### Yol B — kendi node'un

| Belirti | Neye bakmalı |
|---|---|
| Madenci hemen kapanıyor | Kimlik doğrulama ya da sürüm: madenci sunucusunun dinlemesini bekledin mi, iki auth dosyası var mı, iki binary de `quantus-miner/2` konuşuyor mu |
| TLS `no application protocol` | Node ve madenci uyumsuz bir çift |
| Yanlış token ya da pin | Tasarımı gereği kalıcı hata — dosyaları yeniden oku, körlemesine tekrar denemeyin |
| `Verification failed`, 0 peer | Node sürümü ağdan kopmuş |
| Senkron hiç bitmiyor | Bant genişliği 3 Mbps altında, ya da SSD yerine HDD |
| Windows'ta senkron takılıyor | Defender RocksDB dizinini tarıyor — dışlama ekle, ya da WSL2 kullan |
| Kazıyor ama ödül yok | Hâlâ senkron (öksüz bloklar), yanlış inner hash, ya da cüzdanından farklı bir seed |
| Node madenci portunda kapanıyor | Bind, TLS ya da token hatası. Yerel madenciliğe geri dönüş yok |
| Linux ARM64'te madenci yok | x86_64 ya da macOS'tan kaz |

Windows'ta senkrondan önce node veri dizinini Defender'dan dışla:

```powershell
Add-MpPreference -ExclusionPath "$env:USERPROFILE\.quantus"
```

> Kimlik doğrulama öncesi sürümlerde (node v0.9.0, madenci v3.3.1 ve öncesi) madenci doğrulaması yoktur. Bunları güncel mainnet'e karşı kullanma.

---

## Ekonomi — Ne Beklemeli

**Emisyon.** Blok ödülleri `(MaxSupply − CurrentSupply) / EmissionDivisor` formülünü izler — sabit 21.000.000 arzın yumuşak eksponansiyel azalması, halving uçurumu yok. Madenciler zamanla **arzın %50'sini** alır ve tüm arzın kabaca %99'u yaklaşık 40 yılda dağıtılır. Blok ödülleri üzerindeki geliştirici vergisi %15'i şirkete ayırır, yıllar içinde hak edilerek.

**Zincir üstü ücretler.** Standart transferler madenciye giden sabit bir ücret öder. Yüksek güvenlikli geri alınabilir transferler hacme bağlı bir ücret öder ve o ücret yakılır; ZK'da toplanan işlemler ise madenci ile yakma arasında bölünen daha küçük bir hacim ücreti öder. Dolayısıyla madencilik geliri ağırlıklı olarak ücretlerden değil emisyondan gelir.

**Yol A — PPLNS nasıl öder.** Sana **blok başına değil, pay başına** ödenir. Hiçbir blok "senin" değildir; bakiyen sürekli birikir ve bütün mesele de budur — varyansı ortadan kaldırır. Havuz sabit bir komisyon alır ve tek bir eşiğin üstünde ödeme yapar; ikisi de havuz sitesinde yayınlanır, ikisi de zaman içinde değişmiştir. Bu yüzden o değerleri hiçbir rehberdeki sayıya değil (bu rehber dâhil) siteye bakarak öğren. Havuz içindeki solo modu aynı komisyonu taşır ama loto gibi öder.

**Yol B — solo nasıl öder.** Sıfır komisyon, tam kontrol ve kazandığın her seferde bir tam blok ödülü. TH/s ile ölçülen bir ağa karşı tek bir tüketici kartıyla bu, uzun sessizlikler demek olabilir. Bir şey bozuk değil; varyans, paylaşmamanın bedeli.

**Gelirini ne oynatır.** Zorluk her sonlanan blokta yeniden ayarlanır, yani ağın hash gücü artarken rig'in hiç değişmese bile günlük QTC'n düşer. Kendi sayıların her tablodan değerlidir: `benchmark` ile ölç, sonra havuzun canlı ağ hash gücüyle karşılaştır.

> QTC piyasa verisi sığdır ve birim, resmî dokümanların ve araçların bir kısmında `QUAN` olarak etiketlenir — alakasız tickerlarla karıştırmak çok kolay. Herhangi bir fiyat projeksiyonunu spekülatif kabul et.

---

## Sonradan Yol Değiştirme

İki yol cüzdanı, adresi ve GPU tarafını paylaşır; bu yüzden geçiş ucuzdur.

**A → B (havuzdan kendi node'una)**

```bash
sudo systemctl disable --now quanpool-miner
```

Sonra [Adım B1](#adım-b1--otomatik-kurulum-betiği)'den başla. Inner hash'i **aynı** 24 kelimeden türet, ödüller aynı cüzdan uygulamasına düşmeye devam eder. Havuzdaki bekleyen bakiyeye dokunma — normal takviminde ödenir.

**B → A (kendi node'undan havuza)**

Önce madenciyi, sonra node'u durdur. Geri dönme ihtimalin varsa `chains/mainnet/` dizinini sakla; yeniden senkrondan kurtarır. Sonra aynı `qz…` adresiyle [Adım A1](#adım-a1--havuz-madencisini-kur)'den başla.

**İkisini birden çalıştırmak bir strateji değil.** Tek GPU'daki iki madenci birbirini yarıya düşürür. İki kartın varsa her yola `--gpu-devices` ile ayrı bir kart ve ayrı servis ver — yoksa birini seç.

---

## Sıradaki Adımlar

| Hedef | Doküman |
|---|---|
| Windows PC'yi madencilik makinesine çevirmek | [ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md) |
| Saatlik GPU kiralamak | [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md) |
| Ağ bilgileri, bağlantılar ve yol karşılaştırması | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

## Yazar Hakkında

Bu rehber **HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
