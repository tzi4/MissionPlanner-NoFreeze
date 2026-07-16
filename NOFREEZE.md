# Mission Planner — NoFreeze Fork

Resmî [ArduPilot/MissionPlanner](https://github.com/ArduPilot/MissionPlanner)'ın, **çoklu araç kullanımındaki arayüz donmasını gideren** fork'u.

**Sorun:** Tek UDP portuna bağlı birden çok İHA ile uçarken (örn. 5 drone → 14550), Arm/Takeoff/Set WP gibi komutlar ACK bekleme süresince arayüzü **30–60 saniye tamamen donduruyor** ve bu sırada bağlı TÜM araçların telemetri işlenmesi duruyordu.

**Çözüm (branch: `fix/multi-uav-freeze`):**
- Arm/Disarm, Takeoff ve Set WP düğmeleri komutu artık **asenkron** gönderir: arayüz donmaz, yalnızca ilgili düğme işlem süresince devre dışı kalır, sonuç/hata mesajları aynen gelir.
- `giveComport` sızıntısı kapatıldı: komut beklerken bağlantı hatası oluşursa telemetrinin kalıcı kesilmesine yol açan bug düzeltildi (`try/finally`).
- Timeout süreleri bilinçli olarak değiştirilmedi: ulaşılamayan araca verilen komut yine ~40 sn sonra "No response" der — ama artık donmadan.

Teknik derinlik: [docs-nofreeze/ANALIZ.md](docs-nofreeze/ANALIZ.md) (kök neden analizi, QGC karşılaştırması) ve [docs-nofreeze/YAMA-NOTLARI.md](docs-nofreeze/YAMA-NOTLARI.md) (değişiklik listesi, derleme, test planı).

---

## Kurulum — Linux (derleme GEREKMEZ)

Hazır derlenmiş paket **[Releases](../../releases)** sayfasındadır. Mission Planner, Linux'ta Mono ile çalışır:

```bash
# 1) Mono kur (Ubuntu/Debian; mono >= 6 önerilir)
sudo apt update
sudo apt install -y mono-complete unzip

# 2) Releases sayfasından zip'i indirip aç
unzip MissionPlanner-NoFreeze-*.zip -d ~/MissionPlanner-NoFreeze

# 3) Çalıştır
cd ~/MissionPlanner-NoFreeze
mono MissionPlanner.exe
```

Notlar:
- İlk açılış yavaş olabilir (Mono JIT); ayarlar `~/Documents/Mission Planner/` altında tutulur.
- Bağlantı: sağ üstten **UDP** seçin → Connect → port **14550**. Aynı porta paket atan tüm araçlar tek bağlantıda, sağ üstteki araç seçiciden görünür.
- Video akışı isteğe bağlıdır (`sudo apt install -y gstreamer1.0-tools gstreamer1.0-plugins-good` vb.); GCS işlevleri için gerekmez.
- Sorun olursa terminaldeki konsol çıktısı doğrudan hatayı gösterir.

## Kurulum — Windows

Releases'tan zip'i indirin → bir klasöre açın → `MissionPlanner.exe` çift tık. (Kurulu resmî MP ile **aynı anda çalıştırmayın** — aynı ayar klasörünü paylaşırlar.)

## Güncelleme

Yeni sürüm çıktığında Releases sayfasından yeni zip'i indirip eski klasörün üzerine açmanız yeterli (ayarlarınız `Documents/Mission Planner`'da olduğu için korunur).

## Uyarı

Bu deneysel bir yapıdır. Gerçek uçuş operasyonunda birincil YKİ olarak kullanmadan önce SITL veya pervanesiz bench testinde doğrulayın. Kaynak: master `a2fcd74` (2026-07-11) + yama commit'leri; Debug konfigürasyonuyla derlenmiştir.
