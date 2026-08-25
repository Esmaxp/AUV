# MAVLink · MAVSDK · MAVROS — Kapsamlı Açıklayıcı Not

**Hiç bilmeyen birinin de anlayacağı şekilde hazırlanmıştır**

---

## 0. Bu Üçü Neden Bir Arada Duyuyoruz?

Su altı/hava aracınızın beyni olan **Pixhawk** (uçuş/otopilot kartı) ile aracı kontrol etmek istediğiniz **bilgisayar** (Jetson Nano, Raspberry Pi, laptop, QGroundControl uygulaması vb.) arasında bir "konuşma" olması lazım. Bu konuşmayı mümkün kılan üç katman var:

| Katman | Ne olduğu, tek cümlede |
|---|---|
| **MAVLink** | Aracın ve bilgisayarın **konuştuğu ortak dil** (protokol/mesaj formatı) |
| **MAVSDK** | Bu dili sizin yerinize konuşan **hazır kütüphane** — "arm et", "git" gibi hazır fonksiyonlar sunar |
| **MAVROS** | Bu dili **ROS** (Robot Operating System) diline çeviren **tercüman/köprü** |

Basit bir benzetme: **MAVLink** = Fransızca gibi bir dil. **MAVSDK** = "Ben bu dili biliyorum, sana kullanıma hazır cümle kalıpları veririm" diyen bir konuşma kitabı. **MAVROS** = Fransızca konuşan biriyle, siz Türkçe konuşurken aranızda çeviri yapan simultane tercüman (ve o tercüman ROS ekosistemindeki herkesle de konuşabiliyor).

Üçü de **rakip değil, farklı seviyelerde çalışan, birbirini tamamlayan araçlardır.**

---

## 1. MAVLink Nedir?

**MAVLink (Micro Air Vehicle Link)**, 2009 yılında **Lorenz Meier** tarafından ETH Zürih'te (İsviçre) geliştirilmiş, açık kaynaklı (LGPL lisanslı) bir **iletişim protokolüdür**. Bugün PX4, ArduPilot/ArduSub, Pixhawk, QGroundControl gibi neredeyse tüm açık kaynaklı otopilot ve yer istasyonu yazılımlarının **ortak/standart dili** haline gelmiştir.

### 1.1 Ne İşe Yarar?

- Araç (drone, ROV, rover) ile yer istasyonu (QGroundControl gibi) veya companion computer (Jetson, Raspberry Pi) arasında **telemetri** gönderir: konum, hız, derinlik/irtifa, batarya seviyesi, sensör verileri...
- Yer istasyonundan araca **komut** gönderir: arm et (motorları etkinleştir), git, dur, mod değiştir, görev yükle...
- Bu bilgiyi **çok küçük, hafif paketler** halinde gönderir — düşük bant genişlikli, gürültülü kablosuz bağlantılarda bile çalışacak şekilde tasarlanmıştır.

### 1.2 Nasıl Çalışır? (Basitleştirilmiş)

MAVLink, verileri **ikili (binary) paketler** halinde gönderir. Her paket kabaca şu bilgileri taşır:

```
[Başlangıç Baytı] [Uzunluk] [Sıra No] [Sistem ID] [Bileşen ID] [Mesaj ID] [Veri (Payload)] [Checksum]
```

- **Sistem ID / Bileşen ID:** Mesajı kimin gönderdiğini belirtir (birden fazla araç/bileşen aynı ağda olabilir).
- **Mesaj ID:** Bu paketin "ne anlama geldiğini" belirtir (örn. `HEARTBEAT`, `ATTITUDE`, `GLOBAL_POSITION_INT`).
- **Checksum (CRC):** Verinin bozulup bozulmadığını kontrol eder.

### 1.3 İki Çalışma Modeli

MAVLink, iki farklı iletişim tarzını birleştirir:

1. **Publish-Subscribe (Yayın-Abone) modeli:** Telemetri verileri (konum, hız, tutum/attitude gibi) düzenli aralıklarla "yayınlanır"; onay (ACK) beklenmez. Yüksek frekanslı veri akışı için idealdir.
2. **Point-to-Point (Noktadan Noktaya) modeli:** Görev yükleme, parametre değiştirme gibi kritik işlemler onay bekler, paket kaybolursa tekrar gönderilir. Güvenilirlik önemlidir.

### 1.4 HEARTBEAT — Kalp Atışı Mesajı

Her MAVLink cihazı (araç, yer istasyonu, companion computer) **saniyede en az 1 kez** `HEARTBEAT` mesajı göndermek zorundadır. Eğer otopilot bir süre heartbeat alamazsa, **failsafe** (güvenli mod) devreye girer — örneğin motorları durdurabilir. Bu, "karşı taraf hâlâ orada mı, bağlantı kopmuş mu?" sorusunun cevabıdır.

### 1.5 MAVLink 1 vs MAVLink 2

| | MAVLink 1 | MAVLink 2 |
|---|---|---|
| Paket başı ek yük (overhead) | ~8 bayt | ~14 bayt |
| Maksimum mesaj ID sayısı | 256 | 16 milyondan fazla |
| Mesaj imzalama (güvenlik) | Yok | Var (SHA-256 tabanlı, opsiyonel) |
| Geriye dönük uyumluluk | — | MAVLink 1 cihazlarla uyumlu çalışabilir |

**Yeni projeler için doğrudan MAVLink 2 kullanılması önerilir.**

### 1.6 Önemli Not: Şifreleme Yok!

MAVLink verileri **varsayılan olarak şifrelenmez.** İsteğe bağlı "mesaj imzalama" (signing) özelliği, mesajın kimden geldiğini doğrulamaya yardımcı olur ama içeriği gizlemez. Bu, MAVLink'in bilinen bir güvenlik zafiyetidir; üretim/gerçek uçuş sistemlerinde ek güvenlik önlemleri (imzalama etkinleştirme, ağ izolasyonu) önerilir.

---

## 2. MAVSDK Nedir?

**MAVSDK**, MAVLink protokolünü sizin adınıza "konuşan" bir **API / kütüphanedir**. C++, Python, Swift, Java gibi dillerde kullanılabilir.

### 2.1 Neden Var? (MAVLink'i Doğrudan Kullanmanın Zorluğu)

MAVLink'i "çıplak" haliyle kullanmak isterseniz, kendi başınıza:
- İkili paketleri elle oluşturmanız/parse etmeniz,
- Hangi mesaj ID'sinin ne anlama geldiğini ezbere bilmeniz,
- Heartbeat, timeout, yeniden gönderme gibi mantıkları kendiniz yazmanız

gerekir. Bu, basit bir görev için bile epey can sıkıcı olabilir.

### 2.2 MAVSDK Bunu Nasıl Kolaylaştırır?

MAVSDK, bu karmaşıklığı sizin yerinize halleder ve size hazır, okunaklı fonksiyonlar sunar. Örneğin Python'da:

```python
from mavsdk import System
import asyncio

async def run():
    drone = System()
    await drone.connect(system_address="udp://:14540")

    await drone.action.arm()          # Motorları etkinleştir
    await drone.action.takeoff()      # Kalk (hava aracı için)

    async for position in drone.telemetry.position():
        print(f"İrtifa: {position.relative_altitude_m} m")

asyncio.run(run())
```

Görüldüğü gibi, arka planda hangi MAVLink mesajlarının gönderildiğini bilmenize gerek kalmadan `arm()`, `takeoff()`, `telemetry.position()` gibi anlamlı fonksiyonlarla çalışıyorsunuz.

### 2.3 MAVSDK'nin Güçlü ve Zayıf Yönleri

**Güçlü yönleri:**
- Öğrenmesi ve kullanması **hızlı ve kolay**
- ROS bilmenize gerek yok — bağımsız bir Python/C++ scripti olarak çalışabilir
- Özellikle küçük/orta ölçekli, tek görevli otonom uygulamalar için ideal (görüntü işleme + basit hareket komutu gibi)

**Zayıf yönleri:**
- MAVROS'a göre **daha sınırlı** — her MAVLink mesaj tipine veya her senaryoya native destek sunmayabilir
- Büyük, çok sensörlü/çok modüllü sistemlerde (ör. birden fazla kamera, LiDAR, harita oluşturma, path planning gibi bileşenlerin birbiriyle konuşması gereken projelerde) tek başına yeterli bir mimari sağlamaz

---

## 3. MAVROS Nedir?

**MAVROS**, MAVLink ile **ROS (Robot Operating System)** arasında **köprü/çevirmen** görevi gören bir ROS paketidir.

### 3.1 ROS Nedir? (Çok Kısa)

ROS, robotik projelerinde farklı yazılım parçalarının ("node" denen küçük programların) birbiriyle **konu (topic) yayınlayarak/dinleyerek** haberleştiği bir çatı/altyapıdır. Kamera görüntüsü işleyen bir node, yol planlayan başka bir node, motor komutlarını gönderen üçüncü bir node — hepsi ROS üzerinden birbirine bağlanabilir.

### 3.2 MAVROS Ne Yapar?

MAVROS, Pixhawk'tan gelen MAVLink mesajlarını otomatik olarak **ROS topic'lerine** çevirir (ve tam tersi — ROS'tan gelen komutları MAVLink'e çevirir). Örneğin:

- `GLOBAL_POSITION_INT` MAVLink mesajı → `/mavros/global_position/global` ROS topic'i
- ROS üzerinden `/mavros/setpoint_velocity/cmd_vel` topic'ine yazdığınız hız komutu → MAVLink `SET_POSITION_TARGET` mesajına çevrilip araca gönderilir

Böylece, aracınızı kontrol eden yazılımı **tüm ROS ekosistemiyle** (görüntü işleme, sensör füzyonu, SLAM, path planning, rviz görselleştirme vb.) doğal bir şekilde entegre edebilirsiniz.

### 3.3 MAVROS'un Güçlü ve Zayıf Yönleri

**Güçlü yönleri:**
- Çok sayıda sensör/modülün bir arada, birbiriyle konuşarak çalışması gereken **karmaşık sistemler** için en uygun seçenek
- ROS'un devasa ekosistemine (hazır algoritmalar, görselleştirme araçları, simülasyon ortamları — Gazebo gibi) doğrudan erişim
- TEKNOFEST gibi yarışmalarda görüntü işleme + navigasyon + görev mantığının bir arada çalışması gerektiğinde doğal bir mimari sağlar

**Zayıf yönleri:**
- Önce **ROS öğrenmeniz** gerekir — bu ayrı bir öğrenme eğrisi demek
- Kurulumu ve yapılandırması MAVSDK'ye göre biraz daha karmaşık

### 3.4 Önemli Not: ROS1 mi ROS2 mi?

- **ROS1** için: `mavros` paketi kullanılır (bu, geleneksel ve en yaygın kullanılan yöntemdir).
- **ROS2** için: `mavros` paketinin ROS2 portu mevcuttur ve **ArduPilot/ArduSub tabanlı** araçlarda (yani bizim Pixhawk+ArduSub senaryomuzda) hâlâ MAVROS kullanılır.
- Not: **PX4** tabanlı (ArduPilot değil, PX4 firmware'i kullanan) araçlarda ROS2 dünyasında MAVROS yerine daha çok `px4_ros_com` / uXRCE-DDS köprüsü tercih ediliyor — ama bu bizim durumumuz değil, çünkü su altı araçlarında (ArduSub) **MAVROS kullanılmaya devam ediyor.**

---

## 4. Üçünü Bir Arada Görselleştirme

```
┌─────────────────┐        MAVLink Protokolü        ┌──────────────────────┐
│                  │ ◄──────────────────────────────► │                      │
│  Pixhawk         │      (UDP / Seri Port üzerinden)  │  Companion Computer  │
│  (ArduSub)       │                                    │  (Jetson / RPi)      │
│                  │                                    │                      │
└─────────────────┘                                    └──────────┬───────────┘
                                                                    │
                                                     ┌──────────────┴───────────────┐
                                                     │                               │
                                              ┌──────▼───────┐               ┌───────▼────────┐
                                              │   MAVSDK      │               │    MAVROS       │
                                              │  (Python/C++  │               │  (ROS/ROS2      │
                                              │   scripti)    │               │   köprüsü)      │
                                              └───────────────┘               └────────┬────────┘
                                                                                        │
                                                                              ┌─────────▼─────────┐
                                                                              │   ROS Ekosistemi   │
                                                                              │ (görüntü işleme,   │
                                                                              │  path planning,    │
                                                                              │  sensör füzyonu…)  │
                                                                              └────────────────────┘
```

**Özetle:** MAVLink, aracın "konuştuğu dil"dir. Bu dili companion computer üzerinde ya **doğrudan MAVSDK** ile (basit, tek script) ya da **MAVROS üzerinden ROS'a çevirerek** (karmaşık, çok modüllü sistem) kullanabilirsiniz.

---

## 5. Karşılaştırma Tablosu

| Özellik | MAVLink | MAVSDK | MAVROS |
|---|---|---|---|
| **Ne olduğu** | İletişim protokolü (dil) | Protokolü kullanan hazır kütüphane | Protokolü ROS'a çeviren köprü |
| **Seviyesi** | En alt katman (ham veri) | Orta katman (hazır fonksiyonlar) | Orta-üst katman (ROS entegrasyonu) |
| **ROS bilgisi gerekli mi** | Hayır | Hayır | **Evet** |
| **Öğrenme eğrisi** | Orta (protokolü anlamak gerekir) | **Kolay** | Zor (önce ROS, sonra MAVROS) |
| **En uygun olduğu senaryo** | Kendi özel entegrasyonunuzu sıfırdan yazacaksanız | Basit, bağımsız otonom görevler | Çok sensörlü/çok modüllü karmaşık sistemler |
| **Diller** | C, C++, Python, Java, Rust, vb. (üretilen kütüphaneler) | Python, C++, Swift, Java | Python (rospy/rclpy), C++ |
| **Bizim proje için uygunluk** | Her ikisinin de temeli — anlaşılması şart | Küçük bir görev/prototip için hızlı çözüm | İleri Kategori'nin çok modüllü otonomi ihtiyacı için genelde tercih edilen yol |

---

## 6. Bizim Proje (TEKNOFEST İleri Kategori) ile İlişkisi

Su altı otonom aracımızda:

1. **Pixhawk** (ArduSub firmware'i ile) motor/itici kontrolü, IMU, basınç sensörü gibi donanımları yönetiyor ve bunlarla **MAVLink** üzerinden konuşuyor.
2. Üzerimizdeki **companion computer** (görüntü işleme yapan Jetson/Raspberry Pi gibi bir kart), Pixhawk ile MAVLink üzerinden haberleşiyor.
3. Bu haberleşmeyi kurarken iki yoldan biri seçilir:
   - **MAVSDK ile doğrudan Python/C++ scripti** yazıp basit komutlar gönderme (örn: "ileri git", "derinliği koru") — görevler basitse ve tek bir script yeterliyse pratik.
   - **ROS/ROS2 + MAVROS** kullanarak, kamera görüntü işleme node'u, navigasyon node'u, görev mantığı node'u gibi birden fazla parçayı birbirine bağlama — İleri Kategori'nin istediği "tüm görüntü işleme onboard, tam otonom karar verme" senaryosu için daha ölçeklenebilir ve organize bir mimari sunar.

**Genel eğilim:** TEKNOFEST su altı/hava araçlarında, görev karmaşıklaştıkça (görüntü işleme + navigasyon + görev mantığı bir arada çalışacaksa) takımların çoğu **ROS2 + MAVROS** mimarisine yöneliyor. Basit bir prototip veya tek işlevli bir modül için ise **MAVSDK** daha hızlı bir başlangıç noktası olabilir.

---

## 7. Hangisini Ne Zaman Kullanmalıyım? (Karar Rehberi)

| Durumunuz | Öneri |
|---|---|
| ROS bilmiyorum, hızlıca basit bir otonom hareket scripti yazmam lazım | **MAVSDK** |
| Kamera + navigasyon + görev mantığı gibi birden fazla modülü birbirine bağlamam gerekiyor | **MAVROS (ROS2 ile)** |
| Zaten ROS/ROS2 biliyorum veya ekibimde bilen var | **MAVROS** |
| Sadece telemetri okuyup basit bir arayüzde göstereceğim | **MAVSDK** (veya doğrudan `pymavlink`) |
| Yarışma değerlendirmesinde "yenilikçi/ölçeklenebilir yazılım mimarisi" önemli | **MAVROS (ROS2)** — jüri ve kritik tasarım raporu açısından daha "profesyonel" bir mimari izlenimi verir |

---

## 8. Sık Karışan Bir Nokta: pymavlink Nedir?

Bahsi geçen üç araca ek olarak, bir de **pymavlink** vardır — bu, MAVLink mesajlarını Python'da doğrudan okuyup yazmanızı sağlayan, MAVSDK'den daha "ham" (low-level) bir kütüphanedir. MAVSDK'nin sunduğu kolaylık katmanı pymavlink'te yoktur; mesajları elle oluşturup göndermeniz gerekir. ArduSub dokümantasyonunda örnek scriptler pymavlink ile de veriliyor. Özetle seviye sıralaması şöyledir:

```
pymavlink (en ham, en esnek)  <  MAVSDK (hazır fonksiyonlar)  <  MAVROS (tam ROS entegrasyonu)
```

---

## 9. Özet — Tek Paragrafta

**MAVLink**, aracınızla bilgisayarınızın konuştuğu ortak dildir ve her ikisini kullanmak için de anlaşılması şarttır. **MAVSDK**, bu dili sizin yerinize hazır fonksiyonlarla (arm et, git, telemetri oku) konuşan, öğrenmesi kolay bir kütüphanedir — basit, tek script'lik otonom görevler için idealdir. **MAVROS** ise bu dili ROS'a çeviren bir köprüdür; ROS bilmeniz gerekir ama karşılığında görüntü işleme, navigasyon ve görev mantığı gibi birçok modülü organize bir şekilde bir arada çalıştırma imkânı sunar — TEKNOFEST İleri Kategori gibi çok modüllü, tam otonom sistemler için genelde tercih edilen yoldur.

---

## 10. Kaynakça

- MAVLink Geliştirici Rehberi (resmi): https://mavlink.io/en/
- MAVLink Protokol Genel Bakış: https://mavlink.io/en/about/overview.html
- MAVLink GitHub Organizasyonu: https://github.com/mavlink
- ArduPilot MAVLink Temelleri: https://ardupilot.org/dev/docs/mavlink-basics.html
- PX4 MAVLink Mesajlaşma Rehberi: https://docs.px4.io/main/en/mavlink/index
- MAVSDK Resmi Sitesi: https://mavsdk.mavlink.io/
- MAVSDK GitHub: https://github.com/mavlink/MAVSDK
- MAVROS ROS Paket Sayfası: https://index.ros.org/p/mavros/
- ArduSub Pymavlink Dokümantasyonu: https://www.ardusub.com/developers/pymavlink.html
- MAVLink Vikipedi (protokol paket yapısı): https://en.wikipedia.org/wiki/MAVLink
- Dronecode Forum — MAVROS vs MAVSDK Tartışması: https://discuss.px4.io/t/mavros-or-mavsdk-which-one-should-i-choose/31573

---

*Bu not, kamuya açık resmi dokümantasyon ve topluluk tartışmaları baz alınarak Ağustos 2026 tarihi itibarıyla hazırlanmıştır.*
