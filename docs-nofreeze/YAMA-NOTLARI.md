> Not: Yerel analiz klasöründen kopyalanmıştır; dosya yolları oradaki düzene göredir.

# `fix/multi-uav-freeze` Yaması — Notlar, Derleme ve Test

**Nerede:** `MissionPlanner/` klonunda `fix/multi-uav-freeze` branch'i (commit `459b73a`, master `a2fcd74` üzerine) + taşınabilir kopya olarak [mp-freeze-fix.patch](mp-freeze-fix.patch).
**Neyi çözüyor:** Çoklu araçta (5 drone, tek UDP 14550) Arm/Takeoff/Set WP basınca 30–60 sn'lik tam UI donması ve `giveComport` sızıntısıyla oluşabilen kalıcı telemetri kesilmesi (ayrıntılı kök neden: [ANALIZ.md](ANALIZ.md)).

---

## 1. Değişiklikler

### A) `ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs` — `giveComport` artık her yoldan bırakılıyor

`doCommandAsync`, `doCommandIntAsync` ve `setWPCurrentAsync` içindeki ACK bekleme döngüleri `try/finally` içine alındı; `giveComport = false` artık `finally`'de. Önceden `readPacketAsync`/`generatePacket` bekleme sırasında istisna fırlatırsa (link kopması vb.) bayrak `true` kalıyor, `MainV2.SerialReader` süresiz duruyor ve **5 aracın telemetrisi yeniden bağlanana kadar kesiliyordu**. Davranış değişikliği yok; yalnızca istisna yolları güvenceye alındı (döngü içindeki eski `giveComport = false` satırları `finally`'ye taşındığı için kaldırıldı).

### B) `GCSViews/FlightData.cs` — 3 handler async'e çevrildi

| Handler | Eski | Yeni |
|---|---|---|
| `BUT_ARM_Click` | `doARM(...)` — UI thread'inde 40 sn'ye kadar bloke | `await doARMAsync(...)`; buton işlem boyunca devre dışı; force-arm akışı ve STATUSTEXT aboneliği aynen korunur (abonelik artık `finally` ile garantili bırakılır) |
| `takeOffToolStripMenuItem_Click` | `doCommand(TAKEOFF)` — 8 sn'ye kadar bloke | `await doCommandAsync(TAKEOFF)` |
| `BUT_setwp_Click` | `setWPCurrent(...)` — 12 sn'ye kadar bloke | `await setWPCurrentAsync(...)`; butonu geri açma `finally`'de |

Desen, MP'nin kendi kodunda zaten kullanılan async handler deseninin birebir aynısıdır (`modifyandSetSpeed_Click`, `setHomeHereToolStripMenuItem_Click` — `await ...Async(...).ConfigureAwait(true)`).

Ek iyileştirme: `BUT_ARM_Click` hedef `sysid/compid`'i **await'ten önce** yakalar — komut beklerken araç seçicisinden başka drone'a geçmek komutun hedefini artık değiştiremez (eskiden force-arm ikinci çağrısı o anki seçime giderdi).

### Yeni davranış (kullanıcı gözünden)

- Arm'a bastınız, ACK kayboldu → **UI donmaz**; diğer 4 aracın HUD'u canlı kalır*; 40 sn sonra aynı "No response" kutusu gelir (timeout süresi değiştirilmedi — davranış korundu, sadece bekleme arka plana alındı).
- Bekleme sırasında Arm butonu gri olur; diğer kontroller kullanılabilir.

\* Not: `giveComport` bekleme sırasında hâlâ set edildiği için SerialReader o link'te durur; ancak ACK arayan arka plan görevi `readPacketAsync`'i sürekli çağırdığından paketler yine parse edilip `MAVlist`/HUD'a dağıtılır — artık UI thread bloke olmadığından ekran bunları çizebilir.

---

## 2. Bilinen sınırlar / bilinçli kapsam dışı

1. ~~Derlenerek doğrulanmadı~~ → **DERLENDİ (2026-07-14):** Klasör içine kurulan taşınabilir .NET SDK 8.0.422 ile, sisteme ve çalışan MP'ye dokunmadan **0 hata** ile derlendi. Hazır çıktı: `MissionPlanner\bin\Debug\net461\MissionPlanner.exe` (~281 MB klasör, 304 dosya). Uyarıların tamamı repo'da önceden var olan stil uyarılarıdır; yamalanan bölgelerden hata/uyarı çıkmadı.
2. **Aynı anda iki komut:** UI artık donmadığı için bekleme sırasında başka komut düğmelerine basılabilir. MP'nin komut katmanı eşzamanlı iki ACK beklemesini desteklemez (`giveComport` çakışması → ikinci komut ACK'ini kaçırıp "failed" diyebilir). Bu, stock MP'deki async handler'larda da var olan bir sınırdır; donma yerine yumuşak hata verir. Pratik kural: bir komutun sonucunu beklemeden aynı linkte ikinci komut vermeyin.
3. **Dönüştürülmeyen bloke handler'lar:** `BUTactiondo_Click` (Actions/Do Action, FlightData.cs:1798–1865 bölgesi), `BUT_resumemis_Click` (busy-wait + DoEvents), `BUTrestartmission_Click`, `BUT_mountmode_Click`. Bunlar hâlâ UI thread'ini bloklar (2–12 sn); (A) düzeltmesi istisna yollarını yine de korur. Aynı desenle dönüştürülebilirler — gelecek iş.

---

## 3.0 Bu makinede yapılan derleme (hazır çıktı)

Derleme, sisteme hiçbir kurulum yapılmadan şu düzenekle tamamlandı (tamamı klasör içinde, tekrar üretilebilir):

- Taşınabilir SDK: `.dotnet\` (dotnet-install.ps1, kanal 8.0 → SDK 8.0.422, `-NoPath` ile sistem PATH'ine dokunulmadı)
- NuGet paketleri: `NUGET_PACKAGES` → `.nuget\packages\` (kullanıcı profiline yazılmadı)
- `MissionPlanner\Directory.Build.targets` (yerel yardımcı, commit'lenmedi): makinede .NET Framework targeting pack olmadığından net472 referans asemblilerini `Microsoft.NETFramework.ReferenceAssemblies` NuGet paketinden çözer
- Sıra önemli: önce `ExtLibs\DriverCleanup\DriverCleanup.csproj` (çıktısı `Drivers\DriverCleanup.exe`; ana csproj bunu Content olarak kopyalar, yoksa MSB3030 verir), sonra `MissionPlanner.csproj`
- Komut: `.dotnet\dotnet.exe build MissionPlanner.csproj -c Debug` → **0 hata**; tam log: `build-main.log` / `build-main2.log`

**Çıktı:** `MissionPlanner\bin\Debug\net461\MissionPlanner.exe`

**Çalıştırmadan önce:** Kurulu/çalışan Mission Planner'ı kapatın — ikisi aynı `Belgeler\Mission Planner` config klasörünü paylaşır, aynı anda açmayın. İlk denemeyi gerçek araç yerine SITL ile yapın (Bölüm 4). Bu exe deneyseldir; saha operasyonunda birincil GCS olarak kullanmadan önce test şarttır.

## 3. Derleme (alternatif: Visual Studio ile, herhangi bir makinede)

1. **Visual Studio 2022 Community** kurun → workload: **".NET desktop development"** (.NET Framework 4.6.1+ targeting pack dahil olmalı; VS Installer → Individual components → ".NET Framework 4.6.1 targeting pack" işaretli değilse ekleyin).
2. `MissionPlanner/` klasöründe `MissionPlanner.sln`'i açın (ilk açılışta NuGet restore internetten paket indirir, birkaç dakika sürebilir).
3. Solution Configuration: `Debug` (veya `Release`), Platform: varsayılan bırakın; Startup project: `MissionPlanner`.
4. Build → yalnızca şu iki proje değişti: `MissionPlanner.ArduPilot` (MAVLinkInterface.cs) ve ana `MissionPlanner` (FlightData.cs). Solution'daki Android/iOS head projeleri derlenmezse sorun değil — masaüstü exe için `MissionPlanner` projesine sağ tık → Build yeterli.
5. Çıktı: `MissionPlanner/bin/Debug/net461/MissionPlanner.exe` (veya csproj'un OutputPath'ine göre `bin/Debug/`).

**Önemli:** Derlenen exe'yi uçuş yönetiminde kullanmadan önce SITL ile test edin (aşağıda). Kurulu MP'nizle aynı config klasörünü (`Belgeler\Mission Planner`) paylaşır — ikisini aynı anda çalıştırmayın.

### Yamayı başka bir çalışma kopyasına uygulamak

```bash
cd <MissionPlanner-kaynak-klasörü>     # master, a2fcd74 civarı
git apply --check "..\mp-freeze-fix.patch"   # önce kuru kontrol
git am "..\mp-freeze-fix.patch"              # commit olarak uygular
```

---

## 4. Test planı

### SITL (donanımsız, güvenli)

1. 5 ArduCopter SITL örneği başlatın, hepsini aynı porta yönlendirin: her örnek için `--out udp:127.0.0.1:14550` (sysid'ler 1–5 olacak şekilde `SYSID_THISMAV` ayarlayın).
2. Yamalı MP → UDP 14550 dinle → 5 araç tek bağlantıda görünmeli.
3. **Donma testi:** Bir SITL sürecini duraklatın (Windows'ta Process Explorer → Suspend) → o araca Arm verin → **beklenen:** UI donmaz, Arm butonu ~40 sn gri kalır, diğer araçların HUD'u akmaya devam eder, sonunda "No response" kutusu gelir. Stock MP'de aynı test 40 sn tam donma verir (fark net görülür).
4. **Normal yol testi:** Canlı araca Arm/Takeoff/Set WP → davranış stock MP ile aynı olmalı (başarı/red mesajları dahil, force-arm dialogu dahil).
5. **Sızıntı testi:** ACK beklerken SITL'i tamamen kapatın (link kopması) → istisna sonrası diğer araçların telemetrisi akmaya devam etmeli (stock MP'de `giveComport` takılıp tüm telemetri kesilebilirdi).

### Saha (SITL testi geçtikten sonra)

Önce 2 araçla düşük riskli deneme; arm-reddi ve zayıf link senaryolarını yerde (pervanesiz) deneyin. 5 araca ondan sonra geçin.
