# Navigasyon Sistemleri

Bu doküman, otonom bir aracın (özellikle su altı/insansız araç) konum, yönelim ve hareket bilgisini elde etmesini sağlayan dört temel navigasyon bileşenini — **IMU, Basınç Sensörü, Manyetometre ve Dead Reckoning** — en ince ayrıntısına kadar açıklar.

---

## 1. IMU (Inertial Measurement Unit — Eylemsizlik Ölçüm Birimi)

### 1.1 Nedir?

IMU, bir aracın **açısal hız** ve **doğrusal ivmesini** ölçen, genellikle aynı çip üzerinde birleşik olan bir sensör grubudur. Temelde iki (bazen üç) alt sensörden oluşur:

- **İvmeölçer (Accelerometer):** X, Y, Z eksenlerindeki doğrusal ivmeyi ölçer (yer çekimi dahil).
- **Jiroskop (Gyroscope):** X, Y, Z eksenleri etrafındaki açısal hızı (dönme hızını) ölçer.
- **Manyetometre (opsiyonel, 9 eksenli IMU'larda bulunur):** Yön/pusula bilgisi sağlar (ayrıca aşağıda ayrı başlıkta ele alınmıştır).

3 eksen ivme + 3 eksen jiroskop = **6 DOF (Degrees of Freedom)** IMU. Buna manyetometre eklenirse **9 DOF** olur.

### 1.2 Çalışma Prensibi

Modern IMU'lar **MEMS (Micro-Electro-Mechanical Systems)** teknolojisi ile üretilir:

- **MEMS İvmeölçer:** Küçük bir kütle (proof mass) yaylarla çipe bağlıdır. İvme uygulandığında kütle hareket eder, bu hareket kapasitif plakalar arasındaki kapasitans değişimi olarak ölçülür ve elektriksel sinyale çevrilir.
- **MEMS Jiroskop:** Coriolis kuvveti prensibiyle çalışır. Titreşen bir kütle döndürüldüğünde, dönme eksenine dik yönde bir Coriolis kuvveti oluşur; bu kuvvet kapasitif olarak algılanır ve açısal hıza dönüştürülür.

### 1.3 Ham Veriden Yönelime: Sensör Füzyonu

IMU tek başına ham veri verir (ivme m/s², açısal hız °/s). Bunlardan **roll, pitch, yaw** açılarını (Euler açıları) veya **quaternion** temsilini elde etmek için **sensör füzyon algoritması** gerekir:

- **Tümleştirme (Integration) sorunu:** Jiroskop verisini zamana göre integre ederek açı elde edilebilir, ancak küçük ölçüm hataları zamanla birikir → **drift (kayma)** oluşur.
- **İvmeölçer sorunu:** Yer çekimi vektörü sayesinde statik durumda roll/pitch hesaplanabilir ama titreşim/ivmelenme sırasında gürültülüdür, yaw bilgisi veremez.
- **Çözüm — Füzyon Filtreleri:**
  - **Tamamlayıcı Filtre (Complementary Filter):** Jiroskopun yüksek frekanslı (kısa vadeli) güvenilirliği ile ivmeölçerin düşük frekanslı (uzun vadeli, driftsiz) güvenilirliğini ağırlıklı olarak birleştirir. Basit ve düşük işlem yüküyle çalışır: `angle = α*(angle + gyro*dt) + (1-α)*(accel_angle)`
  - **Kalman Filtresi / Extended Kalman Filter (EKF):** İstatistiksel olarak optimal tahmin sağlar; sensör gürültü modelini kullanarak "predict-update" döngüsüyle çalışır. Daha karmaşık ama daha doğru.
  - **Madgwick / Mahony Filtreleri:** Quaternion tabanlı, hesaplama açısından hafif, gömülü sistemlerde (mikrodenetleyici) yaygın kullanılan füzyon algoritmalarıdır.

### 1.4 Aracın Kontrol Sistemindeki Rolü

- Roll, pitch, yaw açıları → **tutum (attitude) kontrol döngüsüne** (PID/Adaptive PID) geri besleme olarak verilir.
- Açısal hız verisi → dönme sönümleme (damping) ve hızlı tepki gerektiren iç döngülerde (rate loop) kullanılır.
- İvme verisi → titreşim/çarpışma algılama, ivmelenme tahmini için kullanılabilir.

### 1.5 Yaygın Donanımlar

| Sensör | Özellik |
|---|---|
| MPU6050 | 6 eksen (ivme+jiro), ucuz, hobi projelerinde yaygın |
| MPU9250 | 9 eksen (ivme+jiro+manyeto) |
| BNO055 | Dahili sensör füzyon çipi (donanımsal quaternion çıktısı) |
| ICM-20948 | Düşük güç, yüksek hassasiyet, 9 eksen |

### 1.6 Dikkat Edilmesi Gerekenler

- **Kalibrasyon:** İvmeölçer ve jiroskop offsetleri (bias) araç sabitken kalibre edilmelidir.
- **Sıcaklık etkisi:** MEMS sensörlerin bias'ı sıcaklıkla değişir; sıcaklık kompanzasyonu gerekebilir.
- **Örnekleme hızı:** Kontrol döngüsü için genellikle 100–1000 Hz arası güncelleme hızı tercih edilir.
- **Titreşim izolasyonu:** Motor/pervane titreşimleri IMU verisini bozabilir; mekanik izolasyon (damper) önerilir.

---

## 2. Basınç Sensörü (Pressure Sensor)

### 2.1 Amaç

Su altı araçlarında (AUV/ROV) GPS sinyali su içinde yayılamadığı için **derinlik bilgisi** doğrudan basınç ölçümünden elde edilir. Basınç sensörü, aracın su yüzeyine göre ne kadar derinde olduğunu hesaplamak için kullanılır.

### 2.2 Fiziksel Prensip: Hidrostatik Basınç

Bir sıvı içindeki basınç, derinlikle doğru orantılı artar (hidrostatik basınç denklemi):

```
P = P0 + ρ * g * h
```

- **P**: Ölçülen mutlak basınç (Pa)
- **P0**: Yüzeydeki atmosfer basıncı (yaklaşık 101325 Pa)
- **ρ (rho)**: Sıvının yoğunluğu (tatlı su ≈ 997 kg/m³, deniz suyu ≈ 1025 kg/m³)
- **g**: Yerçekimi ivmesi (9.80665 m/s²)
- **h**: Derinlik (m)

Buradan derinlik şu şekilde hesaplanır:

```
h = (P - P0) / (ρ * g)
```

### 2.3 Sensör Teknolojisi

- **Piezorezistif basınç sensörleri:** Basınç uygulandığında direnci değişen bir diyafram kullanır; Wheatstone köprüsü ile ölçülür.
- **MEMS dijital basınç sensörleri:** Basınç + sıcaklık verisini I2C/SPI üzerinden dijital olarak verir.

| Sensör | Kullanım Alanı |
|---|---|
| MS5837-30BA | Su altı (Blue Robotics Bar30), 300m'ye kadar |
| Bar02 | Sığ su, düşük maliyetli su altı sensörü |
| BMP280/BMP388 | Hava basıncı (irtifa), su altı için uygun değil |

### 2.4 Kalibrasyon ve Doğruluk

- **Yüzey sıfırlama (zeroing):** Araç suya girmeden önce, o anki atmosfer basıncı P0 olarak kaydedilir.
- **Yoğunluk değişkenliği:** Tatlı su/tuzlu su yoğunluk farkı derinlik hesabında hataya yol açabilir; kullanıcı ρ değerini ortama göre ayarlamalıdır.
- **Sıcaklık kompanzasyonu:** Sensörün dahili sıcaklık ölçümü, basınç okumasını sıcaklığa göre düzeltmek için kullanılır.
- **Gürültü filtreleme:** Ham basınç verisi dalga/türbülans nedeniyle gürültülü olabilir; düşük geçiren filtre (low-pass filter) veya hareketli ortalama uygulanır.

### 2.5 Kontrol Sistemindeki Rolü

Basınç sensöründen elde edilen derinlik bilgisi, **derinlik tutma (depth-hold) PID döngüsüne** doğrudan geri besleme olarak verilir: hedef derinlik ile ölçülen derinlik arasındaki fark (hata), dikey iticilere (vertical thrusters) uygulanacak itki komutunu belirler.

---

## 3. Manyetometre (Magnetometer)

### 3.1 Amaç

Manyetometre, Dünya'nın manyetik alanını 3 eksende ölçerek aracın **yönünü (heading/yaw)**, yani manyetik pusulaya göre baktığı yönü belirlemeye yarar.

### 3.2 Çalışma Prensibi

- Dünya'nın manyetik alanı, konuma bağlı olarak yatay ve dikey bileşenlere sahiptir.
- Manyetometre, X, Y, Z eksenlerindeki manyetik alan bileşenlerini (genellikle µT veya Gauss cinsinden) ölçer.
- Alanın yatay düzlemdeki izdüşümünden **heading açısı** hesaplanır: `heading = atan2(By, Bx)`
- Araç yatık (tilted) durumdaysa, ham heading hesabı hatalı olur; bu yüzden **tilt-compensation (eğim kompanzasyonu)** gereklidir — IMU'dan gelen roll/pitch açıları kullanılarak manyetik alan vektörü yatay düzleme projekte edilir.

### 3.3 Kalibrasyon: Hard-Iron ve Soft-Iron

Manyetometreler çevresel manyetik bozulmalara karşı çok hassastır ve kalibrasyon şarttır:

- **Hard-Iron (Sert Demir) Hatası:** Araç üzerindeki kalıcı mıknatıslar/manyetik parçalar sabit bir ofset (offset) oluşturur. Kalibrasyon, ölçüm küresinin merkezini orijine kaydırarak düzeltilir.
- **Soft-Iron (Yumuşak Demir) Hatası:** Çevredeki ferromanyetik malzemeler manyetik alanı bozarak küreyi elipsoide dönüştürür. Kalibrasyon, elipsoidi tekrar küreye dönüştüren bir ölçekleme/döndürme matrisi ile yapılır.
- **Kalibrasyon Yöntemi:** Araç her yönde (özellikle "8" çizer şekilde) döndürülerek veri toplanır, en küçük kareler (least-squares) yöntemiyle elipsoid uydurma (ellipsoid fitting) yapılır.

### 3.4 Manyetik Sapma (Declination)

- **Manyetik kuzey** ile **gerçek (coğrafi) kuzey** arasında konuma bağlı bir açı farkı vardır (declination).
- Doğru navigasyon için, ölçülen manyetik heading'e bulunulan konumun deklinasyon açısı eklenir/çıkarılır.

### 3.5 Parazit ve Yerleştirme

- Motor, ESC (Electronic Speed Controller) ve yüksek akım taşıyan kablolar güçlü elektromanyetik parazit üretir.
- Manyetometre, bu kaynaklardan mümkün olduğunca uzağa yerleştirilmeli veya manyetik kalkanlama (shielding) uygulanmalıdır.

### 3.6 Füzyon

Manyetometre tek başına gürültülü ve yavaş tepkilidir; jiroskopla birlikte füzyon filtresine (Complementary/Kalman/Madgwick) girdi olarak verilerek **kararlı ve sürekli yaw tahmini** elde edilir — jiroskop kısa vadede hızlı ve hassastır ama drift yapar, manyetometre uzun vadede driftsiz mutlak referans sağlar.

---

## 4. Dead Reckoning (Kestirilmiş Seyir)

### 4.1 Tanım

Dead Reckoning (DR), bir önceki bilinen konumdan başlayarak; **hız, yön (heading) ve geçen süre** bilgisini kullanarak aracın güncel konumunu **tahmin etme** yöntemidir. GPS gibi mutlak konum verisi olmadığında (özellikle su altında, GPS sinyali su içine giremediği için) kullanılan temel navigasyon tekniğidir.

### 4.2 Matematiksel Model

Ayrık zamanlı (discrete-time) temel DR denklemi:

```
x(t+Δt) = x(t) + v * cos(ψ) * Δt
y(t+Δt) = y(t) + v * sin(ψ) * Δt
z(t+Δt) = z(t) + vz * Δt   (derinlik, basınç sensöründen de doğrudan alınabilir)
```

- **x(t), y(t):** Önceki bilinen konum
- **v:** Aracın hızı (genellikle DVL veya pervane/itki modelinden tahmin edilir)
- **ψ (psi):** Heading açısı (manyetometre + jiroskop füzyonundan)
- **Δt:** Zaman adımı

### 4.3 Hız Bilgisinin Kaynağı

Dead reckoning'in doğruluğu, hız tahmininin kalitesine doğrudan bağlıdır:

- **DVL (Doppler Velocity Log):** Deniz tabanına veya su kütlesine göre Doppler etkisiyle gerçek hızı ölçer — en doğru yöntemdir ama pahalıdır.
- **Pervane devri (RPM) + itki modeli:** Motor devri ile hidrodinamik model kullanılarak hız tahmini yapılır — daha ucuz ama daha az doğru (akıntı etkisini hesaba katmaz).
- **IMU çift integrasyonu:** İvmenin iki kez integre edilmesiyle hız ve konum elde edilebilir, ancak hata kübik olarak büyür (çok hızlı drift eder), tek başına pratik değildir.

### 4.4 Hata Birikimi (Drift) Problemi

Dead reckoning'in en büyük dezavantajı, her ölçümdeki küçük hataların **zamanla kümülatif olarak birikmesidir**:

- Heading hatası (örneğin manyetometre kalibrasyon hatası) mesafe arttıkça konum hatasını büyütür.
- Hız tahmin hatası (özellikle su akıntısı dikkate alınmazsa) doğrudan konum hatasına dönüşür.
- Düzeltme yapılmadan uzun süre DR ile gidilirse, tahmini konum gerçek konumdan önemli ölçüde sapabilir.

### 4.5 Hata Düzeltme Yöntemleri

- **Periyodik GPS düzeltmesi:** Araç su yüzeyine çıktığında GPS ile konum sıfırlanır (fix).
- **Akustik konumlandırma:** USBL (Ultra-Short Baseline) veya LBL (Long Baseline) sistemleri su altında mutlak/relatif konum düzeltmesi sağlar.
- **SLAM (Simultaneous Localization and Mapping):** Çevresel referans noktaları (sonar, kamera) kullanılarak konum düzeltilir.
- **Sensör Füzyonu (EKF/UKF):** Tüm sensörler (IMU, basınç, manyetometre, DVL, GPS-fix) bir **Extended/Unscented Kalman Filter** içinde birleştirilerek en olası konum ve yönelim tahmini üretilir; bu, gerçek sistemlerde tercih edilen yaklaşımdır.

### 4.6 Kontrol Sistemiyle İlişkisi

Dead reckoning çıktısı olan tahmini konum (x, y, z) ve heading (ψ), aracın **yol takibi (waypoint navigation)** ve **heading-hold kontrol döngüsüne** girdi olarak verilir; PID/Adaptive PID kontrolcüsü bu tahmini konuma göre iticileri (thruster) yönlendirir.

---

## 5. Sensörlerin Bütünsel Füzyonu (Özet)

```
        ┌──────────┐     ┌────────────────┐
        │   IMU    │────▶│                │
        └──────────┘     │                │
        ┌──────────┐     │   Extended     │     ┌─────────────────┐
        │ Basınç   │────▶│   Kalman       │────▶│ Konum (x,y,z)    │
        │ Sensörü  │     │   Filter (EKF) │     │ Yönelim (r,p,y)  │
        └──────────┘     │  (Dead Reck.   │     │ Hız (vx,vy,vz)   │
        ┌──────────┐     │   + Füzyon)    │     └─────────────────┘
        │Manyeto-  │────▶│                │              │
        │metre     │     └────────────────┘              ▼
        └──────────┘                              Kontrol Algoritmasına
                                                    (PID / Adaptive PID)
```

Dört sensör tek başlarına eksik/hatalı bilgi verirken, birlikte bir **Kalman filtresi tabanlı füzyon** ile aracın tam durum vektörünü (konum, hız, yönelim) güvenilir şekilde tahmin etmeyi sağlar. Bu tahmin, kontrol algoritmalarının (PID/Adaptive PID) geri besleme girdisidir.
