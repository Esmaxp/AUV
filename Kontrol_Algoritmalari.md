# Kontrol Algoritmaları

Bu doküman, otonom bir aracın hareketini (derinlik, yönelim, rota) istenen hedef değerlere getirmek ve orada tutmak için kullanılan iki kontrol algoritmasını — **PID ve Adaptive PID** — en ince ayrıntısına kadar açıklar.

---

## 1. PID (Proportional–Integral–Derivative) Kontrolcü

### 1.1 Genel Tanım

PID, endüstride ve robotikte en yaygın kullanılan **kapalı çevrim (closed-loop) geri beslemeli kontrol** algoritmasıdır. Sistemin mevcut durumu ile istenen hedef (setpoint) arasındaki farkı (**hata, error**) sürekli ölçer ve bu hatayı sıfırlamak için bir düzeltici komut (kontrol sinyali) üretir.

### 1.2 Temel Denklem

Sürekli zaman formunda PID kontrol yasası:

```
u(t) = Kp * e(t) + Ki * ∫e(t)dt + Kd * de(t)/dt
```

- **u(t):** Kontrol çıkışı (örn. motor/thruster'a gönderilen komut)
- **e(t) = setpoint(t) − ölçülen_değer(t):** Anlık hata
- **Kp, Ki, Kd:** Sırasıyla oransal, integral ve türevsel kazanç katsayıları

### 1.3 Üç Terimin Ayrı Ayrı Anlamı

#### a) Oransal Terim (P — Proportional)

- `Kp * e(t)` — hata ne kadar büyükse, düzeltme de o kadar büyük olur.
- Hızlı tepki sağlar, ancak **tek başına asla hatayı tam sıfırlayamaz** — sabit bir kalıcı hata (steady-state error / "droop") bırakır çünkü hata sıfıra yaklaştıkça kontrol sinyali de sıfıra yaklaşır ve sistemin dengeyi koruması için gereken minimum enerji sağlanamayabilir.
- **Kp çok yüksek** → sistem hedefi aşar (overshoot), salınım yapar, kararsız hale gelebilir.
- **Kp çok düşük** → tepki yavaş, hataya duyarsız kalır.

#### b) İntegral Terim (I — Integral)

- `Ki * ∫e(t)dt` — geçmişteki tüm hataların birikimini (alanını) hesaba katar.
- **Kalıcı durum hatasını (steady-state error) sıfırlar** — P teriminin bırakabileceği sabit hatayı zamanla giderir.
- **Ki çok yüksek** → **integral windup** oluşabilir: hata uzun süre aynı yönde kalırsa (örn. aktüatör doygunluğa ulaşmışsa) integral terimi aşırı büyür ve sistem ciddi aşım (overshoot) ve yavaş toparlanma yaşar.
- **Anti-windup teknikleri:**
  - *Clamping (doyum sınırlama):* Kontrol çıkışı doygunluğa ulaştığında integral biriktirmeyi durdurmak.
  - *Back-calculation:* Doygunluk farkını geri besleyerek integral terimini otomatik düzeltmek.

#### c) Türevsel Terim (D — Derivative)

- `Kd * de(t)/dt` — hatanın değişim hızını (trendini) ölçer, geleceği "öngörür".
- Sistemin hedefe yaklaşırken fren yapmasını sağlayarak **salınımı (oscillation) söndürür**, aşımı azaltır.
- **Gürültüye çok duyarlıdır:** Sensör verisindeki küçük gürültüler türev alma işlemiyle büyütülür; bu nedenle genellikle bir düşük geçiren filtre (low-pass filter) ile birlikte kullanılır ("filtered derivative" veya "derivative on measurement").
- **Kd çok yüksek** → sistem gürültüye aşırı tepki verir, titrek (jittery) davranır.

### 1.4 Ayrık Zamanlı (Dijital) Uygulama

Mikrodenetleyici/gömülü sistemlerde PID, örnekleme periyodu `Ts` ile ayrık zamanda hesaplanır:

```
e[k] = setpoint[k] − ölçüm[k]
integral[k] = integral[k-1] + e[k] * Ts
derivative[k] = (e[k] − e[k-1]) / Ts

u[k] = Kp*e[k] + Ki*integral[k] + Kd*derivative[k]
```

Genellikle çıkış, aktüatörün (thruster/motor) fiziksel sınırlarına göre **saturasyona (clamp)** uğratılır (örn. −100 ile +100 arası PWM).

### 1.5 Kazanç Ayarlama (Tuning) Yöntemleri

| Yöntem | Açıklama |
|---|---|
| Manuel (deneme-yanılma) | Önce Kp artırılır (salınıma kadar), sonra Ki eklenir, son olarak Kd ile sönümlenir |
| Ziegler–Nichols | Sistemin kritik kazancı (Ku) ve kritik periyodu (Tu) ölçülerek ampirik formüllerle Kp, Ki, Kd hesaplanır |
| Cohen–Coon | Ölü zamanı (dead time) olan sistemler için Ziegler-Nichols'a alternatif ampirik yöntem |
| Yazılımsal Auto-tune | Sistemin adım/relay yanıtı otomatik analiz edilerek kazançlar hesaplanır |

### 1.6 Aracın Kontrolündeki Uygulaması

Otonom bir araçta genellikle **her eksen için ayrı bir PID döngüsü** çalıştırılır (çoklu-döngü / multi-loop yapı):

- **Derinlik (Depth) PID:** Basınç sensöründen gelen derinlik hatasına göre dikey iticileri kontrol eder.
- **Yönelim (Heading/Yaw) PID:** Manyetometre+IMU füzyonundan gelen yaw hatasına göre yatay/dönüş iticilerini kontrol eder.
- **Roll/Pitch PID:** IMU'dan gelen açı hatasına göre aracın dengede kalmasını sağlar.
- **Kademeli (Cascade) Kontrol:** Bazı sistemlerde dış döngü (konum) hatası, iç döngünün (hız/açısal hız) hedefini belirler — iç döngü daha hızlı çalışarak daha kararlı bir kontrol sağlar.

### 1.7 Avantaj ve Dezavantajları

**Avantajlar:** Basit, anlaşılır, düşük hesaplama maliyeti, endüstride kanıtlanmış, gerçek zamanlı gömülü sistemlere kolay uygulanabilir.

**Dezavantajlar:** Sabit kazançlar yalnızca ayarlandığı çalışma koşulu (operating point) için optimaldir; sistem dinamikleri değiştiğinde (hız artışı, yük değişimi, akıntı, pil voltajı düşüşü) performans bozulur; doğrusal olmayan (nonlinear) sistemlerde sınırlı başarı gösterir.

---

## 2. Adaptive PID (Uyarlamalı PID) Kontrolcü

### 2.1 Motivasyon: Neden Sabit PID Yetersiz Kalır?

Klasik PID'nin Kp, Ki, Kd katsayıları **sabittir** ve genellikle tek bir çalışma noktası (örn. belirli bir hız, belirli bir derinlik) için optimize edilir. Ancak gerçek dünyada:

- Aracın hızı değiştikçe hidrodinamik direnç ve sistemin tepki karakteristiği (dinamiği) değişir.
- Su akıntısı, dalga, yük (payload) değişimi gibi dış bozucular sistemin davranışını değiştirir.
- Pil voltajı düştükçe motor/thruster tepkisi zayıflar.
- Bu durumlarda sabit kazançlı PID ya çok yumuşak (yavaş, hataya toleranslı) ya da çok agresif (salınımlı, kararsız) kalabilir.

**Adaptive PID**, bu sorunu çözmek için Kp, Ki, Kd kazançlarını **çalışma zamanında (online), mevcut koşullara göre otomatik olarak günceller.**

### 2.2 Temel Yapı

```
       ┌─────────────────────┐
       │  Adaptasyon Yasası   │◀── Sistem durumu / performans metriği
       │ (Adaptation Law)     │     (hata, hata değişimi, hız, vs.)
       └──────────┬──────────┘
                   │ Kp(t), Ki(t), Kd(t)
                   ▼
setpoint ──▶ [ e(t) ] ──▶ [ PID Hesabı ] ──▶ u(t) ──▶ Sistem (Araç) ──▶ ölçüm
                ▲                                                        │
                └────────────────────── geri besleme ─────────────────────┘
```

Standart PID döngüsünün üzerine, kazançları sürekli güncelleyen ek bir **adaptasyon mekanizması** eklenir.

### 2.3 Yaygın Adaptive PID Yaklaşımları

#### a) Gain Scheduling (Kazanç Çizelgeleme)

- Farklı çalışma bölgeleri (örn. düşük hız / yüksek hız, sığ derinlik / derin) için önceden hesaplanmış farklı Kp, Ki, Kd kümeleri bir tabloda saklanır.
- Sistem o anki duruma (örn. hız, derinlik) göre uygun kazanç setini seçer veya kümeler arasında yumuşak geçiş (interpolasyon) yapar.
- **Basit ve öngörülebilir**, ancak yalnızca önceden tanımlanmış senaryolara uyum sağlar.

#### b) MRAC — Model Reference Adaptive Control (Model Referanslı Uyarlamalı Kontrol)

- İstenen "ideal" davranışı tanımlayan bir **referans model** oluşturulur.
- Gerçek sistem çıktısı ile referans model çıktısı arasındaki fark sürekli izlenir.
- Adaptasyon yasası (genellikle **MIT kuralı** veya Lyapunov kararlılık teorisine dayalı kurallar), bu farkı azaltacak şekilde PID kazançlarını gerçek zamanlı günceller.
- Sistemin referans modele "benzemesi" hedeflenir.

#### c) Self-Tuning Regulator (Kendi Kendini Ayarlayan Regülatör)

- Sistem dinamiği (plant model), gerçek zamanlı **sistem tanılama (system identification)** ile sürekli tahmin edilir (örn. Recursive Least Squares — RLS yöntemi).
- Güncel model parametrelerine göre kontrolcü kazançları yeniden hesaplanır (örn. kutup yerleştirme — pole placement yöntemiyle).

#### d) Fuzzy-Logic Tabanlı Adaptive PID

- Hata `e(t)` ve hatanın değişim hızı `de/dt` girdi olarak bir **bulanık mantık (fuzzy logic)** sistemine verilir.
- Önceden tanımlanmış bulanık kurallar (örn. "hata büyük VE artıyorsa → Kp'yi artır") ile ΔKp, ΔKi, ΔKd çıktıları üretilir.
- Doğrusal olmayan sistemlerde esneklik sağlar, matematiksel modele ihtiyaç duymaz (rule-based).

#### e) Yapay Sinir Ağı (Neural Network) Tabanlı Adaptive PID

- Bir sinir ağı, sistem hatası ve durumu girdi alarak uygun kazanç ayarlamalarını öğrenir (genellikle gradyan inişi / backpropagation ile eğitilir).
- Karmaşık, doğrusal olmayan ve önceden modellenmesi zor sistemlerde etkilidir.
- Daha fazla işlem gücü ve eğitim verisi gerektirir.

### 2.4 Aracın Kontrolündeki Uygulama Örnekleri

- **Adaptive Heading Control:** Su akıntısı arttıkça, sistem heading hatasındaki artışı algılayıp Kp/Ki değerlerini otomatik yükselterek daha agresif düzeltme uygular; akıntı azaldığında kazançlar tekrar düşürülerek gereksiz salınım önlenir.
- **Adaptive Depth Control:** Aracın kaldırma kuvveti (buoyancy) yük değişimi veya su yoğunluğu farkı nedeniyle değiştiğinde, derinlik PID kazançları buna göre yeniden ayarlanır.
- **Hıza Bağlı Gain Scheduling:** Düşük hızda daha yumuşak (düşük Kp), yüksek hızda daha keskin (yüksek Kp) tepki verecek şekilde kazançlar hıza göre interpolasyonla değiştirilir.

### 2.5 Avantaj ve Dezavantajları

**Avantajlar:**
- Değişken/belirsiz sistem dinamiklerine karşı **sağlamlık (robustness)** sağlar.
- Geniş bir çalışma aralığında (farklı hız, yük, bozucu koşullar) tutarlı performans sunar.
- Dış bozucu etkilere (akıntı, dalga, rüzgar) karşı otomatik telafi sağlar.

**Dezavantajlar:**
- Klasik PID'ye göre çok daha **karmaşık** tasarım ve daha yüksek **hesaplama yükü** gerektirir.
- **Kararlılık (stability) garantisi** matematiksel olarak ispatlamak zordur; kötü tasarlanmış bir adaptasyon yasası sistemi kararsız hale getirebilir.
- Adaptasyon yasasının kendisinin de ayarlanması/doğrulanması gerekir ("kim kazançları ayarlayanı ayarlayacak" problemi).
- Gerçek zamanlı sistem tanılama veya öğrenme, ek sensör/işlemci kaynağı talep eder.

---

## 3. PID vs Adaptive PID — Karşılaştırma

| Kriter | PID | Adaptive PID |
|---|---|---|
| Kazançlar | Sabit | Çalışma zamanında güncellenir |
| Karmaşıklık | Düşük | Yüksek |
| Hesaplama yükü | Düşük | Orta–Yüksek |
| Değişken koşullara uyum | Zayıf | Güçlü |
| Kararlılık analizi | Kolay (klasik kontrol teorisi) | Zor (Lyapunov, adaptif kontrol teorisi gerekir) |
| Uygulama zorluğu | Kolay | Zor (model/kural/ağ tasarımı gerekir) |
| En uygun senaryo | Sabit/öngörülebilir çalışma koşulları | Değişken hız, yük, akıntı, bozucu içeren ortamlar |

## 4. Sonuç

PID kontrolcü, otonom araçlarda temel ve güvenilir bir kontrol katmanı sağlarken; **Adaptive PID**, aracın karşılaştığı değişken ve öngörülemeyen koşullarda (akıntı, yük değişimi, hız değişimi) performansı korumak için bu temel yapıyı dinamik olarak güçlendirir. Pratikte sistemler genellikle klasik PID ile başlar, ihtiyaç halinde gain scheduling veya fuzzy/MRAC tabanlı adaptasyon eklenerek geliştirilir.
