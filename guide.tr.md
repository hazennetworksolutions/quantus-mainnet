<div align="center">

# ⚛️ Quantus Mainnet GPU Madencilik & Tam Node Rehberi

**Quantus mainnet'te QTC kaz — havuza katıl ya da kendi node'unu harici GPU madencisiyle çalıştır**
*İki eksiksiz yol, tek ortak kurulum: cüzdan, CUDA madencisi, systemd servisi, node senkronu, izleme, sorun giderme.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%2B%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![Node](https://img.shields.io/badge/Node-v1.0.1%2B-brightgreen?style=flat-square)](https://github.com/Quantus-Network/chain/releases)
[![Miner](https://img.shields.io/badge/Madenci%20Protokolü-quantus--miner%2F2-blue?style=flat-square)](https://docs.quantus.com/deep-dives/miner-protocol/)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Yazar:** HazenNetworkSolutions
> **Ağ:** Quantus Mainnet (`--chain mainnet`)
> **Sürümler:** `quantus-node` v1.0.1+ · `quantus-miner` v4.2.x · havuz madencisi 6.2.0
> **Son Güncelleme:** Eylül 2026

---

## İçindekiler

**Önce oku**

- [Madencilik Nasıl İşliyor](#madencilik-nasıl-işliyor)
- [Hangi Yol — A mı B mi](#hangi-yol--a-mı-b-mi)
- [Donanım Gereksinimleri](#donanım-gereksinimleri)
- [Portlar ve Uç Noktalar](#portlar-ve-uç-noktalar)

**Ortak kurulum — herkes yapıyor**

- [Adım 1 — Cüzdan Oluştur](#adım-1--cüzdan-oluştur)
- [Adım 2 — GPU'yu Doğrula (Linux)](#adım-2--gpuyu-doğrula-linux)
- [Karar Noktası — Yolunu Seç](#karar-noktası--yolunu-seç)

**Sonrasında tek yol, ikisi birden değil**

- [Bölüm A — Havuz Madenciliği (Quanpool PPLNS)](#bölüm-a--havuz-madenciliği-quanpool-pplns)
  - [A1 — Havuz Madencisini Kur](#a1--havuz-madencisini-kur)
  - [A2 — Kartı Benchmark Et](#a2--kartı-benchmark-et)
  - [A3 — Token ve TLS Pini](#a3--token-ve-tls-pini)
  - [A4 — İlk Manuel Çalıştırma](#a4--ilk-manuel-çalıştırma)
  - [A5 — systemd Servisi](#a5--systemd-servisi)
  - [A6 — Havuzda Doğrula](#a6--havuzda-doğrula)
  - [Bölüm A Bitiş Çizgisi](#bölüm-a-bitiş-çizgisi)
- [Bölüm B — Kendi Node'un](#bölüm-b--kendi-nodeun)
  - [B1 — Otomatik Kurulum Betigi](#b1--otomatik-kurulum-betigi)
  - [B2 — Manuel Kurulum: Binary'ler](#b2--manuel-kurulum-binaryler)
  - [B3 — Node Kimliği ve Wormhole Inner Hash](#b3--node-kimliği-ve-wormhole-inner-hash)
  - [B4 — Node'u Başlat](#b4--nodeu-başlat)
  - [B5 — Harici Madenciyi Başlat](#b5--harici-madenciyi-başlat)
  - [B6 — Node/Madenci Çiftini Güncelleme](#b6--nodemadenci-çiftini-güncelleme)
  - [Bölüm B Bitiş Çizgisi](#bölüm-b-bitiş-çizgisi)

**Referans — iki yol için de**

- [İzleme ve Günlük Komutlar](#izleme-ve-günlük-komutlar)
- [Performans Referansı](#performans-referansı)
- [Güvenlik Duvarı](#güvenlik-duvarı)
- [Çoklu Makine Çalıştırmak](#çoklu-makine-çalıştırmak)
- [Sorun Giderme](#sorun-giderme)
- [Ekonomi — Ne Beklemeli](#ekonomi--ne-beklemeli)
- [Sonradan Yol Değiştirmek](#sonradan-yol-değiştirmek)

---

## Madencilik Nasıl İşliyor

Quantus madenciliği her zaman **iki iş** demektir ve bu ayrım geri kalan her şeyi açıklıyor:

1. **Node** (`quantus-node`) mainnet'e bağlanır, zincirin tam kopyasını tutar, blok adaylarını hazırlar ve madencilik işlerini dağıtır. `--miner-listen-port` ile başlatıldığında aynı zamanda `9833` portunda bir QUIC sunucusu olur.
2. **Madenci** başka hiçbir şey yapmaz; GPU ya da CPU üzerinde kazanan nonce'u arar. Madenci her zaman *istemcidir*: node'u arar, tersi olmaz.

Algoritma **QPoW** — SHA-256 yerine çift Poseidon2 hash'i. Bir nonce, `Poseidon2(Poseidon2(block_hash ‖ nonce))` zorluk hedefinin altına düştüğünde kazanır. Poseidon2'nin seçilme sebebi ZK devresi içinde ucuz doğrulanması, SHA-256'nın zayıf olması değil.

Node'un gömülü CPU madencisi test içindir — iş parçacığı başına kabaca **15 MH/s**. Gerçek hız harici GPU madencisinden gelir: modern kart başına **500 MH/s – 1,5 GH/s**. Tek node'a birden fazla madenci bağlanabilir; ilk geçerli sonuç kazanır.

**Asıl soru, birinci işi kimin yaptığı.** Node'u kendin çalıştırırsan hattın tamamı senin olur. Havuza katılırsan node'u operatör işletir, senin madencin ise UDP üzerinden `host:9834`'e dışa doğru bağlanır — bu yüzden havuz madenciliği CGNAT arkasında, port yönlendirmesiz, statik IP'siz ve zincir senkronu olmadan çalışır.

İki durumda da ödüller normal bir adrese düşmez. Protokol, *inner hash* denen 32 bytelık bir öngörüntüden türeyen bir **wormhole adresine** ödeme yapar. Bunu cüzdan uygulamandaki aynı 24 kelimeden türetirsen ödüller orada, harcanabilir halde, talep işlemi gerekmeden görünür.

### Bilmeye değer zincir bilgileri

| Parametre | Değer |
|---|---|
| Algoritma | QPoW — çift Poseidon2 |
| Adres formatı | SS58 öneki 189 — adresler `qz…` ile başlar |
| Maksimum arz | 21.000.000, 12 ondalık |
| Zorluk ayarı | Her finalize blokta, blok başına sınırlı — 2016 bloklu dönem yok |
| Çatal seçimi | En uzun değil, kümülatif işe göre en ağır zincir |
| Finalizasyon | Tepenin 179 blok gerisi (maks. reorg derinliği 180) |
| Blok ödülü | `(MaxSupply − CurrentSupply) / EmissionDivisor` — yumuşak azalma, halving yok |

> ⚠️ Her zaman `--chain mainnet` ver. `planck` kapanan testnet: ayrı zincir, ayrı veritabanı, bakiye taşınması yok. `chains/planck/` dizinini asla `chains/mainnet/` içine kopyalama ve mainnet'te asla `--force-authoring` kullanma — o parametre sıfırdan ağ başlatmak için.

---

## Hangi Yol — A mı B mi

İki ayrı yarı, birbirini dışlar. **Hiçbir şey kurmadan önce bunu oku**: karar neyi indireceğini, güvenlik duvarında neyi açacağını ve nasıl ödeme alacağını değiştiriyor.

### Bölüm A — Havuz Madenciliği

Sadece **madenci** çalıştırırsın. Madenci dışa doğru bir havuz node'una bağlanır, o node'un gönderdiği işleri yapar ve pay (share) gönderir. Blokları havuz bulur, ödülü pay sayısına göre bölüştürür (PPLNS).

- **Kurduğun:** tek binary, tek systemd servisi.
- **Gereken:** bir `qz…` adresi, bir GPU, dışa açık UDP.
- **Gerekmeyen:** senkron zincir, disk alanı, açık gelen port, genel IP.
- **Ödeme:** küçük miktarlar, sürekli, ödeme eşiğini geçtikten sonra.
- **İlk hash'e süre:** 15–30 dakika.

### Bölüm B — Kendi Node'un

**İki işi de kendi donanımında** yaparsın: mainnet'i senkronlayan `quantus-node` ve ona `127.0.0.1:9833` üzerinden bağlanan `quantus-miner`.

- **Kurduğun:** eşleşmiş iki binary, node kimliği, wormhole inner hash.
- **Gereken:** 100 GB+ SSD, stabil bant genişliği, ilk senkron için sabır.
- **Ödeme:** node'un kazandığı blokta ödülün **tamamı** — arasında hiçbir şey.
- **Ayrıca:** sıfır komisyon, tam kontrol, operatöre güven gerektirmez ve ağı güçlendiren bir node.
- **İlk hash'e süre:** birkaç saat, çoğu senkron.

### Yan yana

| | **Bölüm A — Havuz** | **Bölüm B — Kendi node** |
|---|---|---|
| Çalışan süreç | 1 (madenci) | 2 (node + madenci) |
| Zincir senkronu | yok | tam senkron zorunlu |
| Disk | ihmal edilebilir | 100 GB+ SSD, HDD olmaz |
| CGNAT arkasında | evet, tasarım gereği | evet, dışa bağlantı yeter |
| Gelen port | yok | `30333/TCP` opsiyonel |
| Gelirin şekli | düzenli akış | seyrek toplu, yüksek varyans |
| Komisyon | havuz komisyonu, havuz sitesinde yayınlı | yok |
| Güven varsayımı | operatör dürüst öder | yok |
| Kime uygun | evde bir iki tüketici kartı | ayırılmış rig ya da zaten istediğin bir node |

### Hangisini seçmelisin?

Şu maddelerden biri doğruysa **Bölüm A** — evde genelde biri doğrudur:

- Çiftlik değil, bir iki tüketici GPU'n var.
- CGNAT, mobil bağlantı ya da yönetmediğin bir modümun arkasındasın.
- Şansı beklemek yerine hafta içinde gelir istiyorsun.
- Sürekli senkron kalması gereken bir süreç istemiyorsun.

**Bölüm B**'yi seç:

- Zaten bir Quantus tam node'u çalıştırmak istiyordun.
- Blok varyansını kaldıracak kadar hash hızın var.
- Ödülleri üçüncü bir taraftan geçirmeyi reddediyorsun.
- Gerçek IP'li, SSD'li, kotasız bir sunucudasın.

> **Karar kalıcı değil.** Cüzdan, adres ve GPU'nun yaptığı iş iki yolda aynı — bkz. [Sonradan Yol Değiştirmek](#sonradan-yol-değiştirmek).

### Bu rehberin düzeni

```text
  Adım 1  Cüzdan oluştur          ─┐
  Adım 2  GPU'yu doğrula           ├─ ikisini herkes yapar
                                    │
  ── Karar Noktası ──────────────┘
         │
         ├── Bölüm A  A1 → A6   "Bölüm A Bitiş Çizgisi"nde biter
         │
         └── Bölüm B  B1 → B6   "Bölüm B Bitiş Çizgisi"nde biter
                 │
  Referans bölümleri bitirdiğin yol için geçerlidir
```

**Tam olarak bir bölüm yap.** İkisini aynı makinede yapmak, tek GPU için boğuşan iki madenci demektir; ikisi de verim vermez.

---

## Donanım Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---|---|---|
| İşletim sistemi | Ubuntu 20.04+, macOS, Windows 10/11 | Ubuntu 24.04 / 26.04 LTS |
| CPU | 2 çekirdek | 4+ çekirdek |
| RAM | 4 GB | 8 GB+ |
| Disk | 100 GB *(sadece Bölüm B)* | 500 GB+ SSD — SATA olur, HDD olmaz |
| Ağ | 3 Mbps | 10+ Mbps |
| GPU | yok (sadece CPU, çok yavaş) | NVIDIA RTX 20/30/40/50 serisi |

- Hash hızını **GPU ve madenci derlemesi** belirler; disk boyutu ya da CPU önbelleği değil. Büyük önbellekli CPU Poseidon2'yi hızlandırmaz.
- QPoW **VRAM'e aç değil**. 8–12 GB fazlasıyla yeter; 140 GB'lık veri merkezi kartı "50 kat hızlı" değildir ve hash başına maliyette genelde bir 4090'a kaybeder.
- Node veritabanı RocksDB'dir ve rastgele I/O yapar. Her SSD işe yarar; HDD senkronu tıkar.
- **Linux ARM64'te resmî madenci binary'si yok** — Linux x86_64 ya da macOS'tan kaz. AMD GPU'lar CUDA havuz madencisiyle çalışmaz.
- **Bölüm A'da yukarıdaki disk ve bant genişliği payı gerekmez** — sadece GPU ve dışa açık UDP.

---

## Portlar ve Uç Noktalar

| Port | Amacı | Yol | Ne yapmalı |
|---|---|---|---|
| `30333/TCP` | Node P2P | B | İnternete bakabilecek tek port. Opsiyonel — dışa bağlantı senkron için yeter |
| `9833/UDP` | Madenci ↔ kendi node'un (QUIC) | B | **Sadece localhost veya VPN.** `0.0.0.0`'a bağlanır; onu yalnızca güvenlik duvarın korur |
| `9834/UDP` | Havuz madencisi → havuz node'u | A | **Sadece dışa.** Dışa UDP kapalıysa sonsuz yeniden bağlanma döngüsü |
| `9944` | Node RPC | B | Localhost |
| `9615` | Node Prometheus metrikleri | B | Localhost |
| `9900` | Madenci metrikleri / `hive-stats` | A + B | Localhost |

Tüm resmî bağlantılar: [README.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.tr.md).

---

## Nereden Başlamalı

| Durumun | Başlangıç |
|---|---|
| Windows PC, NVIDIA kart, henüz Linux yok | **[ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md)** — aşağıdaki Adım 2'de biter |
| Saatlik GPU kiralıyorsun | **[vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md)** — kendi içinde tam bir Bölüm A varyantı |
| Ubuntu ya da macOS, GPU sürücüsü çalışıyor | Adım 1'e geç |
| Sadece genel bakış ve bağlantılar | [README.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.tr.md) |

---

# Ortak Kurulum

İki yol da burada başlar. İki adım, sonra seçim.

## Adım 1 — Cüzdan Oluştur

Her şeyden önce bir `qz…` adresi gerekir. İki yol da aynı cüzdanı ve aynı 24 kelimeyi kullanır.

1. [Quantus Cüzdanı](https://www.quantus.com/wallet/)'nı kur (iOS / Android — bağlantılar [linktr.ee/quantusnetwork](https://linktr.ee/quantusnetwork)'te de var).
2. Cüzdan oluştur ve **24 kelimeyi çevrimdışı olarak kâğıda yaz**.
3. Ana ekrandaki adres `qz…` ile başlar. Bir havuz formuna ya da sorgu kutusuna yapıştıracağın tek değer bu.

CLI alternatifi:

```bash
# https://github.com/Quantus-Network/quantus-cli/releases adresinden
quantus wallet create --name mining
```

> **KRİTİK:** 24 kelime, ödüllerin için tek kurtarma yolu. Onları asla bir sohbet penceresine, havuz formuna ya da kiralık sunucuya yazma. Bölüm A için **adres**, Bölüm B için o kelimelerden **türetilen inner hash** gerekir.

**Bittiği an:** ekrandan bir `qz…` adresi okuyabiliyorsun ve 24 kelime kâğıtta.

---

## Adım 2 — GPU'yu Doğrula (Linux)

Madenci kurmadan önce NVIDIA'nın tescilli sürücüsünün aktif olduğunu doğrula. Yanlış sürücü yolunda çalışan madenci sessizce 4–6 kat yavaş kalır.

> Henüz Linux yok mu? Önce **[ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md)** — seni buraya geri getirir. Kiralık GPU'da sürücüyü host zaten enjekte eder: **[vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md)**.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq

# yalnızca sürücü yoksa
sudo ubuntu-drivers autoinstall
sudo reboot
```

Yeniden başlattıktan sonra:

```bash
nvidia-smi
nvidia-smi -L      # her GPU için bir satır
```

Kart adını, sürücü sürümünü ve bir `CUDA Version` kolonunu görmelisin. Kullanımın düşük olması normal — henüz madenci çalışmıyor.

> Tam `cuda-toolkit`'e **ihtiyacın yok**; hazır madenci binary'leri kendi CUDA çalışma zamanını taşır. `nvidia-smi` hiçbir şey basmıyorsa Secure Boot'u kapat (ya da MOK anahtarını kaydet) ve yeniden başlat. RTX 50 serisi (Blackwell) güncel bir çekirdek ve 570+ sürücü ister.

**Bittiği an:** `nvidia-smi -L` kartını adıyla listeliyor.

---

## Karar Noktası — Yolunu Seç

Ortak kurulum bitti. Bundan sonrası **ya şu yol ya öbürü**.

| Seçim | Git | Sonunda elde ettiğin |
|---|---|---|
| **Bölüm A — Havuz** | [A1](#a1--havuz-madencisini-kur) | Havuza hash basan tek bir `quanpool-miner` servisi, pay başına artan bakiye |
| **Bölüm B — Kendi node** | [B1](#b1--otomatik-kurulum-betigi) | Senkron bir `quantus-node` ve yerel bir `quantus-miner`, tam blok ödülleri |

Hâlâ emin değilsen **Bölüm A** yap. Bir saatte geri alınabilir, kartının ve adresinin çalıştığını kanıtlar ve sen düşünürken ödemeye başlar.

---

# Bölüm A — Havuz Madenciliği (Quanpool PPLNS)

> **Ön koşullar:** Adım 1 (bir `qz…` adresi) ve Adım 2 (çalışan GPU sürücüsü).
> **Bitiş:** [Bölüm A Bitiş Çizgisi](#bölüm-a-bitiş-çizgisi).
> **Dokunmayacakların:** `quantus-node`, zincir senkronu, gelen güvenlik duvarı kuralları, inner hash.

> Quanpool bir **topluluk** havuzu, resmî Quantus altyapısı değil. Host adresi, indirme bağlantısı ve TLS pini [quanpool.com](https://quanpool.com/) → **Start mining** sayfasında canlı yayınlanır. Her seferinde oradan kopyala; aşağıdaki değerler bilinçli olarak yer tutucudur.

### A1 — Havuz Madencisini Kur

```bash
sudo mkdir -p /opt/quantus
sudo chown "$USER:$USER" /opt/quantus
cd /opt/quantus

wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version
```

Bu sürüm 404 verirse güncel Linux bağlantısını **Start mining**'den al (6.1.0 da çalışır).

> `gpu-list` alt komutu 6.1+ ile kaldırıldı. Kartları `nvidia-smi -L` ile listele.

### A2 — Kartı Benchmark Et

Benchmark yereldir, havuzla hiç konuşmaz. Havuz yapılandırmasından **önce** yap — bu rehberin en pahalı sorusunu cevaplar: gerçekten CUDA yolunda mısın?

```bash
cd /opt/quantus
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
```

| Sonuç | Anlamı |
|---|---|
| Karta göre 400 MH/s – 1,5 GH/s | Doğru — Linux CUDA yolu |
| Modern bir RTX kartında ~100 MH/s | **Yanlış binary ya da sürücü** — stock/wgpu yolu, 4–6 kat yavaş. Bunu sonra değil şimdi çöz |

Masaüstünde `--cpu-workers 0` zorunlu kabul et. CPU madenciliği iş parçacığı başına ~15 MH/s ekler — kartın yanında ihmal edilebilir, kartı besleyen çekirdekleri ise çalar.

### A3 — Token ve TLS Pini

[quanpool.com](https://quanpool.com/) → **Start mining** sayfasında doldur:

| Alan | Değer |
|---|---|
| Address | `qz…` adresin — **asla seed** |
| Worker | opsiyonel isim, **makine başına tekil**. `a-z 0-9 . - _`, en fazla 32 karakter, boşluk yok |
| Mode | **Pool (PPLNS)** |
| System | **Linux** |

Kayıt ve şifre yok: adres zaten hesaptır. `--node-addr` (hostname yerine düz `IP:9834` tercih et) ve 64 haneli hex `--tls-cert-sha256` değerini dosyalara yaz, böylece shell geçmişine düşmesinler:

```bash
cd /opt/quantus

# tek satır: qzADRESIN.workeradı
nano auth-token

# Start mining'deki 64 haneli TLS pini
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

### A4 — İlk Manuel Çalıştırma

systemd'ye devretmeden önce bir kez ön planda çalıştır ki hataları okuyabilesin.

```bash
cd /opt/quantus
./quanpool-miner serve \
  --node-addr <HAVUZ_HOST>:9834 \
  --auth-token-file /opt/quantus/auth-token \
  --tls-cert-sha256-file /opt/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
```

`<HAVUZ_HOST>` yerine **Start mining**'deki adresi yaz, `--gpu-devices` değerini `nvidia-smi -L` çıktısındaki kart sayısına ayarla (hepsini kullanmak için parametreyi at).

İlk dakikada sağlıklı belirtiler:

- `nvidia-smi` madenci sürecini ve %95–100 GPU kullanımını gösteriyor
- log iş satırları ve `SHARE FOUND` basıyor
- `curl -s http://127.0.0.1:9900/hive-stats` dolu bir `hs` ve sıfır ret içeren bir `ar` döndürüyor

> İlk saniyelerde bir grup `SOLUTION LOST` / stale mesajı, madenci güncel işe yetişirken normaldir.

Sağlıklı göründüğünde `Ctrl+C` ile durdur — sonraki adım onu yeniden başlatmalara dayanıklı yapar.

### A5 — systemd Servisi

```bash
sudo tee /etc/systemd/system/quanpool-miner.service >/dev/null <<EOF
[Unit]
Description=Quantus havuz madencisi (Quanpool PPLNS)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/opt/quantus
ExecStart=/opt/quantus/quanpool-miner serve --node-addr <HAVUZ_HOST>:9834 --auth-token-file /opt/quantus/auth-token --tls-cert-sha256-file /opt/quantus/tls-cert-sha256 --cpu-workers 0 --gpu-devices 1 --mode pool
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

Başlatmadan önce `ExecStart` içindeki `<HAVUZ_HOST>`'u düzelt. `auth-token` mod `600` olduğu için buradaki `User=` tarafından okunabilir olmalı.

Masaüstünde ayrıca uyku kapatılmalı, yoksa ekran karardığında GPU durur:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

> Kiralık konteynerlerde genelde çalışan bir `systemd` yoktur. Onun yerine imajın süreç yöneticisini kullan — bkz. [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md).

### A6 — Havuzda Doğrula

Önce yerelde:

```bash
curl -s http://127.0.0.1:9900/hive-stats
```

`hs` genelde **kH/s**'dir — `430000` ≈ 430 MH/s. `ar` değeri `[kabul, ret]` olup ret sıfırda kalmalı; `temp`, `fan` ve `pool_rtt_ms` de raporlanır.

Sonra [quanpool.com](https://quanpool.com/)'u aç, `qz…` adresini sorgu kutusuna yapıştır ve **Look up**'a bas. Worker'ın birkaç dakika içinde kendi hash hızı, pay sayısı ve artan bekleyen bakiyesiyle görünür.

> `9900`, `9833`, `9944` ve `9615` portları asla internete açılmamalı. Modümde bunlar için "virtual server" kuralı hiçbir şey kazandırmaz, gerçek risk yaratır.

### Bölüm A Bitiş Çizgisi

**Dördü de doğruysa tamamdır:**

1. `systemctl status quanpool-miner` → `active (running)` ve yeniden başlatmadan sonra da ayakta.
2. `nvidia-smi` A2'deki benchmark hızında %95–100 kullanım gösteriyor.
3. `hive-stats` kabul edilen payları artıyor, ret sıfırda.
4. Worker'ın havuz sorgu sayfasında görünüyor ve bekleyen bakiye artıyor.

Başka kurulacak bir şey yok: node yok, senkron yok, inner hash yok. [İzleme](#izleme-ve-günlük-komutlar), [Çoklu Makine](#çoklu-makine-çalıştırmak) ya da [Ekonomi](#ekonomi--ne-beklemeli) bölümlerine geç. Sonradan kendi node'unu istemedikçe **Bölüm B'yi atla** — isteyince önce [Sonradan Yol Değiştirmek](#sonradan-yol-değiştirmek)'i oku.

---

# Bölüm B — Kendi Node'un

> **Ön koşullar:** Adım 1 (24 kelime — inner hash'i onlardan türetiyorsun), Adım 2 (çalışan GPU sürücüsü), 100 GB+ SSD ve sürekli açık bırakabileceğin bant genişliği.
> **Bitiş:** [Bölüm B Bitiş Çizgisi](#bölüm-b-bitiş-çizgisi).
> **İki yol var:** B1 resmî betiktir, her şeyi senin için yapar. B2–B5 aynı işin manuel karşılığı. **Ya B1 ya B2–B5 — ikisi birden değil.** B6 her iki durumda geçerli.

Komisyon yok ve tam kontrol var; karşılığında tam zincir senkronu ve eşleşmiş binary çifti gerekiyor. Linux, macOS ve WSL2'de çalışır.

### B1 — Otomatik Kurulum Betigi

Resmî betik wormhole inner hash'i ve node kimliğini üretir, eşleşmiş bir çifti `~/quantus-mining/bin/` altına indirir ve `CHAIN=mainnet` ile `~/quantus-mining/mining.conf` dosyasını yazar.

```bash
curl -fsSL https://docs.quantus.com/scripts/quantus-mining.sh -o quantus-mining.sh
chmod +x quantus-mining.sh
./quantus-mining.sh setup
./quantus-mining.sh start -d
```

Günlük kullanım:

```bash
./quantus-mining.sh start          # node ön planda, madenci arka planda
./quantus-mining.sh start-node     # birinci terminal
./quantus-mining.sh start-miner    # ikinci terminal, madenci sunucusu dinlemeye başladıktan sonra
./quantus-mining.sh stop
./quantus-mining.sh config show
./quantus-mining.sh config set GPU_DEVICES 1
./quantus-mining.sh config set CPU_WORKERS 0
```

İki `latest` etiketinin birbirinden ayrışmasını beklemek yerine sürümleri sabitle:

```bash
./quantus-mining.sh config set NODE_VERSION v1.0.1
./quantus-mining.sh config set MINER_VERSION v4.2.0
./quantus-mining.sh stop
./quantus-mining.sh setup --force
./quantus-mining.sh start -d
```

`--force` yalnızca binary'leri yeniler: `INNER_HASH` ve wormhole adresini korur, **yeni anahtar çifti üretmez**. Betik node'un `miner-auth-token` ve `miner-tls-cert-sha256` dosyalarını kendisi okur, yani onları elle kopyalaman gerekmez. Docker modu kaldırıldı.

> **Betik işini çözdüyse burada dur** ve [Bölüm B Bitiş Çizgisi](#bölüm-b-bitiş-çizgisi)'ne atla. B2–B5 aynı sonucu elle kurar — dizinler, sürümler ve servis yönetimi üzerinde kontrol istediğinde ya da betik dağıtımında çalışmadığında işe yarar.

### B2 — Manuel Kurulum: Binary'ler

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

Güncel etiketler için [Releases](https://github.com/Quantus-Network/chain/releases) sayfasına bak — resmî öneri **node v1.0.1 veya üstü** ve iki depo birbirinden bağımsız sürümleniyor.

Devam etmeden önce çiftin kimlik doğrulamalı protokolü konuştuğunu teyit et — iki komut da eşleşen bir satır basmalı:

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

### B3 — Node Kimliği ve Wormhole Inner Hash

```bash
./quantus-node key generate-node-key --file node_key.p2p
./quantus-node key quantus --scheme wormhole --words
```

`--words`, 24 kelimeyi **ekrana yazmadan** sorar; böylece ifade shell geçmişine düşmez. Bastığı değerleri sakla:

| Değer | Nedir | Ne yapmalı |
|---|---|---|
| **Address** | wormhole adresin — ödüller buraya düşer | izleme için sakla |
| **Inner Hash** | 32 bytelık öngörüntü | `--rewards-inner-hash` olarak ver |
| **Secret** | sahipliği kanıtlayan anahtar | çevrimdışı yedekle, asla paylaşma |

Cüzdan uygulamandaki aynı 24 kelimeyi kullanmak önerilen yoldur — ödüller uygulamada otomatik görünür. Bunun yerine yeni bir cüzdana kazmak istersen `./quantus-node key quantus --scheme wormhole` çalıştır ve ürettiği ifadeyi yedekle. Ödüllerin wormhole adresine yönlenmesi **opsiyonel değil**; protokolün içinde tanımlı ve aynı sebeple madencilik kimliğin ile ödeme adresin zincir üstünde bağlantılı görünmez.

### B4 — Node'u Başlat

```bash
screen -S quantus-node
cd ~/quantus
./quantus-node \
  --name <NODE_ADIN> \
  --validator \
  --miner-listen-port 9833 \
  --chain mainnet \
  --node-key-file node_key.p2p \
  --rewards-inner-hash <INNER_HASH_DEGERIN> \
  --max-blocks-per-request 64 \
  --sync full
```

`--name`, node'unun [telemetri](https://telemetry.quantus.cat/) sayfasında görünen adıdır.

`--miner-listen-port` ile ilk başlatışta node, madenci kimlik doğrulama dosyalarını zincir dizinine yazar:

| Dosya | Amacı |
|---|---|
| `miner-auth-token` | madencinin `Ready` mesajında gönderdiği ortak sır. Mod `0600`, asla loglanmaz |
| `miner-tls-cert-sha256` | madenci QUIC sertifikasının SHA-256'sı; madenciler bunu pinler |
| `miner-tls-cert.der` / `miner-tls-key.der` | node TLS materyali — özel anahtarı asla madenciye kopyalama |

| Platform | Zincir dizini |
|---|---|
| Linux | `~/.local/share/quantus-node/chains/mainnet/` |
| macOS | `~/Library/Application Support/quantus-node/chains/mainnet/` |

> ⚠️ **Ödül beklemeden önce tam senkronu bekle.** Tepeye ulaşmadan kazılan bloklar orphan olur ve hiçbir şey kazandırmaz; node'un eşi yoksa madenci kendiliğinden bekler. Log `Syncing`'den güncel yükseklikte `Idle`'a geçtiğinde senkronsun — genelde 15 dakika ile birkaç saat. Senkron sırasındaki `discarding proposal` normaldir. `Verification failed` ve 0 eş ile tıkanma, node sürümünün ağla uyuşmadığı anlamına gelir — [Releases](https://github.com/Quantus-Network/chain/releases)'e bak.

Ayrıca madenci sunucusunun dinlemeye başladığını bildiren log satırını bekle. Madenci sunucusu başlayamazsa node çıkar — yerel madenciliğe geri düşme yok.

### B5 — Harici Madenciyi Başlat

İkinci terminalde:

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

Varyantlar: Vulkan'ın olmadığı NVIDIA kartlarında `--cuda-gpu` ekle; kullanılabilir GPU'su olmayan makinede `--cpu-workers 4 --gpu-devices 0` kullan; ısı ve güç tüketimini sınırlamak için `--gpu-throttle-ms 50` (masaüstünü de kullanıyorsan `5`) ekle.

> Her zaman satır içi değerler yerine `--auth-token-file` / `--tls-cert-sha256-file` kullan. Yanlış token ya da pin **kalıcı** hatadır: madenci yeniden bağlanma döngüsüne girmez, durur. Dosyaları tekrar oku.
>
> macOS'ta `CHAIN_DIR`'i tırnak içinde tut — yolda boşluk var.

### B6 — Node/Madenci Çiftini Güncelleme

Betikle ve elle kurulumda aynı:

1. Önce node'u, sonra madenciyi durdur.
2. **Eşleşmiş** bir çift indir — ikisi de `quantus-miner/2` konuşmalı.
3. `.../chains/mainnet/` dizinini koru. Asla bir `chains/planck/` dizinini içe alma.
4. Node'u başlat, madenci sunucusunun dinlemesini bekle, sonra madenciyi başlat.

### Bölüm B Bitiş Çizgisi

**Beşi de doğruysa tamamdır:**

1. Node logu güncel zincir yüksekliğinde `Idle` ve eş sayısı stabil.
2. `--name` değerin [telemetry.quantus.cat](https://telemetry.quantus.cat/)'ta görünüyor.
3. Node, madenci sunucusunun `9833`'te dinlediğini logladı.
4. Madenci bağlı, `nvidia-smi` %95–100 kullanım gösteriyor ve node logunda `Broadcasting job` var.
5. `ufw status` yalnızca `22/tcp` ve `30333/tcp` izin veriyor, **başka hiçbir şey yok**.

Artık ödüller tam blok halinde, düzensiz aralıklarla, B3'teki wormhole adresine gelir — cüzdan uygulamasından ya da explorer'dan kontrol et. [İzleme](#izleme-ve-günlük-komutlar) ve ardından [Ekonomi](#ekonomi--ne-beklemeli) bölümüne geç ki bloklar arasındaki sessizlik seni endışelendirmesin.

---

# Referans

Bitirdiğin yol için geçerlidir.

## İzleme ve Günlük Komutlar

**Bölüm A — havuz**

```bash
sudo systemctl status quanpool-miner --no-pager
sudo systemctl restart quanpool-miner
journalctl -u quanpool-miner -f
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw,fan.speed --format=csv

# yeniden benchmark ya da dışa UDP testi (nc'den yanıt gelmemesi tek başına arıza kanıtı değil)
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
nc -zvu <HAVUZ_HOST> 9834
```

Havuzun sorgu sayfası worker'ları, bekleyen bakiyeyi ve ödemeleri gösterir. Binary güncellemek için: servisi durdur, yeni dosyayı indir, `chmod +x`, tekrar başlat.

**Bölüm B — kendi node**

| Ne | Nerede |
|---|---|
| Ödüller | cüzdan uygulaması (aynı 24 kelime) ya da `https://explorer.quantus.com/accounts/<QZ_ADRESIN>` |
| Node telemetride | [telemetry.quantus.cat](https://telemetry.quantus.cat/) — `--name` değerini ara |
| Node metrikleri / RPC | `http://localhost:9615/metrics` · `http://localhost:9944` |
| Madenci metrikleri | `http://localhost:9900/metrics` |

```bash
./quantus-node --version
./quantus-node key inspect-node-key --file node_key.p2p
tail -f ~/.local/share/quantus-node/chains/mainnet/network/quantus-node.log
```

Sağlıklı bir node stabil eş sayısı, tepede `Idle`, `Prepared block for proposing` ve `Broadcasting job` gösterir.

---

## Performans Referansı

Linux CUDA havuz madencisi ile stock madencinin yayınlanmış değerleri, yalnızca GPU. Kendi kartını her zaman `benchmark --duration 30` ile doğrula.

| Kart | Stock madenci | Linux CUDA havuz madencisi | Oran |
|---|---|---|---|
| RTX 5090 | 342 MH/s | 1,49 GH/s | ~4,4× |
| RTX 4090 | 179 MH/s | 1,12 GH/s | ~6,3× |
| RTX 4070 Ti | 122 MH/s | 574 MH/s | ~4,7× |
| RTX 4070 | — | ~430–460 MH/s | — |
| RTX 5070 | ~100 MH/s | ~410–450 MH/s | ~4,2× |
| RTX 3080 Ti | 104 MH/s | 435 MH/s | ~4,2× |
| RTX 5060 Ti | 75 MH/s | 314 MH/s | ~4,2× |
| CPU, 8 worker | toplam ~120 MH/s | — | iş parçacığı başına ~15 MH/s |

Komisyon ve şans hariç, kaba beklenen gelir:

```text
günlük QTC ≈ (senin H/s / ağ H/s) × (86400 / blok_saniyesi) × blok_ödülü
```

Her terim değişken. Zorluk her finalize blokta yeniden ayarlandığı için tek bir tüketici kartı, TH/s ile ölçülen bir ağın küçük bir kesridir. Her türlü fiat rakamını doğrulanmamış kabul et.

**Sıcaklık:** 7/24 kazan kart güç limitine yakın, fanları yüksek çalışır. Sürekli ~70 °C normaldir; throttle eşiği ~83–88 °C civarı. Gerçek bakım işi toz ve kasa hava akışı. Kart sürekli 85 °C'de duruyorsa `--gpu-throttle-ms 5` ekle.

---

## Güvenlik Duvarı

**Bölüm A'da gelen kural gerekmez** — yalnızca havuz portuna dışa UDP. Modümun ya da ISS'in dışa UDP/QUIC'i engelliyorsa madenci yeniden bağlanma döngüsüne girer; telefon hotspot'undan test ederek doğrula.

**Bölüm B:**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 30333/tcp
sudo ufw enable
sudo ufw status verbose
```

`9833`, `9944` ya da `9615`'i **açma**. Uzaktaki bir madenciyi node'a WireGuard veya Tailscale ile bağla, madenci portunu özel tut.

CGNAT arkasında (mobil bağlantıların çoğu ve pek çok ev fiberi) gelen `30333` hiç ulaşmaz — kural zararsızdır ama işe yaramaz; node dışa bağlantılarla senkron olmaya devam eder. Port yönlendirme CGNAT'ı çözmez; bunu ancak genel IP ya da VPN uç noktası çözer. Bölüm A'nın evde daha kolay yol olmasının sebebi tam olarak bu.

---

## Çoklu Makine Çalıştırmak

Birden fazla madenci **aynı** `qz…` adresine ödeme yapabilir; PPLNS payları toplanır.

- Her makineye **farklı bir worker adı** ver. Aynı adı tekrar kullanmak iki rig'i çarpıştırır ve biri havuzdan kaybolur.
- İkinci bir makine eklemek için evdeki rig'i durdurmana gerek yok.
- Seed'ini asla kiralık makineye kopyalama. Kiralık bir kutuya yalnızca `qzADRES.worker` ve TLS pini gerekir — bkz. [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md).
- Bölüm B tarafındaki karşılığı, VPN üzerinden tek node'un `9833` portuna bağlanan birden fazla madencidir. Node işleri hepsine yayar, ilk geçerli sonuç kazanır.

---

## Sorun Giderme

### Bölüm A — havuz yolu

| Belirti | Ne kontrol edilir |
|---|---|
| `timed out` / sonsuz yeniden bağlanma | Havuz portuna dışa **UDP**. Ev güvenlik duvarları ve bazı ISS hatları QUIC'i engeller — telefon hotspot'undan test et |
| `certificate` / `fingerprint` hatası | TLS pinini **Start mining**'den yeniden kopyala; tam 64 hex karakter |
| Worker sitede hiç görünmüyor | `auth-token` içindeki adres sorguladığından farklı ya da worker adı kurallara uymuyor. 2–3 dakika tanı |
| Modern kartta ~100 MH/s benchmark | CUDA derlemesi değil — yanlış binary ya da sürücü. `nvidia-smi`'ye bak |
| Hâlâ `127.0.0.1:9833`'e bakıyor | O adres Bölüm B'nin; havuz madenciliği havuz host'unun `9834`'ünü kullanır |
| Servis `running` ama GPU %0 | Logu oku: token, pin ya da `--node-addr` yanlış |
| Rig ekleyince evdeki worker kayboldu | Worker adı çakışması. Birini yeniden adlandır ve ikisini de yeniden başlat |
| Makine boşa çıkınca madencilik duruyor | Uyku hâlâ açık — A5'teki `gsettings` komutlarını tekrar uygula |

### Bölüm B — kendi node yolu

| Belirti | Ne kontrol edilir |
|---|---|
| Madenci hemen çıkıyor | Kimlik doğrulama ya da sürüm: madenci sunucusunun dinlemesini bekledin mi, iki auth dosyası var mı, iki binary de `quantus-miner/2` konuşuyor mu |
| TLS `no application protocol` | Node ve madenci uyumsuz bir çift |
| Yanlış token ya da pin | Tasarım gereği kalıcı hata — dosyaları yeniden oku, körü körüne tekrar deneme |
| `Verification failed`, 0 eş | Node sürümü ağla uyuşmuyor |
| Senkron hiç bitmiyor | Bant genişliği 3 Mbps altında ya da SSD yerine HDD |
| Windows'ta senkron tıkıyor | Defender RocksDB dizinini tarıyor — istisna ekle ya da WSL2 kullan |
| Kazan ama ödül gelmeyen | Hâlâ senkron (orphan), yanlış inner hash ya da cüzdanından farklı bir seed |
| Node madenci portunda çıkıyor | Bind, TLS ya da token hatası. Yerel madenciliğe geri düşme yok |
| Linux ARM64'te madenci yok | x86_64 ya da macOS'tan kaz |

Windows'ta senkrondan önce node veri dizinini Defender'dan muaf tut:

```powershell
Add-MpPreference -ExclusionPath "$env:USERPROFILE\.quantus"
```

> Kimlik doğrulamasız eski sürümlerde (node v0.9.0, madenci v3.3.1 ve öncesi) madenci kimlik doğrulaması yoktur. Güncel mainnet'e karşı kullanma.

---

## Ekonomi — Ne Beklemeli

**Emisyon.** Blok ödülleri `(MaxSupply − CurrentSupply) / EmissionDivisor` formülünü izler — sabit 21.000.000 arzın yumuşak eksponansiyel azalması, halving uçurumu yok. Madenciler zamanla **toplam arzın %50'sini** alır ve arzın yaklaşık %99'u şu 40 yıl içinde çıkar. Blok ödüllerinin %15'i şirkete geliştirme vergisi olarak gider ve yıllar boyunca hak edilir.

**Zincir üstü ücretler.** Standart transferler sabit bir ücret öder ve bu madenciye gider. Yüksek güvenlikli geri alınabilir transferler hacme göre ücret öder ve bu yakılır; ZK ile birleştirilmiş işlemler daha küçük bir hacim ücreti öder ve bu madenci ile yakım arasında bölünür. Yani madencilik geliri ezici çoğunlukla emisyondan gelir, ücretlerden değil.

**Bölüm A — PPLNS nasıl öder.** **Blok başına değil, pay başına** ödersin. Hiçbir blok "senin" olmaz; bakiyen sürekli artar ve zaten amacı bu — varyansı kaldırır. Havuz sabit bir komisyon alır ve tek bir eşiğin üstünde ödeme yapar; ikisi de havuz sitesinde yayınlıdır ve zamanla değişmiştir, bu yüzden bu rehber dahil hiçbir dokümandaki rakama güvenme, oradan oku. Havuz içindeki solo modu aynı komisyonu alır ama piyango gibi öder.

**Bölüm B — solo nasıl öder.** Komisyon yok, tam kontrol var ve kazandığında tam bir blok ödülü. Tek bir tüketici kartıyla TH/s'lik bir ağa karşı bu, uzun sessizlikler demek olabilir. Bir şey bozuk değil; varyans, paylaşmamanın bedeli.

**Gelirini ne oynatır.** Zorluk her finalize blokta yeniden ayarlanır; ağ hash hızı yükselirse rig'in hiç değişmese de günlük QTC'n düşer. `benchmark` ile ölç, sonra havuzun canlı ağ hash hızıyla karşılaştır.

> QTC piyasa verisi sığ ve birim resmî dokümanların ve araçların bir kısmında `QUAN` diye etiketli. Her fiat projeksiyonunu spekülatif kabul et.

---

## Sonradan Yol Değiştirmek

İki yol cüzdanı, adresi ve GPU işini paylaştığı için arasında geçiş ucuzdur.

**A → B (havuzdan kendi node'una)**

```bash
sudo systemctl disable --now quanpool-miner
```

Sonra [B1](#b1--otomatik-kurulum-betigi)'den başla. Inner hash'i **aynı** 24 kelimeden türet, ödüller aynı cüzdan uygulamasına düşmeye devam eder. Havuzdaki bekleyen bakiyeye dokunma — normal takviminde ödenir.

**B → A (kendi node'undan havuza)**

Önce madenciyi, sonra node'u durdur. Geri dönme ihtimalin varsa `chains/mainnet/` dizinini sakla; yeniden senkrondan kurtarır. Sonra aynı `qz…` adresiyle [A1](#a1--havuz-madencisini-kur)'den başla.

**İkisini birden çalıştırmak strateji değil.** Tek GPU'da iki madenci birbirini yarıya düşürür. İki kartın varsa `--gpu-devices` ve ayrı servislerle her yola bir kart ver — aksi halde birini seç.

---

## Sonraki Adımlar

| Hedef | Doküman |
|---|---|
| Windows PC'yi madencilik makinesine çevirmek | [ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md) |
| Saatlik GPU kiralamak | [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md) |
| Ağ bilgileri, bağlantılar ve yol karşılaştırması | [README.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.tr.md) |

---

## Yazar Hakkında

**HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
