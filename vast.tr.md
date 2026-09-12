<div align="center">

# ☁️ Kiralık GPU'da Quantus Madenciliği

**Saatlik GPU kirasıyla QTC kaz — donanım yok, zincir senkronu yok, makinede seed yok**
*Makine seçimi, SSH, CUDA madencisi, supervisor ile otomatik başlatma, maliyet kontrolü, kapatış.*

[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Pool](https://img.shields.io/badge/Mod-Havuz%20PPLNS-orange?style=flat-square)](https://quanpool.com/)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Yazar:** HazenNetworkSolutions
> **Yol:** yalnızca havuz madenciliği — [guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md) Bölüm A'nın kiralık GPU varyantı
> **Sürümler:** havuz madencisi 6.2.0
> **Son Güncelleme:** Eylül 2026

---

## Bu Doküman Ne

Saatlik GPU kiralamak (Vast.ai ve benzeri pazar yerleri) donanım sahibi olmadan kazmayı mümkün kılar. NVIDIA sürücüsü zaten enjekte edilmiş bir konteyner alırsın, tek bir binary kurarsın ve çalıştığı süre boyunca saat başı ödersin.

**Yalnızca havuz madenciliği.** Kiralık bir makinede kendi node'unu çalıştırmak, makine yok edildiğinde kaybolacak 100 GB'lık zincir verisini senkronlamak için saat başı ödeme demektir. Kendi node'unu istiyorsan elinde kalan donanım kullan — [guide.tr.md Bölüm B](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#bölüm-b--kendi-nodeun).

**Kiralık makineler için üç kural:**

1. 24 kelimen makineye **asla** gitmez. Madenciye yalnızca herkese açık `qz…` adresin gerekir.
2. Madenci çalışsa da çalışmasa da kira işler. Önce benchmark, sonra karar.
3. Faturayı yalnızca **Destroy** durdurur. Durdurulan makine, saklı veri için para yakmaya devam eder.

**Ön koşullar:** bir `qz…` adresi ([guide.tr.md Adım 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-1--cüzdan-oluştur)) ve [quanpool.com](https://quanpool.com/) → **Start mining** sayfasındaki host adresi ile TLS pini.

---

## Adım 1 — SSH Anahtarı Oluştur

Kendi makinende:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vast_quantus -C "quantus-mining"
cat ~/.ssh/vast_quantus.pub
```

Genel anahtarı pazar yerinin SSH anahtarları sayfasına yapıştır. Özel anahtarı hiçbir yere yükleme.

---

## Adım 2 — Makine Seç

| Ölçüt | Ne aranmalı |
|---|---|
| GPU | Hız için RTX 4090 / 5090; hash başına maliyet için 3080 Ti / 4070 |
| Güvenilirlik | %99+ — düşük güvenilirlikli host'lar işin ortasında kaybolur |
| İmaj | sürücüsü kurulu herhangi bir CUDA / PyTorch imajı |
| Disk | 20 GB fazlasıyla yeter — zincir saklamayacaksın |
| İnternet | dışa UDP çalışmalı; ağır filtreli host'lardan kaçın |
| Fiyat | $/saat'i kartın adıyla değil benchmark'ıyla kıyasla |

80–140 GB VRAM'li veri merkezi kartı oranında hızlı **değil**: QPoW VRAM'i neredeyse kullanmaz, yani tüketici bir 4090 hash başına maliyette genelde öne geçer.

Kesintili (spot) makineler daha ucuzdur ama her an duraklatılabilir. Madencilik için bu kabul edilebilir — supervisor ile otomatik başlatma varsa makine döndüğünde iş de döner.

---

## Adım 3 — Bağlan

```bash
ssh -i ~/.ssh/vast_quantus -p <PORT> root@<HOST>
nvidia-smi
nvidia-smi -L
```

`nvidia-smi` çalışmıyorsa host bozuk — makineyi yok et ve başka bir tane kirala; başkasının sürücü yığınını saat başı ödeyerek ayıklama.

> Portalın çıkış/temizlik yardımcı betiklerini kabuğunda `source` etme. Bazı imajlarda oturumunu kapatırlar.

---

## Adım 4 — Madenciyi Kur

İmaj sağlıyorsa `/workspace` kullan — kalıcı birim orası.

```bash
mkdir -p /workspace/quantus && cd /workspace/quantus
apt-get update -qq && apt-get install -y -qq wget curl ca-certificates netcat-openbsd jq

wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version
```

Bu sürüm kalkmışsa güncel indirme bağlantısını **Start mining**'den al.

---

## Adım 5 — Karar Vermeden Önce Benchmark

Sıranın böyle olmasının tüm sebebi bu: benchmark kiranın saatlik fiyatına değip değmediğini söyler ve havuzla hiç konuşmaz.

```bash
cd /workspace/quantus
./quanpool-miner benchmark --gpu-devices 1 --cpu-workers 0 --duration 30
```

| Sonuç | Ne yapmalı |
|---|---|
| Karta göre 400 MH/s – 1,5 GH/s | Devam et |
| Modern bir RTX kartında ~100 MH/s | Bu host'ta sürücü yolu yanlış — yok et, başka yerden kirala |
| Kartın yayınlı değerinin çok altı | GPU paylaşımlı ya da kısıtlı — yok et |

---

## Adım 6 — Kimlik Bilgileri

[quanpool.com](https://quanpool.com/) → **Start mining**'den, `qz…` adresin ve bu makineye özgü bir worker adıyla:

```bash
cd /workspace/quantus

# tek satır: qzADRESIN.vast01
nano auth-token

# 64 haneli hex TLS pini
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

> Seed ifadeni asla yapıştırma. Adres herkese açık bir bilgidir ve havuzun sana ödeme yapması için yeterlidir. Her kiralık makineye kendi worker adını ver, aksi halde iki rig çakışır ve biri havuzdan kaybolur.

---

## Adım 7 — İlk Manuel Çalıştırma

```bash
cd /workspace/quantus
./quanpool-miner serve \
  --node-addr <HAVUZ_HOST>:9834 \
  --auth-token-file /workspace/quantus/auth-token \
  --tls-cert-sha256-file /workspace/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
```

Bir dakika içinde sağlıklı görüntü: %95–100 GPU kullanımı, logda `SHARE FOUND` satırları ve `hive-stats` çıktısında sıfır retle birlikte bir hash hızı. Açılışta bir grup stale iş mesajı normaldir.

`timed out` döngüsüne giriyorsa bu host'ta dışa UDP kapalı — yok et ve başkasını kirala. Sağlıklı göründüğünde `Ctrl+C`.

---

## Adım 8 — supervisor ile Otomatik Başlatma

Konteynerlerde genelde çalışan bir `systemd` yoktur, yani guide.tr.md'deki systemd birimi burada geçerli değil. Pazar yeri imajlarının çoğu bunun yerine **supervisor** ile gelir.

```bash
mkdir -p /opt/supervisor-scripts

cat > /opt/supervisor-scripts/quanpool-miner.sh <<'EOF'
#!/usr/bin/env bash
cd /workspace/quantus || exit 1
exec ./quanpool-miner serve \
  --node-addr <HAVUZ_HOST>:9834 \
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

Başlatmadan önce betikteki `<HAVUZ_HOST>`'u düzelt. `autorestart=true`, çökmeden ya da duraklatılmış spot makine geri döndükten sonra madenciyi tekrar ayağa kaldırır.

```bash
supervisorctl restart quanpool-miner
supervisorctl stop quanpool-miner
tail -f /var/log/portal/quanpool-miner.log
```

> İmajda supervisor yok mu? Madenciyi `tmux` ya da `screen` içinde çalıştır ve konteyner yeniden başladığında elle başlatman gerekeceğini kabul et.

---

## Adım 9 — Doğrula

```bash
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw --format=csv
```

`hs` genelde kH/s'dir (`430000` ≈ 430 MH/s) ve `ar` değeri `[kabul, ret]`; ret sıfırda kalmalı. Sonra `qz…` adresini [quanpool.com](https://quanpool.com/)'da sorgula — worker birkaç dakika içinde görünür.

---

## Adım 10 — Maliyet Kontrolü

Kira, hash hızından bağımsız olarak saat başı işler; hesap basit: **saatlik kira, o hash hızının kazandırdığı QTC'ye karşı.** Bunu ilk haftada değil ilk saatte kontrol et.

- Adım 5'teki ölçülen hızı not et ve havuzun canlı ağ hash hızıyla karşılaştır.
- Zorluk her finalize blokta yeniden ayarlanır; dünün tahmini bugün yanlış olabilir.
- Pazar yeri hesabında harcama limiti ya da kredi uyarısı kur.
- Kesintili makineler daha ucuz; otomatik başlatmayla madenciliğe iyi uyar.
- Makine durdurulmuşken bile depolama faturalanır.

Kiralık GPU madenciliği yalnızca kart ucuz ve ağ hash hızı düşükken kârlıdır. Her projeksiyonu doğrulanmamış kabul et ve haftalık gözden geçir.

---

## Adım 11 — Kapatış

1. `supervisorctl stop quanpool-miner`
2. Bekleyen bakiyeni havuz sorgu sayfasından doğrula — bakiye makinede değil adresinde durur.
3. Pazar yeri arayüzünden makineyi **Destroy** et. Durdurmak yetmez; faturayı yalnızca Destroy bitirir.
4. Tamamen bitirdiysen SSH anahtarını pazar yeri hesabından kaldır.

Havuzdaki ödenmemiş bakiye makineyi yok etmekten etkilenmez: `qz…` adresine bağlıdır ve havuzun normal takviminde ödenir.

---

## Güvenlik Kontrol Listesi

| Kural | Neden |
|---|---|
| Seed ifadesi makineye hiç girmez | Host operatörü dosya sistemini okuyabilir |
| Makinede yalnızca `qz…` adresi ve TLS pini durur | İkisi de açık bilgi ya da makineye özgü |
| Sırlar `chmod 600` dosyalarda, satır içinde asla | Shell geçmişi ve süreç listesi okunabilir |
| Gelen port açılmaz | Havuz madenciliği yalnızca dışa çalışır |
| `9900` localhost'ta kalır | Madenci metrikleri internet için değil |
| Makine başına tek worker adı | Aynı ad rig'leri çakıştırır |
| İş bitince Destroy | Durdurulmuş makine depolama için ödemeye devam eder |

---

## Sorun Giderme

| Belirti | Çözüm |
|---|---|
| `timed out` / sonsuz yeniden bağlanma | Host'ta dışa UDP kapalı — yok et, başkasını kirala |
| `certificate` / `fingerprint` hatası | 64 haneli TLS pinini **Start mining**'den yeniden kopyala |
| Modern kartta ~100 MH/s benchmark | Host'un sürücü yolu yanlış — ayıklamak için para ödeme |
| `nvidia-smi` yok ya da cihaz görmüyor | Bozuk host imajı — yok et |
| Worker havuz sayfasında yok | `auth-token` içindeki adres ya da worker adı yanlış; 2–3 dakika tanı |
| Yeniden başlatmadan sonra madenci yok | supervisor yapılandırması eksik ya da `autostart=false` |
| `systemctl` bulunamadı | Konteynerde beklenen durum — supervisor kullan |
| Oturum beklenmedik şekilde kapanıyor | Portal çıkış betiklerini `source` etme; `tmux` kullan |

---

## Sonraki Adımlar

| Hedef | Doküman |
|---|---|
| Tam havuz ve kendi node referansı | [guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md) |
| Kendi donanımına geçmek | [ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md) |
| Ağ bilgileri ve bağlantılar | [README.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.tr.md) |

---

## Yazar Hakkında

**HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
