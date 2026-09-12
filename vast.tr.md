<div align="center">

# ☁️ Kiralık GPU ile Quantus Madenciliği (Vast.ai)

**Saatlik NVIDIA GPU kirala ve Quantus havuz madencisini üzerinde çalıştır — donanım yok, node yok, makinede seed yok**
*Makine seçimi, SSH anahtarları, madenci kurulumu, supervisor ile kalıcılık, maliyet kontrolü ve kapatma — adım adım.*

[![Vast.ai](https://img.shields.io/badge/Pazar%20Yeri-Vast.ai-1A73E8?style=flat-square)](https://vast.ai)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Mod](https://img.shields.io/badge/Mod-Havuz%20PPLNS-orange?style=flat-square)](https://quanpool.com/)
[![Kalıcılık](https://img.shields.io/badge/Servis-supervisor-yellow?style=flat-square)](http://supervisord.org/)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Hazırlayan:** HazenNetworkSolutions
> **Hedef:** saatlik faturalanan, kiralık bir NVIDIA konteyneri
> **Sonuç:** supervisor altında, yeniden başlatmalara dayanan `quanpool-miner`; makinede hiç cüzdan malzemesi yok
> **Son Güncelleme:** Eylül 2026

---

## Kapsam

| Aşama | Doküman |
|---|---|
| GPU kirala ve üzerinde kaz | **Bu rehber** |
| Madencilik kavramları, A/B kararı, kendi node yolu | → [guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md) |
| Kendi Windows PC'ni kullan | → [ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md) |
| Ağa genel bakış ve bağlantılar | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

Burada düz bir NVIDIA Ubuntu imajı, `quanpool-miner` ve madencinin kendi kendine yeniden başlaması için bir supervisor programı kuruyoruz. Bu rehber **kendi node'unu kurmaz**, seed ifadeni hiçbir yere yazmaz ve evde zaten çalışan bir rig'i değiştirmez.

**Ana rehberle ilişkisi.** guide.tr.md madenciliği iki yola ayırır: **Yol A** (sadece havuz) ve **Yol B** (kendi node'un). Bu doküman, kiralık donanım için **kendi içinde tam bir Yol A varyantıdır** — guide.tr.md'deki Adım 2 ve A1–A6 adımlarını konteyner karşılıklarıyla değiştirir; çünkü kiralık bir kutunun kalıcı kimliği, sabit adresi ve zincir senkronlamak için bir sebebi yoktur.

```text
guide.tr.md  Adım 1   cüzdan oluştur         ← orada yapacağın tek şey
      ↓
vast.tr.md   Adım 1 → 11                     ← geri kalan her şey burada
      ↓
vast.tr.md   Maliyet Kontrolü ve Kapatma     ← bu yolun bittiği yer
```

**Burada başlar:** [guide.tr.md → Adım 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-1--cüzdan-oluştur)'den aldığın bir `qz…` adresiyle. Tek ön koşul bu.
**Bittiği yer:** [Adım 11](#adım-11--havuzda-doğrula), işçin havuz sayfasında göründüğünde — ardından sayaca bağlı donanımda okuması opsiyonel olmayan [Maliyet Kontrolü ve Kapatma](#maliyet-kontrolü-ve-kapatma) bölümü.
**Burada hiç dokunulmayanlar:** `quantus-node`, zincir senkronu, içe açık güvenlik duvarı kuralları, wormhole inner hash. O yolu istiyorsan kendi makinen ve [Yol B](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#yol-b--kendi-nodeun-resmî-yol) lazım.

> ⚠️ **Kiralık makineye hiçbir sır gitmez.** Kutuya tam olarak iki değer gerekir: `qzADRESIN.iscininadi` ve havuzun TLS pin'i. 24 kelimen, cüzdan dosyan ve node inner hash'in bunun dışında kalır. Sağlayıcının her dosyayı ve her süreci görebileceğini varsay.

---

## İçindekiler

- [GPU Pazar Yeri Nedir](#gpu-pazar-yeri-nedir)
- [Adım 1 — Hesap ve Bakiye](#adım-1--hesap-ve-bakiye)
- [Adım 2 — Doğru Kartı Seç](#adım-2--doğru-kartı-seç)
- [Adım 3 — Doğru Docker Görüntüsünü Seç](#adım-3--doğru-docker-görüntüsünü-seç)
- [Adım 4 — Arama Filtreleri ve Kiralama](#adım-4--arama-filtreleri-ve-kiralama)
- [Adım 5 — SSH Anahtarı ve Bağlantı](#adım-5--ssh-anahtarı-ve-bağlantı)
- [Adım 6 — Konteynerde Temel Kontroller](#adım-6--konteynerde-temel-kontroller)
- [Adım 7 — Madenciyi Kur](#adım-7--madenciyi-kur)
- [Adım 8 — Token ve TLS Pin](#adım-8--token-ve-tls-pin)
- [Adım 9 — Manuel Çalıştırma](#adım-9--manuel-çalıştırma)
- [Adım 10 — supervisor ile Kalıcı Hale Getir](#adım-10--supervisor-ile-kalıcı-hale-getir)
- [Adım 11 — Havuzda Doğrula](#adım-11--havuzda-doğrula)
- [Maliyet Kontrolü ve Kapatma](#maliyet-kontrolü-ve-kapatma)
- [Güvenlik Kontrol Listesi](#güvenlik-kontrol-listesi)
- [Sorun Giderme](#sorun-giderme)

---

## GPU Pazar Yeri Nedir

Vast.ai, bağımsız sağlayıcıların GPU'larını saatlik kiraya verdiği bir pazar yeridir. Ön ödemeli bakiyeden ödersin, başkasının makinesinde bir konteyner açılır ve SSH ile bağlanırsın. Çoğu makine **sanal makine değil, konteynerdir**: `root` gibi görünürsün ama çekirdek modülü yükleyemez, içinde Docker çalıştıramazsın.

Bu rehberin şeklini belirleyen üç sonuç:

1. **Faturalama süreklidir.** Çalışan bir makine, GPU meşgul olsun ya da olmasın bakiye yer. Sayacı durduran tek şey makineyi yok etmektir.
2. **`systemd` genelde çalışmaz.** Uzun süren süreçler imajın **supervisor**'ı altına girer.
3. **Depolama geçicidir.** Bağlı bir volume yoksa **Destroy her şeyi siler** — madenci, token dosyası, loglar.

Kiralık madencilik sadece havuz madenciliği olarak mantıklıdır: kiralık kutunun kalıcı kimliği, sabit adresi ve zincir senkronlamak için bir gerekçesi yoktur. Yani [guide.tr.md Yol B](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#yol-b--kendi-nodeun-resmî-yol) buradaki yol değil.

---

## Adım 1 — Hesap ve Bakiye

1. [vast.ai](https://vast.ai) üzerinde hesap aç (konsol ayrıca [cloud.vast.ai](https://cloud.vast.ai)).
2. **Billing** altından bakiye yükle. Dayanma süren `bakiye ÷ saatlik fiyat`.
3. İki konsol sayfası önemli: **Search** (kiralanabilir teklifler) ve **Instances** (para ödediğin kutular).

İki faktörlü doğrulamayı aç ve başka yerde kullanmadığın bir şifre seç — ele geçirilen bir pazar yeri hesabı doğrudan parasal kayıptır.

---

## Adım 2 — Doğru Kartı Seç

Quantus QPoW (Poseidon2) **hesaplama sınırlıdır, VRAM sınırlı değil**: 8–12 GB fazlasıyla yeter. Kiralık GPU'da en pahalıya patlayan yanılgı budur — 80–140 GB'lık bir veri merkezi kartı burada "onlarca kat hızlı" değildir, ama saati birkaç kat pahalıdır. Tüketici kartları hash başına maliyette genelde açık ara kazanır.

Yaklaşık Linux CUDA hızları, sadece boyutlandırma için — kendi makinende `benchmark` ile teyit et:

| Kart | Yaklaşık hash hızı | Not |
|---|---|---|
| RTX 4090 | ~1,1 GH/s | Genelde hash başına en iyi maliyet |
| RTX 5090 | 4090'ın üzerinde | Kazandığını varsaymadan önce saatlik fiyata bak |
| RTX 4070 / 5070 | ~410–460 MH/s | İkisi kabaca bir 4090 eder |
| RTX 3080 / 3080 Ti | ~430 MH/s | 10–12 GB yeterli |
| H100 / H200 / B200 | bir 4090 ile onun birkaç katı arası | VRAM boşa gider; saatlik maliyet nadiren haklı çıkar |

> **"Bir saatlik dünya kartlarını kiralayıp kendi bloğumu bulurum"** işlemez. TH/s ile ölçülen bir ağa karşı tek kutunun birkaç saati bir loto biletidir. PPLNS ise seni paylar üzerinden öder, yani aynı kiralık hash gücü düzenli kazanır.

Doğrulanmamış sağlayıcıların çok düşük fiyatları bazen gerçekten ucuz, bazen kullanılamaz çıkar — **Loading**'de kalan bir imaj, HDD, 40 Mbps bant genişliği. İlk denemede **Verified** sağlayıcıları, NVMe'yi, 200+ Mbps'i ve güncel CUDA sürücüsünü tercih et. Bir makine sorun çıkarırsa yok et ve başka teklif al; kaybın birkaç kuruş.

---

## Adım 3 — Doğru Docker Görüntüsünü Seç

LLM, ComfyUI ya da CUDA-devel template'lerini **seçme**. Hiç kullanmayacağın onlarca gigabaytlık katman indirirler ve inerken sen para ödersin.

İstediğin şey, sürücüyü sağlayıcının enjekte ettiği **düz bir NVIDIA Ubuntu temel imajı** — genelde pazar yerinin kendi base ya da Jupyter CUDA imajı. Madenci, sonradan kendin eklediğin ~13 MB'lık tek bir dosya.

| Ayar | Seçim |
|---|---|
| Template | Yok / düz temel imaj |
| Instance disk | 16–32 GB fazlasıyla yeter |
| Volume | Gerekmez — ama olmadan Destroy her şeyi siler |
| GPU sayısı | Teklifte ne varsa; aynı sayıyı `--gpu-devices` olarak ver |

Bir Jupyter arayüzü çalışıyor olabilir. Kullanmak zorunda değilsin ve üzerinden başka bir şey yayınlamamalısın.

---

## Adım 4 — Arama Filtreleri ve Kiralama

Kiralamadan önce Search listesini darlaştır:

- **GPU modeli:** Adım 2'deki tüketici kartları.
- **GPU sayısı:** **1×** ile başla. İki kartlı kutu yaklaşık iki katına çıkar ve yaklaşık iki kat hash verir — büyük kutuda öğrenmenin indirimi yok.
- **Disk:** en az ~16 GB.
- **Verified / NVMe / bant genişliği:** yukarıdaki gibi.

Fiyatı **$/saat** olarak oku: `saatte 0,17 $ × 24 ≈ günde 4,10 $`. Bir kart için piyasanın çok altındaki fiyatlar heves değil şüphe hak eder.

**Rent**'e bas, sonra **Instances**'ı izle. **Loading**, imajın hâlâ indiği anlamına gelir; 10–15 dakika sonra hâlâ yüklüyorsa yok et ve başka teklif seç. Hazır bir makine **running** durumunda ve GPU %0'a yakın görünür — henüz hiçbir şey kazmıyor.

---

## Adım 5 — SSH Anahtarı ve Bağlantı

Pazar yeri şifreyle değil, **SSH açık anahtarı** ile doğrulama yapar. Bu işe özel bir anahtar üret — kişisel sunucularınınkini yeniden kullanma:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/vast_quantus -N '' -C 'vast-quantus'
chmod 600 ~/.ssh/vast_quantus
cat ~/.ssh/vast_quantus.pub
```

Ekrana gelen `ssh-ed25519 AAAA…` satırını kopyala — bu **açık** yarıdır ve konsola yapıştırmak güvenlidir. Uzantısız dosya (`vast_quantus`) özel anahtardır ve makinenden hiç çıkmaz.

**Instances** içinde SSH / anahtar panelini aç, açık anahtarı *add key* alanına yapıştır ve listede göründüğünü doğrula. Yayılması için birkaç saniye tanı.

Panel ayrıca `ssh -p <PORT> root@<HOST_IP> -L 8080:localhost:8080` şeklinde bir bağlantı satırı gösterir. Port ve adres her makinede farklıdır; `-L 8080` tüneli Jupyter içindir, madencilik için değil. Kendi özel anahtarınla bağlan:

```bash
ssh -p <PORT> -i ~/.ssh/vast_quantus \
  -o IdentitiesOnly=yes \
  -o StrictHostKeyChecking=accept-new \
  root@<HOST_IP>
```

`Permission denied (publickey)` alırsan: birkaç saniye bekle, anahtarın *o* makinede listelendiğini teyit et ve `-i`'den sonraki yolu kontrol et. Sağlayıcının güvenlik duvarı doğrudan SSH'ı kesiyorsa konsolun proxy SSH adresini kullan. Windows'tan aynı komut Windows Terminal'de çalışır.

---

## Adım 6 — Konteynerde Temel Kontroller

```bash
nvidia-smi -L
nvidia-smi
```

Kart sayısını, sürücü ve CUDA sürümünü doğrula. `quanpool-miner` kendi CUDA çalışma zamanını taşıdığı için Ada ve Ampere'de makul güncel her sürücü iş görür; 50 serisi (Blackwell) bir kart sağlayıcıda CUDA 12.8+ ister.

İmaj, kendi dokümantasyonuna işaret eden bir karşılama metni yazdırabilir — okumaya değer, çünkü imajın hangi supervisor düzenini kullandığını söyler. Onu bir kılavuz gibi değerlendir, körlemesine uygulanacak emirler gibi değil.

Kurulum yerini belirleyen tek detay: `/workspace` altındaki bir yol aynı konteynerin stop/start'ından sağ çıkar, ama volume bağlamadıysan **Destroy'dan sağ çıkmaz**.

---

## Adım 7 — Madenciyi Kur

Güncel sürümü ve indirme bağlantısını [quanpool.com](https://quanpool.com/) → **Start mining** üzerinden doğrula. Bu satırlar yazılırken Linux CUDA sürümü **6.2.0**.

```bash
mkdir -p /workspace/quantus
cd /workspace/quantus
wget -O quanpool-miner https://download.quanpool.com/quanpool-miner-6.2.0-linux-x86_64
chmod u+x quanpool-miner
./quanpool-miner --version

# opsiyonel yerel benchmark — havuza hiç bağlanmaz, ama geçen süre faturalanır
./quanpool-miner benchmark --cpu-workers 0 --duration 20
```

Sonucu Adım 2 ile karşılaştır. Modern bir kartta kabaca 100 MH/s, CUDA yolunda olmadığın anlamına gelir — bkz. [guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-a2--kartı-benchmark-et) tablosu.

> `gpu-list` alt komutu 6.1+ sürümlerinde kaldırıldı. Kartları `nvidia-smi -L` ile say.

---

## Adım 8 — Token ve TLS Pin

Diğer makinelerinle **aynı `qz…` adresini** ve **farklı bir işçi adını** kullan. Tüm işçilerinin PPLNS payları toplanır; aynı adlar çakışır ve biri havuzdan kaybolur.

İşçi adı kuralları: `a-z 0-9 . - _`, en fazla 32 karakter, boşluk yok. Paylaşılan bir sağlayıcıda `pc1` gibi aşikar adlardan kaçın.

```bash
cd /workspace/quantus

# tek satır: qzADRESIN.benzersizisci
nano auth-token

# Start mining'deki 64 haneli TLS pin
nano tls-cert-sha256

chmod 600 auth-token tls-cert-sha256
```

Pin havuzdaki herkes için aynıdır, ama eski bir nottan değil siteden kopyala — havuz sertifikasını yenilediğinde değişir. Bu değerleri asla `--auth-token` ile satır içinde verme; dosya parametreleri onları kabuk geçmişinden uzak tutar.

`--gpu-devices` değerini `nvidia-smi -L` çıktısındaki satır sayısına ayarla. Parametreyi hiç vermezsen madenci tüm kartları denemeye çalışır.

---

## Adım 9 — Manuel Çalıştırma

Canlı havuz adresini **Start mining**'den al — biçimi `IP:9834`:

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

Sağlıklı belirtiler: GPU kullanımı %98'e yakın, logda `SHARE FOUND` ve `curl -s http://127.0.0.1:9900/hive-stats`'tan sıfır reddetmeli dolu bir yanıt.

Kiralık bir veri merkezinden gidiş-dönüş süresi evden yüksek olabilir (300–400 ms alışılmadık değil). Bu hash hızını düşürmez — havuz pay zorluğunu ayarlar. Sürekli `timed out` döngüsü ise başka şey: o sağlayıcı dışa UDP/QUIC'i kesiyor, makineyi yok et ve başka bölgede kirala.

`Ctrl+C` ile durdur ve supervisor altına al — yoksa madenci SSH oturumunla birlikte ölür.

---

## Adım 10 — supervisor ile Kalıcı Hale Getir

Temel imajın deseni: `/opt/supervisor-scripts/` içinde bir betik, artı `/etc/supervisor/conf.d/` içinde ona uyan bir yapılandırma.

```bash
cat > /opt/supervisor-scripts/quanpool-miner.sh << 'EOF'
#!/bin/bash
utils=/opt/supervisor-scripts/utils
. "${utils}/logging.sh"
. "${utils}/environment.sh"
cd /workspace/quantus
exec /workspace/quantus/quanpool-miner serve \
  --node-addr <POOL_HOST>:9834 \
  --auth-token-file /workspace/quantus/auth-token \
  --tls-cert-sha256-file /workspace/quantus/tls-cert-sha256 \
  --cpu-workers 0 \
  --gpu-devices 1 \
  --mode pool
EOF
chmod 755 /opt/supervisor-scripts/quanpool-miner.sh
```

Devam etmeden önce `<POOL_HOST>`'u canlı adrese ve `--gpu-devices`'ı kart sayına ayarla.

> Bu betikte imajın portal-exit yardımcısını source etme. Portal listesinde olmayan bir programın atlanması gerektiğine karar verebilir; bu da madencinin sessizce hiç başlamamasına yol açar.

```bash
cat > /etc/supervisor/conf.d/quanpool-miner.conf << 'EOF'
[program:quanpool-miner]
environment=PROC_NAME="%(program_name)s"
command=/opt/supervisor-scripts/quanpool-miner.sh
autostart=true
autorestart=true
startsecs=8
stopasgroup=true
killasgroup=true
stopsignal=TERM
stopwaitsecs=15
stdout_logfile=/var/log/portal/quanpool-miner.log
redirect_stderr=true
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=3
EOF

supervisorctl reread
supervisorctl update
supervisorctl status quanpool-miner
```

Durum birkaç saniye sonra `STARTING` → `RUNNING` olur; CUDA ısınması 10–20 saniye sürer. Günlük kullanım:

```bash
supervisorctl status quanpool-miner
supervisorctl restart quanpool-miner
tail -f /var/log/portal/quanpool-miner.log
curl -s http://127.0.0.1:9900/hive-stats
nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,power.draw --format=csv
```

İmajın kendi yönetim programlarını (portal, tünel yöneticisi, ters proxy) çalışır bırak — konsol kutuya onlar üzerinden ulaşıyor.

> **`9900` portunu asla yayınlama.** Metrikler senin için, SSH üzerinden, localhost'ta. Kiralık bir kutuda port açmak onu tüm internete açar.

---

## Adım 11 — Havuzda Doğrula

[quanpool.com](https://quanpool.com/) adresini aç, `qz…` adresini yapıştır ve **Look up**'a bas. Yeni işçi birkaç dakika içinde, zaten çalıştırdığın rig'lerin yanında görünür.

`hs` değeri **kH/s** cinsindendir — `440000` ≈ 440 MH/s. İki kartlı bir makine iki GPU satırı gösterir ve redler sıfırda kalmalı. İlk saniyelerde bir grup `SOLUTION LOST` / stale mesajı normaldir; birkaç dakika sonra düzenli `SHARE FOUND` satırları görmelisin.

### Bitiş Çizgisi

Şu beşi doğruysa kiralık yol tamamlandı:

1. `supervisorctl status quanpool-miner` `RUNNING` diyor.
2. `nvidia-smi` %95–100 kullanım ve Adım 2'de beklediğin hızı gösteriyor.
3. `hive-stats` kabul edilen payları artarken redleri sıfırda gösteriyor.
4. İşçi, `qz…` adresinin altında havuz sorgu sayfasında görünüyor ve adı başka hiçbir makinede kullanılmıyor.
5. Saatlik fiyatı biliyorsun ve makineyi ne zaman yok edeceğine karar verdin.

Kuracak başka bir şey yok. Geri kalanı para yönetimi — hemen aşağıda.

---

## Maliyet Kontrolü ve Kapatma

Harcaman, saatlik fiyat çarpı makinenin var olduğu süredir.

| İşlem | Ne olur |
|---|---|
| Madenciyi durdurmak (`supervisorctl stop`) | GPU boşa çıkar, **fatura işlemeye devam eder** |
| Makineyi **Stop** etmek | Konteyner durur; faturalama ve disk saklama sağlayıcının politikasına bağlı |
| Makineyi **Destroy** etmek | Makine ve disk silinir, fatura biter. Madenci, token ve loglar gider |
| **Reboot** | `autostart=true` ile supervisor madenciyi geri getirir |

İşin bittiğinde **Destroy** et. Unutulan bir makine bakiyeni sessizce yer ve sadece madenciyi durdurmak faturayı durdurmaz.

Aynı teklifi tekrar kiralamak, volume bağlamadıysan sıfırdan kurulum demektir — rehberi Adım 5'ten itibaren uygula. Eski makine yok edildikten sonra aynı işçi adını yeniden kullanmak sorun değil.

---

## Güvenlik Kontrol Listesi

- [ ] Makinenin hiçbir yerinde seed ifadesi, cüzdan dosyası ya da node inner hash'i yok
- [ ] Kiralık kutular için ayrı bir SSH anahtarı, kişisel sunucu anahtarı değil
- [ ] `auth-token` ve `tls-cert-sha256` `600` modunda, dosya olarak veriliyor, asla satır içi değil
- [ ] `9900` portu ve madenci portu dışa açık değil
- [ ] İşçi adı çalıştırdığın diğer tüm makinelerden farklı
- [ ] Saatlik fiyatı biliyorsun ve iş bittiğinde Destroy edeceksin

---

## Sorun Giderme

| Belirti | Neye bakmalı |
|---|---|
| Makine 15+ dakika **Loading**'de takılı | Yavaş depolama ya da çok büyük imaj. Yok et, Verified NVMe teklifi al |
| `Permission denied (publickey)` | Anahtar *o* makineye eklendi mi, `-i`'den sonraki yol doğru mu, birkaç saniye bekle |
| Doğrudan SSH hiç bağlanmıyor | Sağlayıcı güvenlik duvarı — konsolun proxy SSH adresini kullan |
| `nvidia-smi` yok ya da GPU listelenmiyor | Yanlış imaj, ya da GPU geçirilmemiş. Başka yerde yeniden oluştur |
| CUDA ya da PTX hatası | Sağlayıcı sürücüsü çok eski, ya da 50 serisi kartta eski bir sürüm. Güncel madenciyi ya da başka teklifi dene |
| Bitmeyen `timed out` / yeniden bağlanma | O sağlayıcı havuz portunda dışa UDP'yi kesiyor. Yok et, başka bölgede kirala |
| `certificate` / `fingerprint` hatası | 64 haneli TLS pin'i **Start mining**'den yeniden kopyala |
| İşçi sitede hiç görünmüyor | `auth-token` içindeki adres ile sorguladığını karşılaştır, işçi adı kuralları, 2–3 dakika tanı |
| supervisor `RUNNING` ama GPU %0 | Logu oku: token, pin ya da `--node-addr` yanlış. `supervisorctl tail quanpool-miner` |
| Güçlü bir kartta ~100 MH/s | CUDA yolu değil — yanlış binary ya da yanlış GPU parametreleri |
| Evdeki işçi kayboldu | Evdeki işçi adını tekrar kullanmışsın. Bunu yeniden adlandır ve ikisini de başlat |
| SSH kapanınca madenci ölüyor | Hâlâ ön planda çalışıyor — Adım 10'u bitir |

---

## Sıradaki Adımlar

| Hedef | Doküman |
|---|---|
| Madencilik kavramları, A/B kararı, kendi node yolu | [guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#hangi-yol--a-mı-b-mi) |
| Windows PC'yi madencilik makinesine çevirmek | [ubuntu.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/ubuntu.tr.md) |
| Ağ bilgileri ve bağlantılar | [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md) |

---

## Yazar Hakkında

Bu rehber **HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
