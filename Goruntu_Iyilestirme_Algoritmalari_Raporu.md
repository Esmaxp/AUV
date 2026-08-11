# GÖRÜNTÜ İYİLEŞTİRME ALGORİTMALARI ARAŞTIRMA VE ÖN İŞLEM RAPORU

## 1. Su Altı Görüntüleme Problemleri ve Görüntü İyileştirmenin Gerekliliği

Otonom su altı araçlarında (AUV – Autonomous Underwater Vehicle / Otonom Sualtı Aracı ve ROV – Remotely Operated Vehicle / Uzaktan Kumandalı Sualtı Aracı), çevresel farkındalığın oluşturulması ve hedefe yönelik kararların alınabilmesi için kamera sistemlerinden elde edilen görüntüler önemli bir veri kaynağıdır.

Ancak su altı ortamı, havaya kıyasla çok daha karmaşık optik koşullara sahiptir. Işığın su içerisinde ilerlerken soğurulması ve saçılması nedeniyle kameradan alınan ham görüntüler; renk, parlaklık, kontrast ve nesne sınırları açısından bozulabilir.

Bu nedenle görüntünün doğrudan nesne tespit veya otonomi algoritmasına gönderilmesi yerine, öncelikle bir **görüntü ön işleme (preprocessing)** aşamasından geçirilmesi önerilmektedir.

### Ön işleme (Preprocessing) nedir?

Ön işleme, kameradan alınan ham görüntünün daha sonraki algoritmalar tarafından kullanılmaya uygun hale getirilmesi işlemidir.

Sistemdeki temel mantık:

**Kamera → Ham Görüntü → Ön İşleme → Nesne Tespiti / Görsel Algılama → Karar → Hareket**

Buradaki amaç görüntüyü yalnızca insan gözüne daha güzel göstermek değildir. Asıl amaç, sonraki algoritmaların görüntü içerisindeki nesneleri daha doğru ve daha kararlı şekilde algılayabilmesini sağlamaktır.

---

## 1.1. Renk Soğurulması (Color Absorption) ve Spektral Bozulma

**Renk soğurulması (color absorption)**, ışığın farklı dalga boylarının su içerisinde farklı oranlarda enerjisini kaybetmesi olayıdır.

Güneş ışığı veya yapay aydınlatma su içerisinde ilerlerken tüm renkler aynı şekilde kameraya ulaşmaz. Özellikle kırmızı gibi daha uzun dalga boylarına sahip ışık bileşenleri su içerisinde daha hızlı zayıflayabilir. Bu durum suyun özelliklerine, derinliğe, partikül miktarına ve aydınlatma koşullarına bağlı olarak değişir.

Bunun sonucunda görüntüde:

- Kırmızı kanalın bilgisi önemli ölçüde azalabilir.
- Mavi ve yeşil kanallar baskın hale gelebilir.
- Görüntü mavi/yeşil bir **renk baskısına (color cast)** sahip olabilir.
- Nesnelerin gerçek renkleri değişmiş veya ayırt edilmesi zor hale gelmiş olabilir.

### RGB nedir?

**RGB (Red, Green, Blue)**, görüntüyü kırmızı, yeşil ve mavi olmak üzere üç renk kanalıyla ifade eden renk modelidir.

Bir görüntüde her piksel için kabaca:

- **R = kırmızı miktarı**
- **G = yeşil miktarı**
- **B = mavi miktarı**

bilgileri tutulur.

Su altında kırmızı ışığın zayıflaması durumunda görüntüdeki R kanalının bilgisi azalırken G ve B kanalları daha baskın hale gelebilir.

### Otonomi açısından neden önemlidir?

Eğer görev sırasında belirli bir renkteki nesnenin bulunması gerekiyorsa, renk bozulması nesnenin tespit edilmesini zorlaştırabilir.

Örneğin kırmızı bir görev nesnesi su altında gerçek görüntüde kırmızı iken kamerada koyu, gri veya mavi-yeşil tonlarda görünebilir.

Bu durum renk tabanlı görüntü işleme yöntemlerini ve renk bilgisinden faydalanan makine öğrenmesi modellerini olumsuz etkileyebilir.

---

## 1.2. Işık Saçılması (Light Scattering) ve Görüntü Bozulması

Su içerisinde mikroorganizmalar, çözünmüş maddeler, kum, tortu ve diğer partiküller bulunabilir. Ayrıca aracın iticileri (thruster) tarafından oluşturulan su hareketi de tortu ve partiküllerin kameranın görüş alanına girmesine neden olabilir.

Bu partiküller ışığın doğrusal olarak ilerlemesini engelleyerek **ışık saçılması (light scattering)** meydana getirir.

Su altındaki görüntü bozulmasında özellikle iki saçılma türü önemlidir:

### İleri saçılma (Forward Scattering)

Nesneden kameraya doğru gelen ışığın, kameraya ulaşmadan önce su içerisindeki partiküllerle etkileşerek yön değiştirmesidir.

Bunun sonucunda:

- Nesne sınırları bulanıklaşabilir.
- Kenarlar birbirine karışabilir.
- Küçük detaylar kaybolabilir.
- Görüntünün keskinliği azalabilir.

Bu durum özellikle **kenar tespiti (edge detection)** ve **kontur çıkarma (contour detection)** gibi yöntemleri olumsuz etkileyebilir.

### Geri saçılma (Backward Scattering)

Su içerisindeki partiküllere çarpan ortam ışığının veya aracın aydınlatmasının kameraya doğru geri saçılmasıdır.

Bu durum kameranın önünde bir çeşit sis/perde oluşmasına neden olabilir.

Sonuç olarak:

- Görüntünün kontrastı azalır.
- Uzak nesnelerin görünürlüğü düşer.
- Arka plan ile nesne arasındaki fark azalır.
- Kameranın etkin görüş mesafesi düşebilir.

### Kontrast nedir?

**Kontrast**, görüntü içerisindeki açık ve koyu bölgeler arasındaki farktır.

Örneğin:

**Koyu arka plan + açık nesne → yüksek kontrast**

**Koyu gri arka plan + biraz daha açık gri nesne → düşük kontrast**

Otonom sistem açısından yüksek kontrast, nesne ile arka planın birbirinden ayrılmasını kolaylaştırabilir.

---

# 2. Görüntü Ön İşleme (Preprocessing) Pipeline'ı

Yukarıdaki fiziksel problemler nedeniyle kameradan alınan görüntünün doğrudan karar algoritmasına gönderilmesi yerine bir görüntü ön işleme boru hattı (pipeline) oluşturulması önerilmektedir.

### Pipeline nedir?

**Pipeline (boru hattı)**, verinin belirli işlemlerden sırayla geçirilmesi anlamına gelir.

Bu projedeki temel yapı:

**Ham Görüntü → Renk Düzeltme → Sis/Bulanıklık Giderme → Kontrast İyileştirme → Nesne Tespiti → Karar**

şeklindedir.

Bu çalışma kapsamında üç temel yöntem aday olarak incelenmektedir:

1. **Gray World / Color Correction**
2. **UDCP (Underwater Dark Channel Prior)**
3. **CLAHE (Contrast Limited Adaptive Histogram Equalization)**

Ancak bu üç algoritmanın kesin olarak birlikte kullanılacağı varsayılmamalıdır. Uygulama aşamasında farklı kombinasyonlar test edilerek sistem için en uygun yapı belirlenmelidir.

---

## 2.1. Spektral Dengeleme – Gray World Assumption

### Gray World nedir?

**Gray World Assumption**, ideal bir görüntüde ortalama renk değerlerinin nötr bir griye yakın olması gerektiğini varsayan renk düzeltme yaklaşımıdır.

Basit şekilde algoritma şunu yapmaya çalışır:

> Görüntüde belirli bir renk aşırı baskınsa, renk kanallarını yeniden dengeleyerek bu baskınlığı azalt.

Örneğin su altında:

**R = düşük**  
**G = yüksek**  
**B = yüksek**

olabilir.

Gray World yaklaşımı kırmızı kanalın kazancını artırıp diğer kanalları dengeleyerek görüntünün renk dağılımını daha nötr hale getirmeye çalışır.

### Gain nedir?

**Gain (kazanç)**, bir renk kanalının değerlerini belirli bir katsayıyla artırma veya azaltma işlemidir.

Örneğin:

**R × 1.8**

yapılması kırmızı kanalın güçlendirilmesi anlamına gelir.

### Neden kullanıyoruz?

Gray World'un pipeline'ın ilk aşamasında kullanılmasının temel nedeni, sonraki algoritmalara daha dengeli bir renk bilgisi sağlamaktır.

Özellikle:

- Renk tabanlı nesne tespiti
- HSV tabanlı eşikleme
- Nesne sınıflandırma
- Görsel özellik çıkarımı

gibi işlemlerde renk bilgisinin daha dengeli olması faydalı olabilir.

Ancak Gray World'un her su altı görüntüsünde başarılı olacağı garanti değildir. Bu nedenle gerçek görüntüler üzerinde test edilmesi gerekmektedir.

---

## 2.2. Renk Uzayı – HSV

Su altı görüntülerinde renk tabanlı nesne tespiti yapılması durumunda RGB dışında **HSV (Hue, Saturation, Value)** renk uzayı da kullanılabilir.

HSV üç temel bileşenden oluşur:

- **Hue:** Rengin kendisini ifade eder.
- **Saturation:** Rengin ne kadar doygun olduğunu ifade eder.
- **Value:** Rengin parlaklığını ifade eder.

HSV'nin avantajı, rengin kendisini parlaklıktan daha ayrı şekilde ifade edebilmesidir.

Örneğin belirli bir renkteki görev nesnesini bulmak için:

**Hue belirli bir aralıkta mı?**

şeklinde eşikleme yapılabilir.

### Thresholding nedir?

**Thresholding (eşikleme)**, görüntüde belirli bir koşulu sağlayan pikselleri seçme işlemidir.

Örneğin:

**Hue 0–10 arasındaysa → kırmızı nesne**

gibi bir kural oluşturulabilir.

Ancak su altında renklerin bozulması nedeniyle HSV tabanlı eşikleme ham görüntüde başarısız olabilir. Bu nedenle renk düzeltmenin önce uygulanması test edilmektedir.

---

## 2.3. Sis ve Saçılma Giderme – UDCP

### UDCP nedir?

**UDCP (Underwater Dark Channel Prior)**, su altı görüntülerindeki saçılma ve atmosferik perde benzeri bozulmaları azaltmak amacıyla kullanılan, Dark Channel Prior yaklaşımının su altına uyarlanmış bir yöntemidir.

Klasik **Dark Channel Prior (DCP)**, özellikle hava koşullarındaki sisli görüntülerin iyileştirilmesi için geliştirilmiştir.

Ancak su altında kırmızı kanalın ciddi şekilde zayıflaması nedeniyle doğrudan klasik DCP kullanmak uygun olmayabilir.

UDCP bu nedenle su altı görüntülerinin fiziksel özelliklerini dikkate alacak şekilde uyarlanmıştır.

### UDCP'nin temel amacı nedir?

Temel amaç:

**Kamera → su → partiküller → kamera**

sürecinde oluşan saçılma kaynaklı perde etkisini tahmin ederek azaltmaktır.

Bunun sonucunda:

- Uzak nesnelerin görünürlüğünün artırılması,
- Görüntü kontrastının iyileştirilmesi,
- Nesne ve arka plan ayrımının güçlendirilmesi,
- Görsel algılamanın daha kararlı hale getirilmesi

amaçlanmaktadır.

### Transmission Map nedir?

UDCP gibi görüntü restorasyon yöntemlerinde **transmission map (iletim haritası)**, sahnedeki farklı bölgelerden gelen ışığın kameraya ne oranda ulaşabildiğini tahmin etmek için kullanılan bir haritadır.

Basitçe:

**Yüksek transmission → ışık daha temiz ulaşmış**

**Düşük transmission → daha fazla saçılma / perde etkisi**

şeklinde düşünülebilir.

UDCP bu bilgiyi kullanarak görüntünün bozulmuş bölgelerini düzeltmeye çalışır.

### Neden kullanıyoruz?

Gray World esas olarak renk problemini çözmeye çalışırken UDCP'nin hedefi daha çok:

**saçılma + perde etkisi + görünürlük kaybı**

problemidir.

Dolayısıyla iki algoritmanın çözmeye çalıştığı problem farklıdır.

---

## 2.4. Bölgesel Kontrast İyileştirme – CLAHE

### CLAHE nedir?

**CLAHE (Contrast Limited Adaptive Histogram Equalization)**, görüntünün lokal bölgelerindeki kontrastı artıran bir görüntü işleme yöntemidir.

İki önemli kavram içerir:

- **Adaptive Histogram Equalization:** Kontrastı görüntünün tamamına tek seferde değil, lokal bölgelerde artırır.
- **Contrast Limited:** Kontrastın aşırı artırılmasını sınırlar.

### Histogram nedir?

Histogram, görüntüdeki piksel parlaklık değerlerinin dağılımını gösterir.

Örneğin görüntüdeki piksellerin çoğu benzer gri değerlerdeyse görüntünün kontrastı düşük olabilir.

Histogram eşitleme, bu dağılımı daha geniş bir aralığa yayarak kontrastı artırmaya çalışır.

### CLAHE neden standart histogram eşitlemeden farklıdır?

Standart histogram eşitleme bütün görüntüye tek bir işlem uygularken CLAHE görüntüyü küçük bölgelere ayırarak her bölge üzerinde lokal iyileştirme gerçekleştirir.

Bu nedenle su altındaki farklı aydınlatma bölgelerinde daha kontrollü sonuç verebilir.

### Clip Limit nedir?

**Clip Limit**, CLAHE'nin kontrast artırma miktarını sınırlayan parametredir.

Çok yüksek kontrast artışı görüntüdeki gürültüyü de belirginleştirebileceğinden bu parametre önemlidir.

Bu nedenle CLAHE'nin amacı:

**Kontrastı artırmak + gürültü artışını kontrol altında tutmak**

şeklinde özetlenebilir.

### Neden kullanıyoruz?

CLAHE'nin temel amacı:

- Nesne kenarlarını belirginleştirmek,
- Lokal kontrastı artırmak,
- Nesne ile arka plan arasındaki farkı güçlendirmek,
- Kenar ve kontur tabanlı algoritmaların performansını desteklemek

olarak belirlenmiştir.

---

# 3. Önerilen Görüntü İşleme Pipeline'ı

İlk aşamada aşağıdaki pipeline'ın test edilmesi önerilmektedir:

**Ham Görüntü**

↓

**Gray World / Renk Dengeleme**

↓

**UDCP / Saçılma ve Perde Giderme**

↓

**CLAHE / Lokal Kontrast İyileştirme**

↓

**YOLO / OpenCV Tabanlı Algılama**

↓

**Karar ve Kontrol**

Ancak bu yapı nihai çözüm olarak kabul edilmemelidir.

Bunun nedeni, her algoritmanın görüntü kalitesini artırırken aynı zamanda işlem yükü oluşturmasıdır.

Bu nedenle aşağıdaki alternatiflerin de deneysel olarak karşılaştırılması gerekmektedir:

### Deney 1 – Baseline

**Raw → YOLO**

Buradaki **baseline**, herhangi bir görüntü iyileştirme uygulanmadan sistemin mevcut performansını gösteren temel referanstır.

Baseline oluşturmanın amacı diğer algoritmaların gerçekten fayda sağlayıp sağlamadığını ölçebilmektir.

### Deney 2

**Raw → Gray World → YOLO**

### Deney 3

**Raw → UDCP → YOLO**

### Deney 4

**Raw → CLAHE → YOLO**

### Deney 5

**Raw → Gray World → CLAHE → YOLO**

### Deney 6

**Raw → Gray World → UDCP → YOLO**

### Deney 7

**Raw → Gray World → UDCP → CLAHE → YOLO**

Bu deneylerin sonuçları karşılaştırılarak en uygun pipeline belirlenecektir.

---

# 4. Görüntü İyileştirmede Başarı Kriterleri

Bu çalışmanın amacı yalnızca görsel olarak daha güzel görüntüler elde etmek değildir.

Asıl hedef:

**Nesne tespit doğruluğu + işlem hızı + gecikme + donanım kullanımı**

arasında uygun bir denge kurmaktır.

Bu nedenle algoritmaların değerlendirilmesinde aşağıdaki kriterlerin kullanılması önerilmektedir.

## 4.1. Accuracy / Detection Accuracy

Görüntü iyileştirme algoritmasının nesne tespit sistemine gerçekten fayda sağlayıp sağlamadığını görmek için nesne tespit performansı ölçülmelidir.

Eğer sistemde YOLO kullanılıyorsa aşağıdaki metrikler değerlendirilebilir:

### Precision

Modelin "nesne var" dediği sonuçların ne kadarının gerçekten nesne olduğunu gösterir.

Yüksek precision:

**Daha az false positive**

anlamına gelir.

### Recall

Gerçekte bulunan nesnelerin ne kadarının model tarafından yakalandığını gösterir.

Yüksek recall:

**Daha az false negative**

anlamına gelir.

### False Positive

Gerçekte nesne olmadığı halde modelin nesne var demesidir.

### False Negative

Gerçekte nesne olduğu halde modelin bunu tespit edememesidir.

### mAP

**mAP (mean Average Precision)**, nesne tespit modellerinin genel tespit performansını değerlendirmek için kullanılan önemli bir metriktir.

Bu çalışmada görüntü iyileştirme algoritmalarının YOLO performansını artırıp artırmadığını karşılaştırmak için kullanılabilir.

---

## 4.2. FPS – Frames Per Second

**FPS (Frames Per Second)**, sistemin saniyede kaç görüntü karesini işleyebildiğini gösterir.

Örneğin:

**20 FPS = saniyede 20 görüntü**

anlamına gelir.

AUV hareket halindeyken çevresel değişimler hızlı olabileceğinden görüntünün çok yavaş işlenmesi karar mekanizmasında gecikmeye neden olabilir.

Bu nedenle ilk aşamada yaklaşık **15–20 FPS veya üzeri** bir işlem hızı hedef olarak değerlendirilebilir.

Ancak kesin FPS gereksinimi, aracın hareket hızı ve kontrol döngüsünün gereksinimleri ile birlikte belirlenmelidir.

---

## 4.3. Latency – Gecikme

**Latency**, görüntünün kameradan alınması ile işlenmiş sonucun karar algoritmasına ulaşması arasındaki gecikmedir.

Örneğin:

**Kamera → 10 ms**

**Gray World → 3 ms**

**UDCP → 40 ms**

**CLAHE → 5 ms**

**YOLO → 30 ms**

olursa toplam işlem süresi yaklaşık olarak:

**88 ms**

olabilir.

Bu durumda sadece tek bir algoritmanın hızlı olması yeterli değildir. Pipeline'ın tamamının gecikmesi önemlidir.

---

## 4.4. Computational Complexity – Hesaplama Karmaşıklığı

**Computational complexity**, bir algoritmanın giriş verisi büyüdükçe ne kadar hesaplama gerektirdiğini ifade eder.

Teorik analizlerde genellikle **Big-O notation** kullanılır.

Örneğin:

**O(n)**

bir algoritmanın işlem maliyetinin giriş boyutuyla yaklaşık doğrusal olarak arttığını ifade eder.

Ancak bu projede yalnızca Big-O değerine bakmak yeterli değildir.

Gerçek gömülü sistem üzerinde:

- Ortalama işlem süresi
- CPU kullanımı
- GPU kullanımı
- RAM kullanımı
- Güç tüketimi
- FPS

gibi gerçek ölçümlerin alınması daha önemlidir.

---

# 5. Gömülü Sistem Kısıtları

Otonom su altı araçlarında masaüstü bilgisayarlar yerine genellikle **SBC (Single Board Computer)** gibi daha küçük ve düşük güç tüketimli bilgisayarlar tercih edilebilir.

### SBC nedir?

**SBC (Single Board Computer)**, işlemci, RAM ve gerekli bağlantı birimlerini tek bir kart üzerinde bulunduran küçük bilgisayardır.

Örneğin proje donanımına bağlı olarak Raspberry Pi, NVIDIA Jetson gibi platformlar kullanılabilir.

Bu sistemlerde masaüstü bilgisayarlara kıyasla:

- İşlem gücü
- RAM
- Enerji bütçesi
- Soğutma kapasitesi

daha sınırlı olabilir.

Bu nedenle görüntü işleme algoritmasının yalnızca kaliteli sonuç vermesi değil, donanım üzerinde çalışabilecek kadar hafif olması gerekir.

---

## 5.1. Termal Darboğaz – Thermal Throttling

Gömülü sistem yüksek işlem yükü altında uzun süre çalıştığında sıcaklığı artabilir.

**Thermal throttling**, işlemci veya GPU'nun aşırı ısınmayı önlemek amacıyla çalışma frekansını otomatik olarak düşürmesidir.

Sonuç olarak:

**Yüksek işlem yükü → sıcaklık artışı → frekans düşüşü → performans düşüşü → FPS azalması**

oluşabilir.

Bu nedenle görüntü iyileştirme algoritmalarının gereksiz yere işlemciyi sürekli yüksek yükte çalıştırmaması önemlidir.

---

# 6. Gerçek Zamanlı (Real-Time) İşleme

**Real-time processing**, verinin karar mekanizmasının ihtiyaç duyduğu zaman aralığında işlenmesi anlamına gelir.

Buradaki amaç yalnızca "olabildiğince hızlı" olmak değildir.

Asıl amaç:

> Görüntünün, aracın hareket ve kontrol sistemi açısından yeterince hızlı işlenmesi.

Örneğin aracın engelden kaçması gerekiyorsa:

**Kamera → Görüntü İşleme → Nesne Tespiti → Karar → Motor Komutu**

zincirindeki gecikmenin kontrol sistemi açısından kabul edilebilir olması gerekir.

Bu nedenle sistemin yalnızca görüntü kalitesini değil, uçtan uca gecikmeyi de ölçmesi gerekmektedir.

---

# 7. Yazılım Altyapısı

## 7.1. OpenCV

Görüntü işleme algoritmalarının geliştirilmesinde **OpenCV (Open Source Computer Vision Library)** kullanılması önerilmektedir.

OpenCV; görüntü okuma, renk uzayı dönüşümleri, filtreleme, histogram işlemleri, kenar tespiti ve çeşitli görüntü işleme işlemleri için hazır fonksiyonlar sağlayan açık kaynaklı bir bilgisayarlı görü kütüphanesidir.

### Neden OpenCV?

Çünkü:

- Python ve C++ desteği bulunmaktadır.
- Görüntü işleme işlemleri için geniş bir fonksiyon altyapısı vardır.
- Prototipleme sürecini hızlandırır.
- Gömülü sistemlerde kullanılabilecek kadar yaygın ve optimize bir ekosisteme sahiptir.
- YOLO gibi bilgisayarlı görü sistemleriyle birlikte kullanılabilir.

---

## 7.2. Python

Algoritmaların ilk prototiplerinin Python ile geliştirilmesi önerilmektedir.

Python'un tercih edilme nedeni:

- Hızlı geliştirme
- Kolay parametre değiştirme
- OpenCV entegrasyonu
- Deney sonuçlarının hızlı alınabilmesi

gibi avantajlarıdır.

Örneğin:

**Gray World → parametre değiştir → görüntüyü göster → sonucu karşılaştır**

döngüsü Python ile hızlı şekilde gerçekleştirilebilir.

---

## 7.3. C++

Algoritmalar Python üzerinde doğrulandıktan sonra, gerçek zamanlı performansın yetersiz olması durumunda kritik modüllerin C++ ile yeniden uygulanması değerlendirilebilir.

Burada önemli nokta şudur:

**Başlangıçta doğrudan C++ yazmak zorunlu değildir.**

Önce algoritmanın gerçekten gerekli olduğu ve fayda sağladığı Python üzerinde kanıtlanmalı, ardından performans problemi varsa optimizasyon yapılmalıdır.

---

# 8. Modüler Pipeline Mimarisi

Görüntü iyileştirme sisteminin tek parça bir kod yerine modüler şekilde tasarlanması önerilmektedir.

Örneğin:

**Raw Image**

↓

**Color Correction Module**

↓

**Dehazing Module**

↓

**Contrast Enhancement Module**

↓

**Detection Module**

şeklinde bir yapı oluşturulabilir.

### Modüler mimari nedir?

Her işlemin ayrı bir yazılım modülü olmasıdır.

Örneğin:

- `GrayWorld()`
- `UDCP()`
- `CLAHE()`

gibi ayrı fonksiyonlar oluşturulabilir.

Bunun avantajı, herhangi bir algoritmanın kolayca açılıp kapatılabilmesidir.

Örneğin berrak bir havuz ortamında UDCP'nin gerekli olmadığı görülürse:

**Gray World → CLAHE → YOLO**

şeklinde daha hafif bir pipeline kullanılabilir.

Bu sayede gereksiz işlem yükünün azaltılması ve FPS'nin artırılması mümkün olabilir.

---

# 9. Deneysel Test Planı

Çalışmanın teorik araştırmadan gerçek bir mühendislik sonucuna dönüşebilmesi için algoritmaların aynı veri seti üzerinde karşılaştırılması gerekmektedir.

Testlerde mümkün olduğunca farklı su altı koşullarını içeren görüntüler kullanılmalıdır:

- Berrak su
- Bulanık su
- Düşük ışık
- Farklı mesafeler
- Farklı renklerde nesneler
- Farklı arka planlar
- Farklı partikül yoğunlukları

Her görüntü için farklı pipeline'lar uygulanacaktır.

Önerilen karşılaştırma:

| Test | Pipeline | Ölçülecek değerler |
|---|---|---|
| 1 | Raw → YOLO | FPS, latency, precision, recall, mAP |
| 2 | Gray World → YOLO | FPS, latency, precision, recall, mAP |
| 3 | UDCP → YOLO | FPS, latency, precision, recall, mAP |
| 4 | CLAHE → YOLO | FPS, latency, precision, recall, mAP |
| 5 | Gray World → CLAHE → YOLO | FPS, latency, precision, recall, mAP |
| 6 | Gray World → UDCP → YOLO | FPS, latency, precision, recall, mAP |
| 7 | Gray World → UDCP → CLAHE → YOLO | FPS, latency, precision, recall, mAP |

Bu testlerin amacı yalnızca en yüksek görüntü kalitesini bulmak değil, **algılama doğruluğu ile işlem maliyeti arasındaki en uygun dengeyi** belirlemektir.

---

# 10. Beklenen Sonuç ve Karar Mekanizması

Testler sonucunda örneğin aşağıdaki gibi bir sonuç ortaya çıkabilir:

| Pipeline | FPS | mAP |
|---|---:|---:|
| Raw | 25 | 0.61 |
| Gray World | 24 | 0.69 |
| CLAHE | 23 | 0.64 |
| UDCP | 8 | 0.70 |
| Gray World + CLAHE | 22 | 0.74 |
| Gray World + UDCP | 7 | 0.75 |
| Gray World + UDCP + CLAHE | 5 | 0.76 |

Bu değerler yalnızca yöntemin nasıl değerlendirileceğini göstermek amacıyla verilmiş örnek değerlerdir; gerçek sonuçlar deneyler sonucunda elde edilmelidir.

Bu örnekte en yüksek mAP değeri son pipeline'da bulunmasına rağmen yalnızca 5 FPS elde edilmektedir.

Buna karşılık Gray World + CLAHE pipeline'ı daha düşük mAP değerine sahip olsa da 22 FPS ile çalışmaktadır.

Bu durumda gömülü gerçek zamanlı bir sistem için ikinci pipeline daha uygun olabilir.

Dolayısıyla nihai karar:

**"En güzel görüntüyü veren algoritma"**

yerine:

**"Algılama doğruluğu, FPS, latency ve donanım kullanımı açısından en uygun dengeyi sağlayan algoritma"**

üzerinden verilmelidir.

---

# 11. Derin Öğrenme Tabanlı Görüntü İyileştirme Yöntemleri

Literatürde su altı görüntülerinin iyileştirilmesi amacıyla derin öğrenme tabanlı yöntemler de bulunmaktadır.

Örneğin:

- WaterGAN
- FUnIE-GAN
- Diğer CNN/GAN tabanlı görüntü iyileştirme yöntemleri

incelenebilir.

### GAN nedir?

**GAN (Generative Adversarial Network)**, görüntü gibi veriler üretmek veya dönüştürmek için kullanılan bir derin öğrenme mimarisidir.

Bu yöntemler görsel olarak güçlü sonuçlar verebilir; ancak modelin eğitim ve çıkarım (inference) maliyeti, kullanılan donanım ve gerçek zamanlı çalışma gereksinimleri açısından değerlendirilmelidir.

Bu nedenle mevcut sistem için ilk aşamada daha hafif ve kontrol edilebilir klasik görüntü işleme yöntemlerinin baseline olarak kullanılması önerilmektedir.

İlerleyen aşamalarda klasik yöntemlerin performansı yeterli değilse derin öğrenme tabanlı yöntemler alternatif olarak test edilebilir.

---

# 12. Nihai Yazılım Mimarisi İçin Önerilen Yapı

Araştırmanın uygulama aşamasında aşağıdaki yapı hedeflenebilir:

**Kamera**

↓

**Frame Capture**

↓

**Image Preprocessing**

→ Gray World

→ UDCP

→ CLAHE

↓

**Object Detection**

→ YOLO / OpenCV

↓

**Detection Result**

↓

**Decision / Control**

↓

**AUV Actuators**

Buradaki:

### Frame Capture

Kameradan görüntü karesinin alınmasıdır.

### Object Detection

Görüntü içerisinde belirli nesnelerin bulunmasıdır.

Örneğin:

- Kapı
- Duba
- Engel
- Görev nesnesi

### Decision

Algılama sonucuna göre aracın ne yapacağına karar verilmesidir.

Örneğin:

**Kapı sağda → sağa yönel**

**Engel önde → kaçın**

**Hedef bulundu → hedefe yaklaş**

### Control

Verilen kararın motor/itici sistemine komut olarak uygulanmasıdır.

Bu nedenle görüntü işleme sistemi tek başına amaç değildir. Görüntü işleme, AUV'nin çevreyi algılayarak doğru karar vermesine hizmet eden zincirin bir parçasıdır.

---

# 13. Sonuç ve Değerlendirme

Bu rapor, otonom su altı aracının kamera tabanlı algılama sisteminde karşılaşılan renk bozulması, ışık saçılması, düşük kontrast ve bulanıklık problemlerini azaltmak amacıyla gerçekleştirilen ön araştırmayı kapsamaktadır.

Su altı ortamında meydana gelen fiziksel görüntü bozulmaları nedeniyle ham kamera görüntüsünün doğrudan nesne tespit ve otonomi algoritmalarına verilmesi her koşulda yeterli olmayabilir. Bu nedenle görüntünün karar mekanizmasından önce bir ön işleme pipeline'ından geçirilmesi önerilmektedir.

Bu çalışma kapsamında:

**Gray World** → renk dengesini iyileştirmek,

**UDCP** → saçılma ve perde etkisini azaltmak,

**CLAHE** → lokal kontrastı artırmak

amacıyla aday algoritmalar olarak belirlenmiştir.

Ancak bu algoritmaların kesin olarak en iyi çözüm olduğu varsayılmamaktadır. Farklı algoritma kombinasyonlarının aynı veri seti ve aynı donanım koşulları altında test edilmesi gerekmektedir.

Değerlendirme yalnızca görsel kaliteye göre yapılmamalıdır. Bunun yerine:

- Precision
- Recall
- mAP
- FPS
- Latency
- CPU/GPU kullanımı
- RAM kullanımı
- Gerektiğinde güç tüketimi

gibi ölçütler birlikte değerlendirilmelidir.

Çalışmanın nihai amacı maksimum görüntü kalitesini elde etmek değil; **gömülü sistem üzerinde gerçek zamanlı çalışabilecek, nesne tespit performansını iyileştirebilecek ve otonom karar mekanizmasına yeterli kalitede veri sağlayabilecek optimum görüntü işleme pipeline'ını belirlemektir.**

Bu nedenle araştırmanın bir sonraki aşaması algoritmaların Python ve OpenCV kullanılarak prototiplenmesi, farklı pipeline kombinasyonlarının aynı veri seti üzerinde test edilmesi ve elde edilen sonuçların FPS, latency ve nesne tespit başarımı açısından karşılaştırılmasıdır.

Test sonuçlarına göre en uygun yapı belirlendikten sonra, gerekli görülmesi halinde performans optimizasyonu, C++ entegrasyonu veya donanım hızlandırma gibi ileri optimizasyon yöntemleri değerlendirilecektir.

Sonuç olarak bu çalışma, görüntü iyileştirme algoritmalarının teorik olarak seçilmesinden ziyade, **algılama doğruluğu ile gerçek zamanlı çalışma performansı arasında ölçülebilir ve mühendislik açısından uygulanabilir bir denge kurmayı** hedeflemektedir.
