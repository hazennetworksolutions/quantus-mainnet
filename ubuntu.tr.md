<div align="center">

# 🐧 Windows PC → Quantus Madenciliği İçin Ubuntu Desktop

**Windows'u kaybetmeden yedek bir diske Ubuntu kur ve NVIDIA sürücüsünü çalıştır**

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04%20LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/download/desktop)
[![Quantus](https://img.shields.io/badge/Quantus-Mainnet-6C4DF6?style=flat-square)](https://quantus.com)
[![GPU](https://img.shields.io/badge/GPU-NVIDIA%20CUDA-76B900?style=flat-square&logo=nvidia&logoColor=white)](https://docs.quantus.com/deep-dives/qpow)
[![Boot](https://img.shields.io/badge/Boot-UEFI%20%C2%B7%20GPT-blue?style=flat-square)](https://rufus.ie/en/)
[![Dual Boot](https://img.shields.io/badge/Windows-Korunur-0078D4?style=flat-square&logo=windows&logoColor=white)](https://ubuntu.com/tutorials/install-ubuntu-desktop)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Hazırlayan:** HazenNetworkSolutions
> **Hedef:** NVIDIA GPU'lu ve yedek SSD'li bir Windows 10/11 masaüstü
> **Sonuç:** çalışan CUDA sürücüsüyle bare-metal Ubuntu, Windows'a dokunulmamış hâlde
> **Son Güncelleme:** Eylül 2026

---

## Kapsam

Bu doküman tek bir iş yapar: bir Windows makinesini, NVIDIA sürücüsü çalışan bir Ubuntu makinesine çevirir. **Madenci yok, cüzdan yok, seed ifadesi yok ve henüz havuz/node seçimi de yok.**

**Nerede biter:** [Adım 8](#adım-8--son-kontrol)'de. Oradan ana rehbere devam edersin; o rehber iki ortak adımla başlar ve ancak ondan sonra yol seçmesini ister:

```text
ubuntu.tr.md  Adım 1 → 8     şu an buradasın
      ↓
guide.tr.md   Adım 1  cüzdan oluştur
guide.tr.md   Adım 2  GPU'yu doğrula   (burada kurduğunu 2 dakikada teyit)
guide.tr.md   Karar Noktası → Yol A (havuz) veya Yol B (kendi node'un)
```

Buradan doğrudan Yol A ya da Yol B'ye atlamaya çalışma. guide.tr.md Adım 1'deki cüzdan ikisinin de ön koşulu.

GPU kiralamayı mu düşünüyorsun? → [vast.tr.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/vast.tr.md) · Ağa genel bakış → [README.md](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/README.md)

**Neden zahmet:** hızlı CUDA yolu sadece Linux'ta var. Windows madencisi aynı kartta **4–6 kat yavaş** — orta seviye bir RTX'te 400–450 MH/s yerine ~100 MH/s.

---

## İçindekiler

- [Gerekenler](#gerekenler)
- [Adım 1 — ISO'yu Al ve Doğrula](#adım-1--isoyu-al-ve-doğrula)
- [Adım 2 — Rufus ile USB Yaz](#adım-2--rufus-ile-usb-yaz)
- [Adım 3 — BIOS ve Boot Menüsü](#adım-3--bios-ve-boot-menüsü)
- [Adım 4 — Hedef Diski Belirle](#adım-4--hedef-diski-belirle)
- [Adım 5 — Ubuntu'yu Kur](#adım-5--ubuntuyu-kur)
- [Adım 6 — Güncellemeler ve NVIDIA Sürücüsü](#adım-6--güncellemeler-ve-nvidia-sürücüsü)
- [Adım 7 — Uyku Modunu Kapat](#adım-7--uyku-modunu-kapat)
- [Adım 8 — Son Kontrol](#adım-8--son-kontrol)
- [Sorun Giderme](#sorun-giderme)

---

## Gerekenler

| Kalem | Gereksinim |
|---|---|
| PC | **UEFI** modunda açılan Windows 10/11 (Legacy/CSM kapalı) |
| GPU | NVIDIA RTX 20/30/40/50 serisi — AMD, CUDA madencisiyle çalışmaz |
| Disk | Üzerinde değerli hiçbir şey olmayan **ayrı bir SSD** — kurulum onu siler |
| USB bellek | 8 GB+. Rufus sileceği için önce yedekle |
| Ağ | Kablolu Ethernet tercih edilir — kurulum sırasında sürücü iner |

Üç temel kural:

1. **Bare metal.** WSL değil, sanal makine değil.
2. **Windows kalsın.** Fiziksel disk başına tek işletim sistemi. *Windows ile birlikte kur* seçeneğinden kaçın; o, Ubuntu'yu Windows diskine sıkıştırır.
3. **Bu makinede seed ifadesi yok.** Cüzdan telefonda oluşur — [guide.tr.md → Adım 1](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-1--cüzdan-oluştur).

---

## Adım 1 — ISO'yu Al ve Doğrula

1. [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) adresinden **Intel/AMD 64-bit Desktop** LTS imajını al — ARM ISO'su değil.
2. RTX 50 serisi (Blackwell) daha yeni bir çekirdek ister: **26.04.x**'i tercih et. 30/40 serisi için 24.04 LTS yeterli.
3. Aynı sayfadan `SHA256SUMS` dosyasını indir ve PowerShell'de doğrula:

```powershell
Get-FileHash .\ubuntu-*-desktop-amd64.iso -Algorithm SHA256
```

> ⚠️ Hash, `SHA256SUMS` satırıyla eşleşmiyorsa ISO'yu sil ve yeniden indir. Bozuk imaj yazma.

---

## Adım 2 — Rufus ile USB Yaz

Rufus'u [rufus.ie](https://rufus.ie/en/) adresinden al — imzada **Akeo Consulting** yazmalı. Taşınabilir `.exe` yeterli.

| Rufus alanı | Değer |
|---|---|
| Device | **USB bellek** — bunu iki kez kontrol et |
| Boot selection | Az önce doğruladığın ISO |
| Persistent partition size | **0** |
| Partition scheme | **GPT** |
| Target system | **UEFI (non CSM)** |
| File system | Large FAT32 (Rufus seçer) |
| Quick format | açık · bad-block kontrolü: kapalı |

**START**'a bas. ISOHybrid modu sorarsa DD değil, **ISO image mode** seç. **READY** yazmasını bekle, sonra çıkar.

> "Windows User Experience" penceresi çıkarsa yanlışlıkla bir Windows ISO'su seçmişsin.

---

## Adım 3 — BIOS ve Boot Menüsü

Boot menüsü genelde **F11 / F12 / F8 / Esc**; BIOS ayarları **Del / F2**.

| Ayar | Ne yapmalı |
|---|---|
| Boot kaydı | Başında **UEFI:** yazanı seç — asla `Legacy` ya da `CSM` |
| Secure Boot | Desteklenir, ama sonra `nvidia-smi` boş çıkarsa en hızlı çözüm **kapatmaktır** |
| Intel RST / RAID | Ubuntu AHCI ister. RST açık ve Windows kuruluysa değiştirmeden önce anakartını araştır — Windows'u bozabilir |

USB'den aç ve **Try or Install Ubuntu**'yu seç. Kurulumu henüz başlatma — önce Adım 4'ü yap.

---

## Adım 4 — Hedef Diski Belirle

Yanlış disk seçimi Windows'u yok eder ve kurulum ekranının listesinde iki aynı SSD birbirinin tıpatıp aynısı görünür. Diski, kurulumu başlatmadan **önce** canlı oturumdaki bir terminalden (`Ctrl+Alt+T`) belirle.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MODEL        # diskler ve bölümleri
sudo lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,ROTA # fiziksel diskler, model + seri no
```

Tipik çıktı:

```text
NAME        SIZE TYPE FSTYPE LABEL    MODEL
sda         1.8T disk        —        <disk modeli>
├─sda1      100M part vfat   SYSTEM
├─sda2       16M part
└─sda3      1.8T part ntfs   Windows
sdb       465.8G disk        —        <disk modeli>
nvme0n1   931.5G disk        —        <disk modeli>
└─nvme0n1p1 931.5G part ntfs Games
```

Nasıl okunur:

| Gördüğün | Ne demek | Karar |
|---|---|---|
| Küçük `vfat` (~100–500 MB) **+** büyük `ntfs` | Windows sistem diski — o `vfat` Windows EFI bölümü | **Asla seçme** |
| `ntfs` bölümler, EFI bölümü yok | Veri / oyun / yedek diski | **Asla seçme** |
| Hiç bölümü yok ya da kendi boşfaltığın | Hedefin | ✅ Bu |
| `ROTA` = `1` | SSD değil, tabaklı HDD | Kaçın |

Kesinleştirmeden önce çapraz kontrol:

```bash
sudo blkid | grep -i ntfs    # hangi bölümlerde Windows dosya sistemi var
sudo fdisk -l /dev/sdb       # adayın tam bölüm tablosu
```

`fdisk -l` çıktısında `EFI System` ya da `Microsoft reserved` bölümü görüyorsan o bir Windows diskidir — dur ve listeyi baştan oku.

Aygıtın tam adını, boyutunu ve modelini bir yere yaz (ör. `/dev/sdb, 465.8G`). Kurulumun özet ekranında bu metni birebir eşleyeceksin.

> ⚠️ İki aynı disk mi var? `SERIAL` kolonunu kullan ya da dokunmaman gerekeni fiziksel olarak sök. Tahmin etmek bir seçenek değil.

---

## Adım 5 — Ubuntu'yu Kur

Kurulumu başlat; dil ve klavyeyi seç, Ethernet'i tak, **Interactive installation** seç ve **üçüncü parti yazılım / NVIDIA sürücüleri** kutusunu işaretle.

Disk adımında:

- **Erase disk and install Ubuntu**'yu seç ve Adım 4'teki aygıtı göster — başka hiçbir şeyi değil.
- *Windows ile birlikte kur* seçeneğini **asla** kullanma.
- Kapalı bırak: TPM / tam disk şifreleme (NVIDIA sürücüleriyle çatışabilir) ve ZFS.

Özet ekranında model ve boyutu notunla karşılaştır. **Metin birebir uymuyorsa dur.**

Kullanıcı adı, makine adı ve güçlü bir şifre belirle. Otomatik giriş, sadece madencilik yapan bir kutuda pratiktir; dizüstünde kapalı tut. USB'yi çıkar ve yeniden başlat.

> GRUB Windows EFI bölümüne kurulduysa BIOS önyeklemesinde Windows diskini başa al ve Ubuntu'yu boot menüsünden seç. Kozmetik bir rahatsızlık — bunu asla disk silerek "düzeltme".

---

## Adım 6 — Güncellemeler ve NVIDIA Sürücüsü

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common wget curl ca-certificates netcat-openbsd jq
```

Kurulum sürücüyü yüklemediyse:

```bash
sudo ubuntu-drivers autoinstall
sudo reboot
```

Sonra aktif olduğunu teyit et:

```bash
nvidia-smi
nvidia-smi -L    # her GPU için bir satır — bu sayı --gpu-devices olacak
```

Kart adı, sürücü sürümü ve bir `CUDA Version` kolonu görmelisin. Kullanımın düşük olması normal — henüz hiçbir şey kazmıyor.

Boş ya da eksikse: Secure Boot'u kapat ve yeniden başlat → ya da `ubuntu-drivers devices` çalıştırıp önerilen sürücüyü açıkça kur → 50 serisi kartta sürücünün 570+ olduğunu doğrula.

> `cuda-toolkit`'e **ihtiyacın yok**. Madenciler kendi CUDA çalışma zamanını taşır; önemli olan tek şey çalışan bir `nvidia-smi`.

---

## Adım 7 — Uyku Modunu Kapat

Ekranın kararması sorun değil — ama **uyku modu GPU'yu durdurur**. Bunları grafik oturumu içinde, kendi masaüstü kullanıcın olarak çalıştır:

```bash
gsettings set org.gnome.desktop.session idle-delay 300
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

`idle-delay 300` ekranı beş dakika sonra karartır; `0` hiç karartmaz.

---

## Adım 8 — Son Kontrol

Bu dokümanın sonu burası. Buradan ayrılmadan önce her kutuyu işaretle:

- [ ] Ubuntu sadece Adım 4'teki diskte ve Windows hâlâ açılıyor
- [ ] `nvidia-smi` kartı, sürücüyü ve CUDA sürümünü gösteriyor
- [ ] `nvidia-smi -L`'in kaç GPU raporladığını biliyorsun
- [ ] Uyku modu kapalı
- [ ] `apt` internete ulaşıyor
- [ ] Bu makinede hiç seed ifadesi yazılmadı

✅ Hepsi tamamsa → **[guide.tr.md → Adım 1: Cüzdan Oluştur](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-1--cüzdan-oluştur)** ile devam et.

O cüzdan ve Adım 2'deki kısa GPU teyidi iki yolun ortak kurulumudur; ardından [Karar Noktası](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#karar-noktası--yolunu-seç) seni **Yol A**'ya (havuz, evde en kolayı) ya da **Yol B**'ye (kendi node'un) yönlendirir. Oraya varmadan önce dengeyi okumak istersen: [Hangi Yol](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#hangi-yol--a-mı-b-mi).

---

## Sorun Giderme

| Problem | Neye bakmalı |
|---|---|
| USB açılmıyor | Boot menüsünde **UEFI:** kaydını seç; GPT + UEFI (non CSM) ile yeniden yaz |
| ISO hash'i uyuşmuyor | ubuntu.com'dan yeniden indir |
| Hangi diski sileceğimden emin değilim | Adım 4'ü tekrar yap. Aynı diskler → `SERIAL` kullan ya da birini sök |
| Kurulum diski kabul etmiyor | Intel RST/RAID açık, ya da disk bir Windows dinamik biriminin parçası |
| Windows kayboldu gibi | Önyeklemede Windows diskini başa al. **Hiçbir şeyi silme** |
| `nvidia-smi` boş ya da yok | `sudo ubuntu-drivers autoinstall`, Secure Boot'u kapat, yeniden başlat |
| Sürücü iyi ama sonra hash hızı düşük | Sürücü sorunu değil — bkz. [benchmark tablosu](https://github.com/hazennetworksolutions/quantus-mainnet/blob/main/guide.tr.md#adım-a2--kartı-benchmark-et) |
| Makine boşa çıkınca madencilik duruyor | Adım 7'yi grafik oturumu içinde tekrar uygula |
| Kurulumdan sonra Wi-Fi yok | Ethernet kullan, ya da `sudo ubuntu-drivers autoinstall` + yeniden başlat |

Resmî anlatım: [Install Ubuntu Desktop](https://ubuntu.com/tutorials/install-ubuntu-desktop).

---

## Yazar Hakkında

**HazenNetworkSolutions** tarafından hazırlandı.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
