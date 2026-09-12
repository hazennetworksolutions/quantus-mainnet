<div align="center">

# 🐧 Quantus Madenciliği İçin Ubuntu Kurulumu

**NVIDIA kartlı bir Windows PC'yi Linux madencilik makinesine çevir — Windows'u kaybetmeden**
*USB bellek, yedek SSD'ye çift açılış, GPU sürücüsü, sonra madencilik rehberine devir.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/download/desktop)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Yazar:** HazenNetworkSolutions
> **Amacı:** yalnızca işletim sistemini hazrlamak — burada madencilik yazılımı kurulmuyor
> **Bitiş:** [guide.tr.md → Adım 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-1--cüzdan-oluştur)
> **Son Güncelleme:** Eylül 2026

---

## Neden Zahmet Etmeli

Aynı kart Linux CUDA altında Windows madencisine kıyasla **4–6 kat hızlı** kazıyor. Bir RTX 4090 ~179 MH/s'den 1,1 GH/s üzerine çıkıyor. Aşağıdaki kurulum yaklaşık bir saat sürüyor ve kendini ilk gün amorti ediyor.

Bu doküman madenciliğin başladığı yerde biter. `nvidia-smi` kartını basıyorsa ve çalışan bir Ubuntu masaüstüne açıldıysan **[guide.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md)** ile devam et ve havuz madenciliği (Bölüm A) ya da kendi node'un (Bölüm B) arasında seçim yap.

**Gerekenler:** NVIDIA GPU'lu bir Windows PC, 8 GB+ USB bellek ve ya bir **yedek SSD** ya da boş bir bölüm. Bunun yerine GPU kiralıyorsan hepsini atla ve [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md)'yi oku.

---

## Sürüm Seçimi

| GPU'n | Kur |
|---|---|
| RTX 20 / 30 / 40 serisi | **Ubuntu 24.04 LTS** — güvenli varsayılan |
| RTX 50 serisi (Blackwell) | **Ubuntu 26.04 LTS** — daha yeni çekirdek, 570+ sürücü |

Her zaman **Desktop LTS** seç. LTS olmayan sürümlerin desteği dokuz ayda biter ve Server sürümü bir madencilik makinesine hiçbir şey katmaz.

---

## Adım 1 — ISO'yu İndir ve Doğrula

Desktop ISO'sunu [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) adresinden indir, sonra PowerShell'de doğrula:

```powershell
Get-FileHash -Algorithm SHA256 "$env:USERPROFILE\Downloads\ubuntu-24.04-desktop-amd64.iso"
```

Çıktıyı indirmenin yanında yayınlanan `SHA256SUMS` dosyasıyla karşılaştır. Uyuşmuyorsa dosya bozuk — kurmak yerine yeniden indir.

---

## Adım 2 — USB Belleği Yaz

[Rufus](https://rufus.ie/) kullan (portable sürüm de olur).

| Ayar | Değer |
|---|---|
| Device | USB belleğin — **iki kez kontrol et, silinecek** |
| Boot selection | Ubuntu ISO'su |
| Partition scheme | **GPT** |
| Target system | **UEFI (non CSM)** |
| File system | FAT32, varsayılan küme boyutu |
| Persistent partition size | **0** |

Rufus sorarsa **ISO modunda** yaz. Kalıcılığı sıfırda tut: bu bellek bir kurucu, canlı sistem değil; kalıcılık kafa karıştıran açılış davranışlarına yol açar.

---

## Adım 3 — USB'den Aç

1. Yeniden başlat ve firmware menüsünü aç (üreticiye göre `F2`, `F10`, `F12`, `Del` ya da `Esc`).
2. USB belleğin **UEFI** girişini seç — legacy/CSM olanı değil.
3. **Try or Install Ubuntu**'yu seç.

USB görünmüyorsa firmware'de Fast Boot'u kapat. Secure Boot'u da şimdi kapatmayı düşün: tescilli NVIDIA sürücüsü ya Secure Boot'un kapalı olmasını ya da kayıtlı bir MOK anahtarı ister.

---

## Adım 4 — Hedef Diski Belirle

**Windows diskinin silinmesini önleyen adım budur.** Canlı oturumun terminalinde:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MODEL
sudo lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,ROTA
sudo blkid | grep -i ntfs      # NTFS = Windows, dokunma
sudo fdisk -l
```

Hedef cihazı (`/dev/nvme1n1`, `/dev/sdb`, …) harfini tahmin ederek değil **model, seri numarası ve boyutu** eşleştirerek not al. Windows NTFS bölümlerini tutan disk dokunulmazdır.

---

## Adım 5 — Kur

**Install Ubuntu**'yu çalıştır ve seç:

- Normal installation
- **Install third-party software for graphics and Wi-Fi hardware** — bunu işaretle; NVIDIA sürücüsünü kurulum sırasında getirir
- Bağlantın izin veriyorsa kurulum sırasında güncellemeleri indir

Disk adımında:

| Durum | Seçim |
|---|---|
| Boş yedek SSD, Windows başka diskte | **Erase disk and install Ubuntu**, sonra o yedek diski açıkça seç |
| Windows'la aynı tek disk | **Install alongside Windows** ya da Manual ile `/` bölümünü boş alana koy |

> ⚠️ "Erase disk" **seçilen** diski tamamen siler. Devam etmeden önce özet ekranında Adım 4'teki cihaz adını doğrula. Özette Windows diskin görünüyorsa dur ve geri dön.

Gerçekten hatırlayacağın bir kullanıcı adı ve şifre belirle — sonraki her komut `sudo` kullanıyor. İstendiğinde USB'yi çıkar ve yeniden başlat.

---

## Adım 6 — GPU Sürücüsünü Kur

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

Yeniden başlattıktan sonra:

```bash
nvidia-smi
nvidia-smi -L
```

Kart adını, sürücü sürümünü ve bir `CUDA Version` kolonunu görmelisin.

| Sorun | Çözüm |
|---|---|
| `nvidia-smi` yok ya da cihaz görmüyor | Secure Boot modülü engelliyor — firmware'den kapat ya da MOK anahtarını kaydet, sonra yeniden başlat |
| RTX 50 serisi tanınmıyor | 570+ sürücü gerekir; Ubuntu 26.04 kur ya da graphics-drivers PPA'sını ekle |
| Ekran düşük çözünürlükte kalıyor | Önceki çekirdek girişiyle aç ve `ubuntu-drivers autoinstall`'u tekrar çalıştır |

> Tam `cuda-toolkit`'e **ihtiyacın yok**. Hazır Quantus madencileri kendi CUDA çalışma zamanını taşır; sürücü yeterli.

---

## Adım 7 — Makinenin Uyumasını Engelle

Uyuyan masaüstü hiçbir şey kazmaz. GNOME için:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

Ayrıca **Settings → Power → Automatic Suspend: Off** yap. Ekransız bir makinede sistem uykusunu da maskele:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

---

## Adım 8 — Son Kontrol Listesi

| Kontrol | Komut | Beklenen |
|---|---|---|
| Ubuntu sürümü | `lsb_release -a` | 24.04 ya da 26.04 LTS |
| GPU görünür | `nvidia-smi -L` | kartın, adıyla |
| Windows hâlâ açılıyor | firmware menüsünden yeniden başlat | Windows girişi duruyor |
| Disk alanı | `df -h /` | Bölüm B düşünüyorsan 100 GB+ boş |
| Ağ | `ping -c 3 quantus.com` | yanıt geliyor |
| Uyku kapalı | `gsettings get … sleep-inactive-ac-type` | `'nothing'` |

Altısı da tamam mı? Makine hazır.

---

## Sonraki Adım

**[guide.tr.md → Adım 1 — Cüzdan Oluştur](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-1--cüzdan-oluştur)** ile devam et. Oradaki Adım 2 `nvidia-smi` kontrolünü tekrarladığı için hızlı geçebilirsin, sonra seç:

- **Bölüm A — havuz madenciliği:** tek binary, tek servis, günler içinde gelir.
- **Bölüm B — kendi node'un:** tam senkron, sıfır komisyon, tam blok ödülleri.

Kararsz isen geri alınabilir seçim Bölüm A.

---

## Yazar Hakkında

**HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
