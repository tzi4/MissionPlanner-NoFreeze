> Not: Bu belge yerel analiz klasöründen kopyalanmıştır; `MissionPlanner/...` ile başlayan satır linkleri bu reponun köküne göre, `qgroundcontrol/...` linkleri ise yerel QGC klonuna göredir.

# Mission Planner vs QGroundControl — Çoklu Drone'da Donma Sorunu: Kök Neden Analizi

**Senaryo:** 5 drone, tek UDP portu (14550), Microhard mesh modem (ortalama iyi ama dalgalı link).
**Belirti:** Mission Planner'da Arm/Takeoff gibi hemen her düğmede 30–60 sn tam donma; QGroundControl'de donma yok, komut ulaşmazsa ~1,5 sn'de net hata.

**İncelenen kod:** `MissionPlanner/` master `a2fcd74` (2026-07-11), `qgroundcontrol/` master `5bc5dfb` (2026-07-10). Aşağıdaki tüm satır numaraları bu klonlardan doğrulanmıştır.

---

## 0. Yönetici Özeti

Fark, UDP veya ağ katmanında değil, **komut bekleme mimarisinde**:

| | Mission Planner | QGroundControl |
|---|---|---|
| Komut gönderimi | UI thread'inde **senkron bekler** (`AwaitSync`) | Asenkron kuyruk (`MavCommandQueue`), çağrı anında döner |
| ARM cevapsız kalırsa | 10 sn timeout × 4 gönderim = **~40 sn UI donması**, sonra hata | 1,2 sn timeout × 1 deneme = **~1,5 sn'de** "Vehicle did not respond" |
| Bekleme sırasında diğer 4 drone | `giveComport` bayrağı ana okuyucuyu durdurur → **tüm araçların telemetri işlenmesi askıda** | Araç başına bağımsız kuyruk/timer; diğerleri **hiç etkilenmez** |
| Hata gösterimi | Modal MessageBox (donma bittikten sonra) | Non-modal QML dialog (anında) |
| UDP'de komut kime gider | Görülen **tüm** endpoint'lere (broadcast) | Görülen **tüm** endpoint'lere (broadcast) — **aynı**, fark burada değil |

Kullanıcının gözlemlediği "30–60 sn donma" birebir koddaki sabitlerden çıkıyor: **ARM = 10 000 ms × (1 ilk + 3 retry) = 40 sn** (bkz. Bölüm 2). QGC'nin "tak diye hata vermesi" de sabitlerden: **1 200 ms timeout, arm için retry yok, 500 ms'lik denetim timer'ı → 1,2–1,7 sn** (bkz. Bölüm 6).

---

## 1. MP'de Bir "Arm" Tıklamasının Anatomisi

Zincir (hepsi doğrulanmış):

1. `BUT_ARM_Click` — WinForms click handler, **UI thread'inde** çalışır → [GCSViews/FlightData.cs:1057](../GCSViews/FlightData.cs:1057)
   ```csharp
   bool ans = MainV2.comPort.doARM(!isitarmed);   // UI thread, senkron
   ```
2. `doARM` → `doARMAsync(...).AwaitSync()` → [ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2629](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2629)
3. `AwaitSync` = `Task.Run(...).GetAwaiter().GetResult()` → **çağıran (UI) thread'i işlem bitene kadar bloklar** → [ExtLibs/Utilities/Extensions.cs:110-118](../ExtLibs/Utilities/Extensions.cs:110)
4. `doCommandAsync` içinde `giveComport = true` → [MAVLinkInterface.cs:2716](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2716) — bkz. Bölüm 3.
5. ACK bekleme: `while(true)` döngüsü + `readPacketAsync()` inline çağrısı → [MAVLinkInterface.cs:2775-2800](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2775)
6. Cevap yoksa: timeout dolunca 3 kez yeniden gönderir, sonra `TimeoutException` → [MAVLinkInterface.cs:2786-2797](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2786)
7. Exception, `BUT_ARM_Click`'teki `catch`'e düşer → **donma bittikten sonra** "No response" MessageBox'ı ([FlightData.cs:1076-1079](../GCSViews/FlightData.cs:1076)).

WinForms'ta tek UI thread vardır: bu 40 sn boyunca pencere çizilemez, hiçbir tık işlenmez. **Donma sırasında yapılan her tık kuyruklanır ve donma bitince sırayla patlar** — çoğu da yine bloke bir handler başlatır. Kullanıcının "hangi tuşa bassam kasıyor" algısının mekanik açıklaması budur.

---

## 2. Süre Aritmetiği — "30–60 sn" Nereden Geliyor?

`doCommandAsync` sabitleri ([MAVLinkInterface.cs:2729-2768](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2729)):

```csharp
int retrys = 3;
int timeout = 2000;                                       // varsayılan
...
else if (actionid == MAV_CMD.COMPONENT_ARM_DISARM)
{
    // 10 seconds as may need an imu calib
    timeout = 10000;                                      // ARM'a özel
}
```

Retry döngüsü ([MAVLinkInterface.cs:2784-2797](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2784)): timeout dolunca `confirmation++` ile paketi tekrar gönderir, `start`'ı sıfırlar, `retrys--`; retry kalmayınca `TimeoutException` fırlatır.

| Komut | Timeout | Deneme | **En kötü UI donması** |
|---|---|---|---|
| ARM / DISARM | 10 000 ms | 1+3 | **40 sn** |
| ARM sonrası "Force Arm" onaylanırsa | 10 000 ms | 1+3 | **+40 sn** (arada dialog) |
| TAKEOFF (özel durum değil → varsayılan) | 2 000 ms | 1+3 | **8 sn** |
| `setWPCurrent` (mission restart/setwp) | 2 000 ms | 1+5 | ~12 sn |
| `setParam` (her parametre yazımı) | 700 ms | 1+3 | ~2,8 sn |
| `BUT_resumemis_Click` (busy-wait döngüleri + `Application.DoEvents`) | — | — | 30+ sn |

Dalgalı bir linkte tek bir ACK datagram'ının kaybolması yeterli: kullanıcı Arm'a basar → 40 sn donma. Donma sırasında basılan Takeoff kuyruklanır → +8 sn... Gözlemlenen 30–60 sn pencere, tek ARM timeout'u (40 sn) ± kuyruklanan diğer bloklarla birebir örtüşür.

Not: Mod değiştirme (`setMode`) aslında **bloke değildir** (ACK beklemez, `requireack=false` — [MAVLinkInterface.cs:4636](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:4636)); mod düğmelerinde hissedilen kasma, o sırada zaten donmuş UI'dan ya da devam eden başka bir bloke komuttan gelir.

---

## 3. `giveComport` — Tek Komut, 5 Aracın Telemetrisini Durduruyor

MP'de bir link'in tek arka plan okuyucusu vardır: `MainV2.SerialReader`. Bir komut ACK beklerken `giveComport=true` yapılır ve okuyucu **tamamen devre dışı kalır**:

[MainV2.cs:3008](../MainV2.cs:3008) ve [MainV2.cs:3046-3047](../MainV2.cs:3046):
```csharp
// if not connected or busy, sleep and loop
if (!comPort.BaseStream.IsOpen || comPort.giveComport == true) { ... await Task.Delay(100)... }
...
while (port.BaseStream.IsOpen && port.BaseStream.BytesToRead > minbytes &&
       port.giveComport == false && serialThread && ...)
{
    await port.readPacketAsync()...
}
```

Tek UDP portundaki 5 drone, MP içinde **tek `MAVLinkInterface`'in** `MAVlist` tablosunda (sysid→`MAVState`, [MAVList.cs:10](../ExtLibs/ArduPilot/Mavlink/MAVList.cs:10)) yaşar. `giveComport=true` olduğunda:

- SerialReader hiçbir paket çekmez → **5 aracın HUD/telemetri işlenmesi durur**;
- Tek okuyucu, bloke `doCommandAsync` döngüsünün kendi `readPacketAsync`'i olur ([MAVLinkInterface.cs:2800](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2800)): 5 aracın tüm trafiğini tek tek parse edip aradığı tek ACK'i bekler;
- UI thread de bloke olduğundan ekran zaten çizilmez.

Yani mimari olarak: **bir drone'a verilen tek bir cevapsız komut, hem tüm pencereyi hem tüm filonun veri akışını 40 sn rehin alır.**

---

## 4. Yan Bug — `giveComport` Sızıntısı (kalıcı telemetri kesilmesi riski)

`doCommandAsync`'in ACK bekleme döngüsünü saran bir `try/finally` **yoktur**. Timeout yolu bayrağı temizler ([2796-2797](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2796)), ama `readPacketAsync()` (satır 2800) bir istisna fırlatırsa (ör. akış hatası, bağlantı kesilmesi) istisna `giveComport=true` bırakılarak yukarı kaçar. Sonuç: SerialReader süresiz durur → **5 aracın telemetrisi yeniden bağlanana kadar tamamen kesilir**. Aynı desen `doCommandIntAsync`, `setWPCurrentAsync`, `setParamAsync` gibi kardeş metodlarda da var. Arada yaşanan "40 sn'den de uzun, hiç düzelmeyen" takılmaların olası açıklaması budur. (Hazırlanan yama bunu düzeltir — Bölüm 11.)

---

## 5. Karşı Kıyı: QGC'de Aynı Tıklamanın Anatomisi

QGC master'da komut mantığı `MavCommandQueue` sınıfındadır (Vehicle'dan delegasyon, [src/Vehicle/Vehicle.cc:2144](qgroundcontrol/src/Vehicle/Vehicle.cc:2144)):

1. QML arm butonu → `Vehicle::setArmed(true, showError=true)` → `sendMavCommand(MAV_CMD_COMPONENT_ARM_DISARM...)` ([Vehicle.cc:1437](qgroundcontrol/src/Vehicle/Vehicle.cc:1437)) → kuyruk girişi (`MavCommandListEntry_t {maxTries, ackTimeoutMSecs, QElapsedTimer...}`, [MavCommandQueue.h:76](qgroundcontrol/src/Vehicle/MavCommandQueue.h:76)). **Çağrı anında döner; UI serbest.**
2. Serbest çalışan 500 ms'lik `QTimer` (`_responseCheckTimer`, [MavCommandQueue.cc:24](qgroundcontrol/src/Vehicle/MavCommandQueue.cc:24)) her tick'te `elapsed > ackTimeoutMSecs` (gerçek değer **1 200 ms**, [MavCommandQueue.cc:188](qgroundcontrol/src/Vehicle/MavCommandQueue.cc:188)) olan girişleri işler.
3. **ARM için retry yoktur** — `_shouldRetry(command)` sadece durum-sorgu komutlarında true döner; gerekçe koddaki yorumda açık: *arm komutunun 6 sn sonra aniden işlemesi istenmez* ([MavCommandQueue.cc:200](qgroundcontrol/src/Vehicle/MavCommandQueue.cc:200)). `maxTries = 1`.
4. Deneme hakkı bitince: `commandResult(MAV_RESULT_FAILED, NoResponseToCommand)` sinyali + `QGC::showAppMessage(tr("Vehicle did not respond to command: %1")...)` ([MavCommandQueue.cc:348](qgroundcontrol/src/Vehicle/MavCommandQueue.cc:348)) → **non-modal** QML dialog ([QGCApplication.cc:431](qgroundcontrol/src/QGCApplication.cc:431)).

**Zaman çizelgesi (ACK hiç gelmezse):** t=0 gönder → t=500/1000 ms tick'lerde henüz eşik aşılmadı → t≈1200–1700 ms'deki ilk tick'te vazgeç + hata dialogu. **Kullanıcı ~1,5 sn'de hatayı görür; UI hiçbir an bloke olmaz.** Kullanıcının "arada komut alamıyor ama hatayı anında basıyor" gözlemi birebir bu koddur.

Threading ve izolasyon:

- UDP soket IO ayrı `QThread`'deki `UDPWorker`'da; main thread'e yalnızca `Qt::QueuedConnection` ile veri geçer ([UDPLink.cc:509-527](qgroundcontrol/src/Comms/UDPLink.cc:509)); gönderim de main→worker'a queued'dur ([UDPLink.cc:594](qgroundcontrol/src/Comms/UDPLink.cc:594)) — **komut yolunda hiçbir senkron soket beklemesi yok**.
- Her sysid için ayrı `Vehicle` nesnesi yaratılır (heartbeat → `MultiVehicleManager`, [MultiVehicleManager.cc:103](qgroundcontrol/src/Vehicle/MultiVehicleManager.cc:103)); her Vehicle kendi sysid'si dışındaki mesajları anında eler ([Vehicle.cc:519](qgroundcontrol/src/Vehicle/Vehicle.cc:519)); komut kuyruğu ve comm-lost tespiti (3,5 sn, [VehicleLinkManager.h:84](qgroundcontrol/src/Vehicle/VehicleLinkManager.h:84)) **araç başınadır**. Bir dronun cevapsız komutu diğer 4'ünün telemetrisini matematiksel olarak etkileyemez.

---

## 6. UDP Hedefleme: Fark Burada DEĞİL (yaygın yanlış şüphenin elenmesi)

Her iki program da tek dinleme soketinde gördüğü **tüm** kaynak endpoint'leri kaydeder ve giden her paketi **hepsine** yollar; araç seçimi MAVLink `target_system` alanıyla uçakta yapılır:

- MP: `EndPointList`'e her yeni gönderen eklenir ([CommsUdpSerial.cs:201-202](../ExtLibs/Comms/CommsUdpSerial.cs:201)); `Write` listedeki herkese gönderir ([CommsUdpSerial.cs:264-276](../ExtLibs/Comms/CommsUdpSerial.cs:264)).
- QGC: `_sessionTargets` aynı şekilde birikir ([UDPLink.cc:472](qgroundcontrol/src/Comms/UDPLink.cc:472)); `writeData` yapılandırılmış hedefler + tüm session target'lara gönderir ([UDPLink.cc:390](qgroundcontrol/src/Comms/UDPLink.cc:390)).

Yani "MP komutu yanlış drone'a atıyor" hipotezi **doğrulanmadı** — komut her iki programda da 5 drone'a fiziksel olarak ulaşır. Donma farkının %100'ü bekleme/threading mimarisindedir. (Ortak yan etki: 5 araçta giden her GCS paketi 5 datagram'a kopyalanır — uplink yükü; iki programda da aynı.)

MP'nin ACK eşleştirmesi de doğrudur (sysid+compid+command kontrolü, [MAVLinkInterface.cs:2803-2813](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2803)) — sorun eşleştirme değil, eşleşme gelmediğindeki bekleme modelidir.

---

## 7. Yan Yana: "Arm'a bastım, ACK kayboldu" (5 drone, tek port)

| t | Mission Planner | QGroundControl |
|---|---|---|
| 0 sn | ARM gönderilir; UI thread `GetResult()`'ta bloke; `giveComport=true` → 5 aracın telemetri işlenmesi durur | ARM gönderilir; çağrı döner; UI akıcı, 5 araç canlı |
| 0–10 sn | Pencere tamamen donuk; tıklar kuyruklanıyor | 1,2–1,7 sn: **"Vehicle did not respond to command: Arm"** non-modal dialog; operatör tekrar dener |
| 10/20/30 sn | Sessiz yeniden gönderimler (retry 3→0) | — |
| 40 sn | `TimeoutException` → UI çözülür → modal "No response" → kuyruklanmış tıklar sırayla işlenir (çoğu yeni bloklar başlatır) | — |

---

## 8. MP Handler Envanteri — hangi düğme ne kadar bloklar

Doğrulanan, UI thread'inde senkron comm çağrısı yapan handler'lar ([GCSViews/FlightData.cs](../GCSViews/FlightData.cs)):

| Handler | Satır | Çağrı | En kötü blok |
|---|---|---|---|
| `BUT_ARM_Click` | 1057, 1068 | `doARM` (+force) | 40 sn (+40) |
| `takeOffToolStripMenuItem_Click` | 5305-5310 | `setMode`+`doCommand(TAKEOFF)` | ~8 sn |
| `BUT_setwp_Click` | 1663 | `setWPCurrent` | ~12 sn |
| `BUTrestartmission_Click` | 1889 | `setWPCurrent` | ~12 sn |
| `BUTactiondo_Click` (Actions/Do Action) | 1798-1865 | `doCommand`/`doReboot`/`doEngineControl`/... | 2–8 sn |
| `BUT_mountmode_Click` | 1398-1404 | `setParam`/`doCommand` | ~3 sn |
| `BUT_resumemis_Click` | 1557-1612 | `setMode`+`doARM`+`doCommand` busy-wait + `DoEvents` | 30+ sn |

MP kod tabanında doğru desen zaten mevcut ama yalnızca 2 handler'da kullanılmış: `modifyandSetSpeed_Click:4428` ve `setHomeHereToolStripMenuItem_Click:4854` (`await doCommandAsync(...).ConfigureAwait(true)`) — yama bu deseni arm/takeoff/setwp'ye yaygınlaştırır.

Çapraz doğrulama: ArduPilot/MissionPlanner issue [#2784](https://github.com/ArduPilot/MissionPlanner/issues/2784) "extremely slow param download with multiple vehicles" — aynı paylaşımlı-link/`giveComport` tekelleşmesinin parametre indirme yüzü.

---

## 9. Kod Değiştirmeden Pratik Öneriler (dürüst etki değerlendirmesiyle)

1. **Drone başına ayrı port (14551–14555) + MP'de 5 ayrı UDPCl bağlantısı.**
   Microhard tarafında her aracın telemetrisini farklı hedef porta yönlendirin; MP'de her portu ayrı bağlantı olarak açın (Ctrl ile çoklu bağlantı). Kazanım: `giveComport` **link başına** olduğundan, bir drone'a komut beklerken diğer 4'ünün telemetri **işlenmesi** devam eder; buffer şişmesi ve veri kaybı azalır. Sınır: UI thread bloğu aynen kalır — komut verilen ekran yine donar. **Donmayı hafifletir, çözmez.**
2. **Telemetri hızlarını düşürün** (her araçta `SR0_*`/`SR1_*` parametreleri): 5 aracın toplam trafiği azalır → Microhard'da ACK kaybı olasılığı düşer → 40 sn'lik yolun tetiklenme sıklığı azalır.
3. **Çoklu araç komutlarını QGC'den verin** (mevcut pratiğiniz): mimari olarak doğru davranış QGC'de. MP'yi planlama/param/log için tek araçla kullanmak en risksiz iş bölümü.
4. **Force Arm dialoglarına dikkat:** MP'de arm reddedilirse çıkan "Force Arm" onayı ikinci bir 40 sn'lik blok başlatabilir; dalgalı linkte "Cancel" diyip tekrar denemek çoğu zaman daha hızlıdır.

**Kalıcı çözüm kod tarafındadır** → Bölüm 11'deki yama.

---

## 10. SITL ile Reprodüksiyon (uçuş bittikten sonra, ayrı makinede/ağda önerilir)

1. 5 ArduCopter SITL örneğini aynı GCS portuna yönlendirin (her biri `--out 127.0.0.1:14550`).
2. MP master ile bağlanın (UDP 14550): 5 sysid tek bağlantıda görünür.
3. ACK kaybını simüle edin: bir SITL sürecini `SIGSTOP`/duraklatın ya da güvenlik duvarında o kaynağın giden paketlerini düşürün.
4. Arm'a basın → MP ~40 sn donar (bu analizin doğrulaması). Aynı senaryoda QGC ~1,5 sn'de hata basar.
5. Yamalı MP ile tekrarlayın → UI donmamalı; sonuç mesajı işlem bitince gelmeli.

---

## 11. Deneysel Yama — `fix/multi-uav-freeze`

`MissionPlanner/` klonunda branch olarak + kök klasörde `mp-freeze-fix.patch` dosyası olarak mevcuttur. Kapsam ve gerekçeler `YAMA-NOTLARI.md`'dedir (derleme talimatı dahil). Özet:

1. **`MAVLinkInterface.cs` — `doCommandAsync`, `doCommandIntAsync` ve `setWPCurrentAsync` ACK döngüleri `try/finally` içine alındı:** istisna hangi yoldan kaçarsa kaçsın `giveComport` temizlenir → "kalıcı telemetri kesilmesi" bug'ı kapanır (Bölüm 4).
2. **`FlightData.cs` — `BUT_ARM_Click` async'e çevrildi:** `doARMAsync` + `await`; işlem sürerken buton devre dışı; force-arm akışı ve STATUSTEXT aboneliği korunur. UI artık donmaz; sonuç/hata aynı dialoglarla işlem bitince gösterilir.
3. **`FlightData.cs` — `takeOffToolStripMenuItem_Click` async'e çevrildi:** `doCommandAsync(TAKEOFF)` + `await`.
4. **`FlightData.cs` — `BUT_setwp_Click` async'e çevrildi:** `setWPCurrentAsync` + `await`; buton devre dışı bırakma `finally`'ye taşındı.

Bilinçli kapsam dışı (gelecek iş, satırlar Bölüm 8'de): `BUTactiondo_Click` (200 satırlık çok dallı handler), `BUT_resumemis_Click` (DoEvents'li busy-wait akışının yeniden tasarımı gerekir), `BUT_mountmode_Click`, `BUTrestartmission_Click`. Yama bunlara dokunmaz; ancak (1) numaralı `finally` düzeltmesi bunların da istisna yollarını güvenceye alır.

**Derleme durumu:** Yama, klasör içine kurulan taşınabilir .NET SDK ile (sisteme ve çalışan MP'ye dokunmadan) **0 hata ile derlendi**; hazır çıktı `MissionPlanner\bin\Debug\net461\MissionPlanner.exe`. Ayrıntı ve güvenli çalıştırma notları `YAMA-NOTLARI.md`'de. Henüz çalıştırılarak test edilmedi — önce SITL önerilir.
