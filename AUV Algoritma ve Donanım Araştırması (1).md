Otonom su altı araçlarının (AUV) geliştirilmesi, robotik alanındaki en zorlu mühendislik problemlerinden birini temsil etmektedir. Su altı ortamı, yüksek basınç, elektromanyetik dalgaların hızlı bir şekilde soğurulması, değişken ışık koşulları, bulanıklık ve doğrusal olmayan hidrodinamik bozucu etkiler barındırır1. Bu zorlu koşullar altında çalışan bir aracın algılama, navigasyon, güdüm ve kontrol (GNC - Guidance, Navigation, and Control) sistemlerinin, masaüstü veya sığ havuz ortamlarında elde edilen teorik başarıları gerçek saha koşullarında sürdürebilmesi için, algoritmik temellerinin son derece dayanıklı (robust) ve esnek olması

gerekmektedir. Kullanıcı tarafından sunulan algoritma ve yazılım tasarım süreci incelendiğinde, aracın görev

tanımlarına (şerit takip, mini ROV bırakma, boru hattı takibi) yönelik geliştirilen çözümlerin klasik mekatronik kontrol felsefesine ve eski nesil görüntü işleme standartlarına dayandığı görülmektedir. Bu yaklaşım, sistemin karmaşık hidrodinamik dinamiklerini, sensör hatalarının zamanla birikme (drift) eğilimini ve donanım/işletim sistemi mimarisindeki modern gereksinimleri tam olarak karşılayamamaktadır. Uluslararası arenada düzenlenen RoboSub gibi otonom su altı yarışmalarında ve önde gelen deniz robotolojisi enstitülerinde uygulanan küresel stratejiler3, salt kodlama pratiklerinin ötesinde; donanım hızlandırmalı yapay zeka çıkarımı, akustik-optik sensör füzyonu, deterministik haberleşme protokolleri ve doğrusal olmayan uyarlanabilir kontrol algoritmalarına dayanmaktadır. Bu rapor, mevcut tasarımda yer alan kavramsal yanılgıları ve algoritmik darboğazları tespit ederek, otonom su altı araçlarında uygulanan en güncel küresel standartları, donanım gereksinimlerini ve sistem optimizasyon yöntemlerini derinlemesine incelemektedir.

## Mevcut Tasarımın Teşhisi ve Algoritmik Zafiyet Analizi

Önerilen algoritma mimarisi, kendi içerisinde modüler bir yapı hedeflemiş olsa da; donanım seçimi, kontrol teorisi ve bilgisayarlı görü (computer vision) yöntemleri açısından günümüz su altı robotiği standartlarının gerisinde kalmaktadır. Tespit edilen temel sorunlar, sistemin bütünsel kararlılığını ve görev başarımını doğrudan tehdit eden yapısal kısıtlamalardan kaynaklanmaktadır.

## Donanım ve Yazılım Ekosistemi Uyumsuzluğu: IPC Darboğazı ve İşletim Sistemi Kısıtları

Mevcut sistemde, yüksek seviye görüntü işleme algoritmalarının NVIDIA Jetson Nano üzerinde, düşük seviye sensör okuma ve kontrol işlemlerinin ise Raspberry Pi üzerinde çalıştırıldığı, iki platformun Python scriptleri ve pymavlink üzerinden haberleştiği belirtilmiştir. Sistemin ilerleyen aşamalarında ise ROS 2 Humble Hawksbill sürümüne geçiş planlanmaktadır. Bu mimari karar, yazılım mühendisliği açısından çeşitli açmazlar barındırmaktadır. Öncelikle, NVIDIA Jetson Nano (Maxwell mimarisi), resmi olarak JetPack 4.6 desteğine sahip olup Ubuntu 18.04 işletim sistemi ile sınırlandırılmıştır. Oysa belirtilen ROS 2 Humble sürümü, çekirdek bağımlılıkları itibarıyla Ubuntu 22.04 LTS işletim sistemini yerel (native) olarak talep etmektedir7. Eski bir platforma yeni nesil ROS 2 sürümünün konteyner (Docker) olmadan veya kaynak koddan zorlanarak kurulması, kütüphane çakışmalarına ve GPU hızlandırma (CUDA)

desteğinin kaybedilmesine neden olacaktır.


İkinci büyük sorun, iki farklı tek kartlı bilgisayar (SBC) arasında kurulan pymavlink tabanlı seri/ağ haberleşmesinin yarattığı Süreçler Arası İletişim (Inter-Process Communication - IPC) darboğazıdır6. Otonom sistemlerde kontrol döngülerinin (özellikle 6 eksenli itici kontrolünün) çok düşük gecikmeyle (latency) işletilmesi gerekir. Görüntü işlemeden çıkan sapma hatasının Raspberry Pi'ye aktarılması ve oradan Pixhawk'a iletilmesi, kontrol döngüsü fazında gecikmelere yol açarak sistemin "ölü zaman" (dead time) sınırını aşmasına neden olabilir. Küresel araştırmalarda, parçalı SBC kullanımı yerine tüm otonomi ve algılama yığınının tek bir güçlü platform (örneğin Jetson Orin NX) üzerinde ROS 2 DDS (Data Distribution Service) altyapısı kullanılarak çalıştırılması tercih edilmektedir6.

## Kontrol Kuramı Yanılgıları: Ziegler-Nichols ve Klasik PID Yetersizliği

Araç dinamiğinin kontrolünde klasik PID (Oransal-İntegral-Türevsel) denetleyicilerin kullanıldığı, katsayı ayarlarının ise Ziegler-Nichols (Z-N) yöntemi ile yapılacağı ifade edilmiştir. AUV dinamiği gibi yüksek derecede doğrusal olmayan ve değişken kütle-eylemsizlik özelliklerine sahip bir sistem için Z-N yöntemi kavramsal olarak uygunsuzdur.

Ziegler-Nichols yöntemi, sistemin zamanla değişmeyen doğrusal bir model (LTI) olduğu varsayımıyla marjinal kararlılık (sürekli osilasyon) noktasını bularak parametre hesaplar9. Ancak, su altı aracının dinamiği Newton-Euler hareket denklemleri ile aşağıdaki gibi ifade edilir:

Bu denklemdeki matrisi yalnızca aracın kendi katı cisim kütlesini (

) değil, aracın

hareketi sırasında etrafında sürüklenen suyun oluşturduğu eklenmiş kütle (added mass,

)

matrisini de barındırır (

)11.

Coriolis kuvvetlerini,

doğrusal

olmayan hidrodinamik sönümlemeyi (kuadratik sürüklenme),

hidrostatik geri çağırıcı

kuvvetleri (kütle ve yüzdürme merkezlerinin uyumsuzluğu), ise motorların sağladığı giriş torkunu temsil eder13.

Mevcut tasarımda, aracın hızı veya akıntı şiddeti değiştiğinde

doğrusal olmayan (

sürüklenme matrisi

) bir şekilde değişecektir10. Ziegler-Nichols ile sığ suda ve durağan

koşullarda (havuz testi) ayarlanan kritik kazanç ( ), aracın hızı arttığında veya Mini ROV

bırakma görevinde kütlesi (

) aniden azaldığında geçerliliğini yitirecektir14. Bunun

sonucunda sistem, integral birikmesi (windup) yaşayarak itici doygunluğuna (thruster saturation) girecek ve sönümlenemeyen osilasyonlar üreterek kararsızlığa sürüklenecektir10.

Navigasyon ve Konumlandırma Krizleri: Akustik Hız Ölçümünün


## Yokluğu

Tasarlanan sistemde, şerit takibi veya boru hattı gibi hedef merkezli yönelimlerin geçici olarak

kaybolması durumunda ölü hesaplama (dead reckoning) yöntemine geçileceği; bunun da IMU'dan elde edilen ivme ve açısal hız verilerinin entegrasyonu ile sağlanacağı belirtilmiştir. Tüketici/Endüstriyel sınıf MEMS (Micro-Electro-Mechanical Systems) IMU sensörleri barındırdıkları beyaz gürültü (white noise) ve rastgele yürüyüş (random walk) karakteristiği nedeniyle mutlak konumlandırma için kullanılamazlar16. İvmeölçer verisinin hıza dönüştürülmesi için birinci derece, konuma dönüştürülmesi için ikinci derece (çift) entegrasyon işlemi gereklidir. Gürültülü bir sinyalin çift entegrasyonu, hata payının zaman içinde karesel ve üstel (quadratic) olarak büyümesine (drift) sebep olur16. Uygulamada, sadece IMU kullanılarak yapılan ölü hesaplama birkaç saniye içinde metrelerce konum sapmasına yol açacak ve aracı görev alanından tamamen çıkaracaktır. Su altı ortamında elektromanyetik dalgalar hızla soğurulduğu için (GPS yoksunluğu) hız ve konum tahmini, yüksek frekanslı ve mutlak bir doğrulukla çalışan Doppler Hız Kaydı (Doppler Velocity Log - DVL) akustik sensörlerine devredilmelidir17. Mevcut tasarımdaki mimari, bir DVL entegrasyonu barındırmadığı için uzun süreli otonomiden ziyade "körleme itki" modeline indirgenmiştir.

## Görüntü İşleme: Klasik Bilgisayarlı Görünün Açmazları

Görev algoritmaları kapsamında RGB-HSV dönüşümü, White Balance (Gray World varsayımı), CLAHE, renk tabanlı maskeleme ve morfolojik operatörlerden oluşan bir görüntü işleme hattı (pipeline) sunulmuştur. Bu yaklaşım, su altı optik spektrumunun fiziğini göz ardı eden aşırı basitleştirilmiş bir modeldir.

Işığın su içerisindeki sönümlenmesi dalga boyuna bağlıdır; kırmızı ışık spektrumu 5 metreden sonra neredeyse tamamen yok olurken, daha derinde yalnızca mavi ve yeşil spektrum kalır18. Gray World varsayımı, bir görüntüdeki ortalama rengin nötr gri olduğu kuralına dayanır; ancak su altında genel aydınlatma tamamen mavi/yeşile kaydığı için bu varsayım çöker19. CLAHE (Contrast Limited Adaptive Histogram Equalization) ise kontrastı yerel olarak artırırken, bulanık suda bulunan çözünmüş partiküllerin (backscatter - geri saçılım) oluşturduğu parazitleri nesne dokusu gibi belirginleştirerek hatalı kontur (contour) analizlerine yol açar20. Havuz ortamında ayarlanan manuel HSV eşik değerlerinin, gün ışığı yansımaları veya su dalgalanmaları sırasında tamamen anlamsızlaşacağı küresel test tecrübeleriyle sabittir. Gerçek yarışmalarda ve akademik araştırmalarda kural tabanlı bu işlemler (if-else mantığı), tamamen Derin Öğrenme (Deep Learning) tabanlı algılama mimarilerine yerini bırakmıştır18.

## Küresel AUV Geliştirme Süreçleri ve Modern Yazılım Standartları

Dünya genelindeki otonom su altı aracı projeleri (örn. RoboSub'da yer alan NUS Bumblebee, METU, UWRT, Mecatron ekiplerinin tasarımları ve akademik literatür) incelendiğinde3, mevcut problemlerin çözümü için yazılım mimarisi, sensör füzyonu ve görev planlama mekanizmalarında standartlaşmış modern metodolojiler uygulandığı görülmektedir.


## ROS 2 Humble ve DDS Altyapısı Üzerinde Konteynerizasyon

Gelişmiş AUV sistemleri, donanımdan soyutlanmış, ölçeklenebilir ve deterministik çalışan bir mesajlaşma yapısına ihtiyaç duyar. ROS 2 (Robot Operating System 2), bu ihtiyacı Data Distribution Service (DDS) adı verilen, gerçek zamanlı ve dağınık bir iletişim protokolü ile karşılar6. ROS 2'nin DDS yapısı, UDP üzerinde çalışarak ağda paket kayıplarını tolere edebilen Kalite Yönetimi (QoS - Quality of Service) politikaları sunar. Bu durum, sensör verilerinin (örneğin yüksek bant genişliği gerektiren kamera verisi) işlemci çekirdekleri arasında sıfır kopya maliyetiyle (zero-copy transport) taşınmasını mümkün kılar3.

Küresel ekipler (örn. Mecatron ve Bumblebee), geliştirme sürecini ve saha dağıtımını (deployment) Docker konteynerizasyonu üzerinden yönetmektedir3. Docker kullanımı, Jetson veya x86_64 tabanlı yer kontrol istasyonlarındaki paket bağımlılıklarını izole eder. Sistem genellikle iki ana konteynere ayrılır:

- 1. Algılama (Perception) Konteyneri: Yalnızca GPU kullanan işlemleri (CUDA, TensorRT, PyTorch bağımlılıkları) barındırır.

- 2. Otonomi ve Navigasyon Konteyneri: Derleme kütüphanelerini, EKF düğümlerini ve kontrol çıkışlarını barındırır3.

Bu hiyerarşi, bir geliştiricinin grafik kartı olmadan yalnızca navigasyon düğümleri üzerinde simülasyon yapmasına ve ardından aynı Docker imajını hiçbir sürüm uyumsuzluğu yaşamadan doğrudan AUV donanımına yüklemesine olanak tanır.

## Davranış Ağaçları (Behavior Trees) ile Kapsamlı Görev Yönetimi

Mevcut tasarımda belirtilen "zaman aşımı (timeout)" ve "acil çıkış" durum makinesi protokolleri, karmaşık su altı görevleri için fazlasıyla rijit (esnek olmayan) ve deterministik hatalara açık yöntemlerdir. Otonomi dünyasında klasik Durum Makineleri (Finite State Machines) yerine, Nav2 altyapısında da çekirdek rol oynayan Davranış Ağaçları (Behavior Trees - BT.CPP) kullanılmaktadır6.

Davranış ağaçları, görevleri küçük, test edilebilir yaprak düğümlere (leaf nodes) böler ve çalışma mantığını Akış Kontrol Düğümleri (Sequence, Fallback, Parallel) ile yönetir. Örneğin, "Boru Hattı Takibi ve Hedef Tespiti" görevinde:

- Bir Sequence (Sıralı) düğümü sırasıyla "Hedefe Hizalan" ve "İleri Git" komutlarını işletir.

- Hedef aniden kaybedilirse, bir Fallback (Geri çekilme/Kurtarma) düğümü tetiklenerek anında "Son Konumdan Etrafı Tara" veya "DVL üzerinden akustik arama başlat" adlı başka bir davranış dizisini devreye sokar6. BT mimarisi, beklenmedik sensör kayıpları durumunda AUV'nin sonsuz döngüde kalmasını veya gereksiz yere acil durum çıkışına geçmesini engelleyerek, otonomiyi doğasına uygun olarak adaptif kılar.

## Çift AUV (Dual-Vehicle) Operasyonları ve Risk Dağıtımı

Modern rekabetçi stratejiler, sistem karmaşıklığını ve tek bir hata noktasında (Single Point of Failure) tüm görevin kaybedilmesi riskini yönetmek amacıyla "Çift AUV" operasyonlarını benimsemiştir3. Örneğin Hydra ve Kraken (Mecatron) veya Mini-AUV (Bumblebee) stratejilerinde olduğu gibi3; ağır, büyük eylemsizliğe sahip ve stabilite gerektiren manipülasyon (örneğin kapak açma, hedef bırakma) görevleri için büyük bir platform kullanılırken; yüksek hız


ve çeviklik gerektiren (şerit takip, geçiş kapısı ve slalom) görevleri için optimize edilmiş daha küçük, itici motorları farklı konumlandırılmış bir Mini-AUV aynı anda sahaya sürülür3. Görevlerin iki araca akustik veya optik senkronizasyon ile dağıtılması, taktiksel esnekliği maksimuma çıkarır5.

## İleri Düzey Kontrol ve Yörünge Takip Algoritmaları

Doğrusal olmayan hidrodinamik kuvvetlerin ve kütle-eylemsizlik değişkenlerinin bulunduğu su altı ortamında, sistem performansının Ziegler-Nichols PID ötesine taşınması zorunludur.

## Kayan Mod Kontrolü (Sliding Mode Control - SMC)

Otonom su altı araçlarının 6-DoF hareketlerinin kontrolünde, modellenmemiş hidrodinamik kuvvetlere, su akıntılarına ve ani kütle değişimlerine karşı dünyada en yaygın tercih edilen robust (dayanıklı) kontrol algoritması Kayan Mod Kontrolüdür (SMC)15. SMC'nin çalışma prensibi, sistemin durum değişkenlerini (örneğin AUV'nin mevcut derinliği ve hedef derinliği arasındaki hata) matematiksel olarak önceden tanımlanmış, sistem hatalarının sıfıra yakınsadığı bir hiperdüzleme (sliding surface) yönlendirmektir15.

- Uygulama Senaryosu (Mini ROV Bırakma): Mevcut algoritmada, ROV serbest bırakıldığında AUV'de yaşanacak ani kütle ve yüzdürme kuvveti değişimi için PID'ye aktif stabilizasyon ekleneceği ancak bunun nasıl yapılacağının belirsiz olduğu ifade edilmiştir. SMC tabanlı bir denetleyicide, aracın kütlesi veya eylemsizlik momenti beklenmedik oranda (%20-30 gibi) azalsa dahi, sistemin kayan yüzey üzerinde kalmasını sağlayan yüksek frekanslı bir anahtarlama (switching) kontrol sinyali üretilir. Sistem kütlesindeki bu değişim bir dış bozucu olarak modellenir ve SMC, Lyapunov kararlılık teorisine uygun olarak bu bozucuyu anında sönümler15.

## Bulanık Uyumlu (Fuzzy Adaptive) PID ve Doğrusal Kuadratik Regülatör (LQR)

Klasik PID kontrolünün yapısal sadeliğini koruyarak doğrusal olmayan AUV sistemlerine entegre etmek için modern yaklaşım Bulanık Uyumlu PID (Fuzzy Adaptive PID) kullanmaktır14. Bu

denetleyicide

ayarladığı sabit değerler değildir. Hata ( ) ve hatanın değişim hızı ( ) girdileri, bir bulanık mantık motoruna (Fuzzy Logic Controller) beslenir. "Eğer hata çok yüksekse ve hata hızı

artıyorsa,

anlık olarak (gerçek zamanlı) yeniden hesaplanır14. Çevresel koşullar (akıntı hızının artması vb.) ne kadar değişirse değişsin, bulanık kurallar itici doygunluğunu engelleyerek kararlı (Routh-Hurwitz stabilite sınırları içinde) bir sürüş sunar14.

Ayrıca, enerji bütçesinin kısıtlı olduğu uzun soluklu boru hattı takibi görevlerinde, enerji tüketimini minimumda tutarak hedef noktasına ulaşmayı sağlayan Doğrusal Kuadratik Regülatör (LQR) algoritmaları tercih edilmektedir24. LQR algoritmalarının veya PID parametrelerinin en ideal çalışma ağırlıklarını (Q ve R matrisleri) bulmak için Z-N yöntemi yerine

,

ve

parametreleri dışarıdan bir mühendisin havuz başında

'yi artır ve

'yi düşür" gibi uzman kural tabanları çalıştırılarak parametreler


SMAC (Sequential Model Algorithm Configuration) gibi yapay zeka destekli optimizasyon metotları kullanılır24.

## Şerit Takibinde Line-of-Sight (LOS) Güdüm Yasası

Mevcut tasarımdaki "Şerit Takip Görevi"nde hata sinyali, yalnızca görüntünün yatay merkezi ile hedef ağırlık merkezi arasındaki piksel farkı olarak belirlenmiş ve araca doğrudan "yaw" (sapma) dönüşü olarak beslenmiştir. Bu yaklaşım, kinematik açıdan bir aracın düz bir hattı takip etmesi (path following) yerine "köpek-tavşan" takibi olarak bilinen yengeç yürüyüşü (crabbing) yapmasına sebep olur. Araç, sürekli şeridin üzerine dönmeye çalışırken akıntı onu yana doğru

itecek ve sönümlenemeyen geniş açılı yalpalamalar (S-çizme) ortaya çıkacaktır. Küresel standartlarda, bir şeridi pürüzsüz biçimde takip etmek için Line-of-Sight (LOS) veya Vektör Alanı (Vector Field) güdüm algoritmaları kullanılır25. LOS yaklaşımında, şerit düzlemi

referans alınarak araca sanal bir "Cross-Track Error" (yanal sapma hatası) hesaplatılır. Aracın

hedef açısı ( ), mevcut yanal sapmaya bağlı olarak ve sistemin stabilite marjlarına uygun bir

ileri bakış mesafesi ( ) ile yeniden tanımlanır:

Bu sayede AUV, yanal eksende bir akıntıya maruz kalsa dahi şeridin dışına savrulmadan rüzgar gülü gibi yelken açarak (crab angle) hattı doğrudan takip edebilir25.

## Modern Sensör Füzyonu ve Deterministik Konumlandırma Mimarisi

Otonom navigasyonun bel kemiği, doğruluğundan emin olunamayan veya farklı frekanslarda (örneğin 10 Hz Sonar, 20 Hz Kamera, 100 Hz IMU) sisteme akan sensör verilerinin eş zamanlı olarak birleştirilmesidir.

## Genişletilmiş Kalman Filtresi (EKF) ve UKF Entegrasyonu

Dış dünyanın ROS 2 düğümlerindeki izdüşümü robot_localization paketi ile sağlanır1. Bu yapı içerisinde Genişletilmiş Kalman Filtresi (EKF) veya daha ileri düzey Kokusuz Kalman Filtresi (UKF) çalıştırılmaktadır1. Mevcut tasarımdaki "basit ortalama" veya doğrudan "ölü hesaplama" algoritmalarından farklı olarak EKF, istatistiksel bir gürültü modeli kullanır.

- Tahmin Aşaması (Prediction): Araç, IMU ivme ve jiroskop verilerini kullanarak kinematik model üzerinden kendi konumunu sürekli tahmin eder. Ancak sistem bu veride gürültü olduğunu (kovaryans matrisinin büyüdüğünü) bilir16.

- Güncelleme Aşaması (Update): Akustik (DVL) ve çevresel (basınç sensörü) veya görsel odometri (VIO) verileri ulaştığında, EKF "Kalman Kazancını (Kalman Gain)" hesaplayarak, tahmin edilen konum ile yeni sensör ölçümü arasındaki farkı ağırlıklandırır ve konumu gerçek (ground truth) değerine çekerek günceller1.


Sensör arızası durumunda (örneğin kameranın geçici körlüğü), ROS 2'deki EKF filtreleri ilgili sensörün varyans değerini sonsuza yaklaştırarak o sensörü yumuşak bir şekilde dikkate almamaya (fuse etmemeye) başlar1. Bu, hata durumlarında ani sıçramaların ve algoritmanın çökmesinin önüne geçen en büyük güvencedir.

## DVL (Doppler Velocity Log) Stratejik Gerekliliği

AUV operasyonlarında DVL kullanımı lüks değil, otonom seyir için fiziksel bir gerekliliktir1. AUV'nin tabana gönderdiği 4 akustik hüzme (Janus konfigürasyonu), geri yansıyan ses dalgalarındaki frekans kayması üzerinden aracın X, Y ve Z eksenlerindeki hızını milimetre hassasiyetiyle belirler27. Modern sistemler (örneğin 1 MHz frekansla çalışan Water Linked DVL A50), zemin mesafesi 5 cm iken bile "bottom lock" (zemin kilidi) oluşturarak 3 m/s yatay hıza kadar doğrusal ölü hesaplama yapılmasına imkan tanır28. Eğer DVL olmazsa, mevcut tasarımın güvenmeyi hedeflediği IMU entegrasyonu, aracın rotasını şaşırması ve hedef alanının dışına drift etmesi ile sonuçlanacaktır17. İleri seviye araştırmalarda, görsel ve akustik sensörlerin birleştirildiği (AQUA-SLAM, VISO veya RUSSO gibi) sıkı bağlı (tightly-coupled) faktör grafiği optimizasyonları sayesinde DVL'in taban referansını kaybettiği derin veya aşırı bitki örtüsüne sahip durumlarda kameradan alınan ardışık karelerin hareket vektörleri ile DVL'nin akustik hız okumaları birleştirilmektedir2.

## Derin Öğrenme Tabanlı Sualtı Görüntü İşleme ve Çıkarım

Görüntü işleme katmanında belirlenen "Klasik Yöntemlerden YOLOv8'e planlı geçiş" stratejisi yetersizdir. Küresel AUV yarışmalarında ve su altı endüstrisinde, kural tabanlı OpenCV algoritmaları (özellikle HSV renk ayrıştırması) yıllar önce terk edilmiştir. Suyun derinliğine ve bulanıklığına bağlı olarak anlık değişen fiziksel saçılım modelleri, ancak yapay sinir ağları ile modellenebilir18.

## Uçtan Uca Görüntü İyileştirme (Image Enhancement) Ağları

Geleneksel White Balance ve CLAHE yöntemleri, görüntüyü sadece optik olarak düzeltir; ancak bu işlemler piksellerdeki gürültüyü artırarak hedefin yapısal dokularını da tahrip eder19. Gelişmiş su altı hedef tespiti için sadece renk düzeltme değil, hedef tespit ağının ihtiyaç duyduğu

özellikleri (features) vurgulayan birleşik ağlar kullanılmaktadır. Örneğin, UDCP (Underwater Dark Channel Prior) veya Sea-Thru algoritmaları ortamın optik derinliğini matematiksel olarak modelleyerek ışık soğurulmasını fiziksel kurallara göre geri çevirir20. Günümüzde en ileri nokta, görüntü iyileştirme ile YOLO tabanlı nesne tespitini aynı derin öğrenme mimarisinde buluşturan EnYOLO veya UAED-Net (Unified Adaptive Enhancement and Detection Network) yapılarıdır18. Bu ağlar, ilk katmanlarında bulanıklık ve renk spektrumu kaybını telafi eden bir zenginleştirici modül barındırır. İyileştirilen yüksek özellikli pikseller doğrudan YOLO tespit bloğuna beslenir ve ağ uçtan uca (end-to-end) birleşik bir kayıp fonksiyonu (loss function) ile eğitilerek, geleneksel ayrık metotlara göre muazzam doğruluk

artışları sağlar18.


## Edge AI ve TensorRT Çıkarım (Inference) Optimizasyonu

YOLOv8, Faster-RCNN veya YOLO11-Pose gibi mimarilerin AUV'nin ana işlemcisinde gerçek zamanlı (latency olmadan) çalışabilmesi için salt Jetson Nano'nun gücü yeterli değildir6. Modern araştırmalarda YOLO modelleri veya deniz altı için özelleşmiş LS-YOLO (Lightweight Sonar YOLO) gibi algoritmalar kullanılmaktadır21.

Model, PyTorch ortamında eğitildikten sonra ONNX formatına dönüştürülür ve ardından NVIDIA TensorRT motoru kullanılarak FP32 (32-bit kayan nokta) yerine FP16 veya INT8 (8-bit tamsayı) formatına indirgenir (quantization). Bu optimizasyon işlemi sayesinde, 100 TOPS işlem gücüne sahip NVIDIA Jetson Orin NX üzerinde, bulanık kamera görüntüleri ve İleri Bakan Sonar (FLS) görüntüleri eş zamanlı olarak ortalama 45-72 FPS gibi yüksek bir hızda, sıfıra yakın bir kontrol döngüsü gecikmesiyle analiz edilir8. Yüksek FPS ile elde edilen hedef koordinatları, doğrudan SMC veya LQR güdüm katmanlarına iletilerek akıcı bir otonom izleme performansı oluşturur.

## İleri Düzey Otonomi İçin Donanım ve Malzeme Entegrasyon Matrisi

Yukarıda detaylandırılan teorik algoritmaların, davranış ağaçlarının, çoklu sensör EKF/UKF füzyonunun ve TensorRT tabanlı yapay sinir ağlarının çalıştırılabilmesi için, sistemin donanım katmanının belirli teknik sınırları aşması gerekmektedir. Küresel eğilimlere ve RoboSub yarışmalarında derece elde eden (örn. ITU, METU, NUS Bumblebee) başarılı üniversite sistemlerine dayanılarak5, AUV için zorunlu malzeme ve donanım bileşenleri aşağıda gerekçeleriyle tablolaştırılmıştır.

## 1. Hesaplama ve Kontrol Mimarisi (Compute & Control Core)

Kontrol yazılımlarının düşük gecikmeyle (deterministic) işlenmesi ve ROS 2 düğümlerinin hızlı iletişim kurabilmesi için yüksek kapasiteli, tekil işlemci altyapısı tercih edilmelidir.

| Malzeme / Bileşen | Önerilen Endüstri | Teknik Gerekçe / Özellik |
| --- | --- | --- |
|   | Standardı |   |
| Yapay Zeka İşlemcisi | NVIDIA Jetson Orin NX | Jetson Nano'nun sınırlı |
| (SBC) | (16 GB) | işlem gücü ve eski işletim |
|   |   | sistemi darboğazlarını aşar. |
|   |   | ROS 2 Humble ve Ubuntu |
|   |   | 22.04 LTS'i yerel olarak |
|   |   | çalıştırır. 100 TOPS işlem |
|   |   | gücü ile TensorRT tabanlı |
|   |   | UAED-Net veya YOLOX |
|   |   | algoritmalarını gerçek |
|   |   | zamanlı (>45 FPS) |
|   |   | çözümler6. |


| Gömülü Uçuş | Pixhawk Cube Orange+ | Titreşim izolasyonlu 3 |
| --- | --- | --- |
| Kontrolcüsü | (veya Blue Robotics | yedekli IMU (ivmeölçer, |
|   | Navigator) | jiroskop, barometre) |
|   |   | barındırır. ArduSub |
|   |   | yazılımını kusursuz |
|   |   | çalıştırarak, yüksek |
|   |   | frekanslı itici karıştırma |
|   |   | (motor mixing matrix) |
|   |   | işlemlerini Jetson'ın |
|   |   | üzerinden alır36. |
| Haberleşme Ağ Anahtarı | Blue Robotics Ethernet | Otonomi cihazları (DVL, IP |
| (Switch) | Switch | Kameralar, Ping Sonar ve |
|   |   | Jetson SBC) arasındaki veri |
|   |   | paketlerinin UDP/IP |
|   |   | üzerinden kayıpsız |
|   |   | yönlendirilmesini sağlayan |
|   |   | kompakt bir ağ |
|   |   | altyapısıdır27. |

## 2. Algılama ve Navigasyon Sensörleri (Perception & Navigation Sensors)

Genişletilmiş Kalman Filtresinin (EKF) drift (sapma) yapmadan konumlandırma yapabilmesi için akustik hız ve optik ortam algılama sensörlerine ihtiyaç duyulur.

| Malzeme / Bileşen | Önerilen Endüstri | Teknik Gerekçe / Özellik |
| --- | --- | --- |
|   | Standardı |   |
| Doppler Hız Kaydı (DVL) Water Linked DVL A50 |   | Sadece 66 mm çapı ve 170 |
|   |   | g ağırlığıyla dünyanın en |
|   |   | küçük DVL'sidir. 1 MHz |
|   |   | frekans ile akustik zemin |
|   |   | kilidi oluşturarak 5 cm ile 50 |
|   |   | m irtifa aralığında hız |
|   |   | ölçümü yapar27. EKF için |
|   |   | kritik bir veri kaynağıdır ve |
|   |   | maliyeti yaklaşık |
|   |   | $8,710-$10,580 |
|   |   | aralığındadır27. |
| İleri Bakan / Taramalı | Sonoptix ECHO veya | Derin öğrenme ile entegre |


| Sonar (FLS) | Ping360 Scanning Sonar | çalışarak bulanık su |
| --- | --- | --- |
|   |   | koşullarında hedeflerin |
|   |   | (boru hattı, geçiş kapıları) |
|   |   | kameranın işlevsiz kaldığı |
|   |   | durumlarda akustik |
|   |   | dalgalarla üç boyutlu |
|   |   | haritalanmasını ve |
|   |   | mesafesinin ölçülmesini |
|   |   | sağlar27. |
| Görüntüleme Sensörü | Blue Robotics Low-Light | Sualtında ışığın düşük |
| (Kamera) | HD USB/IP Camera | olduğu durumlara özel |
|   |   | düşük ışık performansı ile |
|   |   | optimize edilmiş sensör36. |
|   |   | Jetson'a donanımsal |
|   |   | H.264/H.265 sıkıştırmasıyla |
|   |   | veriyi ileterek işlemci |
|   |   | yükünü azaltır. |
| Yüksek Hassasiyetli | Bar30 High-Resolution | AUV'nin Z ekseni |
| Basınç Sensörü | 300m Depth Sensor | konumlandırılması ve |
|   |   | Kayan Mod Kontrolünün |
|   |   | (SMC) derinlik girdisi için |
|   |   | santimetre altı çözünürlük |
|   |   | ile mutlak su sütunu basıncı |
|   |   | ölçümü sağlar. |

## 3. Tahrik ve Sızdırmazlık Altyapısı (Actuation & Subsea Infrastructure)

Aracın doğrusal olmayan sönümleme kuvvetlerine ve okyanus akıntılarına karşı durabilmesi, aşırı itici kuvvet (over-actuation) ve yüksek izolasyon ile mümkündür.

| Malzeme / Bileşen | Önerilen Endüstri | Teknik Gerekçe / Özellik |
| --- | --- | --- |
|   | Standardı |   |
| Fırçasız İtici Motorlar | Blue Robotics T200 (8 | Aracın 6-DoF hareketlerini |
| (Thrusters) | Adet - Octocopter | stabil kılabilmek için yatay |
|   | Konfigürasyonu) | (4) ve dikey (4) iticilere |
|   |   | ayrılmış "over-actuated" bir |
|   |   | yapı gerekir. SMC veya |
|   |   | LQR algoritmalarının |
|   |   | manevra kararlılığını artırır3. |


| Elektronik Hız | Basic ESC 500 | Pixhawk üzerinden iletilen |
| --- | --- | --- |
| Kontrolcüler |   | PWM komutlarının |
|   |   | motorlara iletilmesindeki |
|   |   | elektronik darboğazları |
|   |   | ortadan kaldırarak yüksek |
|   |   | tepki hızı sunar. |
| Sızdırmaz Kablo ve | WetLink Penetrator & PUR | Su geçirmeyen O-ring |
| Penetratörler | Subsea Cable | destekli poliüretan |
|   |   | sızdırmazlık bağlantıları, |
|   |   | mekanik ve elektronik |
|   |   | bileşenler arasında |
|   |   | izolasyonu korurken sistem |
|   |   | modülerliğini destekler27. |
| Dinamik Aydınlatma | Lumen Subsea Light V2 | Kamera tabanlı yapay sinir |
| Sistemi | (1500-2000 lümen) | ağlarının renk |
|   |   | maskelemelerinde optik |
|   |   | kontrastı artırmak amacıyla |
|   |   | akıllı olarak ayarlanabilen |
|   |   | PWM kontrollü LED |
|   |   | altyapısıdır37. |

## Sonuç

Analize sunulan başlangıç niteliğindeki görev algoritmaları ve yazılım mimarisi, geleneksel su altı robotiğine giriş standartlarını sağlasa da, global yarışmalarda (örn. RoboSub)3 ve ileri otonomi projelerinde karşılaşılan zorluklar karşısında yetersiz kalmaktadır. Kullanılan donanımların uyumsuzluğu, Ziegler-Nichols PID gibi lineer sistem varsayımlarının doğrusal olmayan su altı dinamiği karşısındaki çaresizliği ve ölü hesaplama (dead reckoning) süreçlerinde DVL eksikliğinden kaynaklanan devasa sapmalar, sistemin saha operasyonlarında sürdürülebilirliğini engelleyecek temel hatalardır.

Çözüm stratejisi olarak, tüm yazılım altyapısının ROS 2 Humble çerçevesinde Docker konteynerleri kullanılarak DDS (Data Distribution Service) mimarisine taşınması gerekmektedir. Hedef algılama süreçlerinde manuel ayarlanan HSV eşikleri tamamen terk edilmeli; donanım hızlandırmalı NVIDIA Jetson Orin NX üzerinde çalıştırılacak olan TensorRT optimize edilmiş (FP16/INT8 quantizasyonlu) uçtan uca ağlara (UAED-Net, LS-YOLO) geçiş yapılmalıdır. Kontrol katmanında ise AUV hidrodinamiğini tolere edebilecek Kayan Mod Kontrolü (SMC) veya Bulanık Uyumlu (Fuzzy Adaptive) PID uygulanmalı; araç hızı ve pozisyonu mutlaka robot_localization EKF algoritmaları üzerinden Water Linked DVL A50 akustik sensörü ve IMU füzyonu ile desteklenmelidir.

Belirtilen bu kavramsal, donanımsal ve algoritmik revizyonların eksiksiz uygulanması, aracın en


türbülanslı su şartlarında dahi otonomisini koruyarak rekabetçi düzeyde hatasız görevler icra

etmesini sağlayacaktır.

## Alıntılanan çalışmalar

- 1. Sensor Fusion Using Error-State Kalman Filter to Improve Localization of Autonomous Underwater Vehicle Under DVL Signal Loss | Request PDF - ResearchGate,

https://www.researchgate.net/publication/375847441_Sensor_Fusion_Using_Error -State_Kalman_Filter_to_Improve_Localization_of_Autonomous_Underwater_Veh [URL 🔗](https://www.researchgate.net/publication/375847441_Sensor_Fusion_Using_Error-State_Kalman_Filter_to_Improve_Localization_of_Autonomous_Underwater_Vehicle_Under_DVL_Signal_Loss)

[icle_Under_DVL_Signal_Loss](https://www.researchgate.net/publication/375847441_Sensor_Fusion_Using_Error-State_Kalman_Filter_to_Improve_Localization_of_Autonomous_Underwater_Vehicle_Under_DVL_Signal_Loss)

- 2. Underwater Visual Acoustic SLAM with Extrinsic Calibration | Request PDF - ResearchGate, https://www.researchgate.net/publication/357105409_Underwater_Visual_Acousti c_SLAM_with_Extrinsic_Calibration [URL 🔗](https://www.researchgate.net/publication/357105409_Underwater_Visual_Acoustic_SLAM_with_Extrinsic_Calibration)

- 3. RoboSub 2026: Pushing The Frontier - Mecatron NTU, https://mecatron.sg/robosub_2026/Mecatron-Technical-Design-Report-2026.pdf [URL 🔗](https://mecatron.sg/robosub_2026/Mecatron-Technical-Design-Report-2026.pdf)

- 4. BRACU Duburi AUV Technical Design Report - RoboNation, https://robonation.org/app/uploads/sites/4/2025/07/RS25_TDR_BRAC-University- Duburi_compressed.pdf

- 5. RoboSub 2025 Technical Design Report - RoboNation, https://robonation.org/app/uploads/sites/4/2025/07/RS25_TDR_National-Universit y-of-Singapore-Bumblebee-compressed.pdf

- 6. MM Nautronics RoboSub 2026 Technical Design Report - RoboNation, https://robonation.org/app/uploads/sites/4/2026/07/RS26_TDR_MiddleEastTechni calUniversity.pdf

- 7. OSU UWRT Talos AUV Design Review | PDF | Art | Computers - Scribd, https://www.scribd.com/document/781186968/TDR-THEOhioStateUniversity-RS20 23-Compressed

- 8. Drones and Unmanned Systems - Sensors Portal, https://sensorsportal.com/DOWNLOADS/DAUS_2026_Proceedings.pdf [URL 🔗](https://sensorsportal.com/DOWNLOADS/DAUS_2026_Proceedings.pdf)

- 9. AUV-IITK/Non_Linear_ROV_PID_MATLAB: Matlab simulation for the non-linear PID controller for ROV - GitHub, https://github.com/AUV-IITK/Non_Linear_ROV_PID_MATLAB

- 10. Feedback control systems for underwater vehicles - Fiveable, https://fiveable.me/underwater-robotics/unit-8/feedback-control-systems-underwat er-vehicles/study-guide/VUEMufrhIzxc5nAs [URL 🔗](https://fiveable.me/underwater-robotics/unit-8/feedback-control-systems-underwater-vehicles/study-guide/VUEMufrhIzxc5nAs)

- 11. MODELING AND CONTROL OF A FULLY ACTUATED UNMANNED SURFACE VEHICLE A THESIS SUBMITTED TO THE GRADUATE SCHOOL OF NATURAL AND APPLI, https://open.metu.edu.tr/bitstream/handle/11511/103234/index.pdf [URL 🔗](https://open.metu.edu.tr/bitstream/handle/11511/103234/index.pdf)

- 12. Design and Control of Torpedo-Shaped Unmanned Underwater Vehicle based on PID Controller with Environmental Disturbances, https://accesson.kr/spacenote/assets/pdf/55937/journal-2-1-11.pdf [URL 🔗](https://accesson.kr/spacenote/assets/pdf/55937/journal-2-1-11.pdf)

- 13. Robust Stabilization of an Autonomous Underwater Vehicle in Depth, https://www.propulsiontechjournal.com/index.php/journal/article/download/2284/15 [URL 🔗](https://www.propulsiontechjournal.com/index.php/journal/article/download/2284/1540/3913)


- 14. Fuzzy Adaptive PID-Based Tracking Control for Autonomous Underwater Vehicles 40/3913 - MDPI, https://www.mdpi.com/2076-0825/14/10/470 [URL 🔗](https://www.mdpi.com/2076-0825/14/10/470)

- 15. Sliding Mode Based Depth Control of an Autonomous Underwater Vehicle (AUV), https://www.researchgate.net/publication/315765741_Sliding_Mode_Based_Depth _Control_of_an_Autonomous_Underwater_Vehicle_AUV [URL 🔗](https://www.researchgate.net/publication/315765741_Sliding_Mode_Based_Depth_Control_of_an_Autonomous_Underwater_Vehicle_AUV)

- 16. Identifying IMU Noise Characteristics from USV Trajectories | Request PDF - ResearchGate, https://www.researchgate.net/publication/394607336_Identifying_IMU_Noise_Cha racteristics_from_USV_Trajectories

- 17. A Low-Cost and High-Precision Underwater Integrated Navigation System - ResearchGate, [URL 🔗](https://www.researchgate.net/publication/377655193_A_Low-Cost_and_High-Precision_Underwater_Integrated_Navigation_System)

[https://www.researchgate.net/publication/377655193_A_Low-Cost_and_High-Prec](https://www.researchgate.net/publication/377655193_A_Low-Cost_and_High-Precision_Underwater_Integrated_Navigation_System)

[ision_Underwater_Integrated_Navigation_System](https://www.researchgate.net/publication/377655193_A_Low-Cost_and_High-Precision_Underwater_Integrated_Navigation_System)

- 18. EnYOLO: A Real-Time Framework for Domain-Adaptive Underwater Object Detection with Image Enhancement | Request PDF - ResearchGate, https://www.researchgate.net/publication/382989651_EnYOLO_A_Real-Time_Fra mework_for_Domain-Adaptive_Underwater_Object_Detection_with_Image_Enha ncement [URL 🔗](https://www.researchgate.net/publication/382989651_EnYOLO_A_Real-Time_Framework_for_Domain-Adaptive_Underwater_Object_Detection_with_Image_Enhancement)

- 19. A Real-Time Framework for Domain-Adaptive Underwater Object Detection with Image Enhancement - arXiv, https://arxiv.org/html/2403.19079v1

- 20. Aqua-Vision: The Future of Underwater Exploration and Innovation - IJIRAE, https://www.ijirae.com/volumes/Vol12/iss-10/10.OCAE10093.pdf [URL 🔗](https://www.ijirae.com/volumes/Vol12/iss-10/10.OCAE10093.pdf)

- 21. A Lightweight Underwater Target Detection Network for Forward-Looking Sonar Images | Request PDF - ResearchGate, https://www.researchgate.net/publication/382136051_A_Lightweight_Underwater_ Target_Detection_Network_for_Forward-looking_Sonar_Images [URL 🔗](https://www.researchgate.net/publication/382136051_A_Lightweight_Underwater_Target_Detection_Network_for_Forward-looking_Sonar_Images)

- 22. Papers | Data Distribution Service (DDS) Community RTI Connext Users, https://community.rti.com/dds_papers

- 23. Nav2 — Nav2 1.0.0 documentation, https://docs.nav2.org/ [URL 🔗](https://docs.nav2.org/)

- 24. FULL DUPLEX HYBRID ACOUSTIC/RF COMMUNICATION FOR UNDERWATER NETWORKED CONTROL SYSTEMS by SAEED NOURIZADEH AZAR Submitted to the - Sabanci University Research Database, https://research.sabanciuniv.edu/47500/1/10531965.pdf [URL 🔗](https://research.sabanciuniv.edu/47500/1/10531965.pdf)

- 25. Geometric line-of-sight guidance law with exponential switching sliding mode control for marine vehicles' path following - PMC, https://pmc.ncbi.nlm.nih.gov/articles/PMC12229861/ [URL 🔗](https://pmc.ncbi.nlm.nih.gov/articles/PMC12229861/)

- 26. The Design of Team VORTEX's 2021 AUV - RoboNation, https://robonation.org/app/uploads/sites/4/2021/07/RoboSub_2021_Vortex-Alexan dria.pdf [URL 🔗](https://robonation.org/app/uploads/sites/4/2021/07/RoboSub_2021_Vortex-Alexandria.pdf)

- 27. DVL Doppler Velocity Log for ROVs and AUVs - Blue Robotics, https://bluerobotics.com/store/the-reef/dvl-a50/

- 28. Chasing - Waterlink DVL kit - Blue Skies Drones, https://www.blueskiesdroneshop.com/products/chasing-waterlinked-dvl-kit

- 29. DVL Doppler Velocity Log - invocean, [URL 🔗](https://www.blueskiesdroneshop.com/products/chasing-waterlinked-dvl-kit)


- [https://www.invoceangroup.com/dopplervelocitylog-dvl](https://www.invoceangroup.com/dopplervelocitylog-dvl)

- 30. Water Linked launches: DVL A50 Doppler Velocity Log - Blue Robotics Community Forums, https://discuss.bluerobotics.com/t/water-linked-launches-dvl-a50-doppler-velocity-l [URL 🔗](https://discuss.bluerobotics.com/t/water-linked-launches-dvl-a50-doppler-velocity-log/7142)

[og/7142](https://discuss.bluerobotics.com/t/water-linked-launches-dvl-a50-doppler-velocity-log/7142)

- 31. YOLO-CAB: An Efficient Deep Learning-Based Underwater Object Detection Method for Autonomous Underwater Vehicles - MDPI, https://www.mdpi.com/2227-7390/14/11/1927 [URL 🔗](https://www.mdpi.com/2227-7390/14/11/1927)

- 32. AO-UOD: A Novel Paradigm for Underwater Object Detection Using Acousto–Optic Fusion, https://www.researchgate.net/publication/390079527_AO-UOD_A_Novel_Paradig [URL 🔗](https://www.researchgate.net/publication/390079527_AO-UOD_A_Novel_Paradigm_for_Underwater_Object_Detection_Using_Acousto-Optic_Fusion)

[m_for_Underwater_Object_Detection_Using_Acousto-Optic_Fusion](https://www.researchgate.net/publication/390079527_AO-UOD_A_Novel_Paradigm_for_Underwater_Object_Detection_Using_Acousto-Optic_Fusion)

- 33. Some examples of sonar images detection and instance segmentation... | Download Scientific Diagram - ResearchGate, https://www.researchgate.net/figure/Some-examples-of-sonar-images-detection-a nd-instance-segmentation-results-For-showing_fig9_348612301 [URL 🔗](https://www.researchgate.net/figure/Some-examples-of-sonar-images-detection-and-instance-segmentation-results-For-showing_fig9_348612301)

- 34. Multi-Scale Marine Object Detection in Side-Scan Sonar Images Based on BES-YOLO, [URL 🔗](https://www.researchgate.net/publication/382136616_Multi-Scale_Marine_Object_Detection_in_Side-Scan_Sonar_Images_Based_on_BES-YOLO)

[https://www.researchgate.net/publication/382136616_Multi-Scale_Marine_Object_](https://www.researchgate.net/publication/382136616_Multi-Scale_Marine_Object_Detection_in_Side-Scan_Sonar_Images_Based_on_BES-YOLO)

[Detection_in_Side-Scan_Sonar_Images_Based_on_BES-YOLO](https://www.researchgate.net/publication/382136616_Multi-Scale_Marine_Object_Detection_in_Side-Scan_Sonar_Images_Based_on_BES-YOLO)

- 35. Student - İTÜ Haberler, https://haberler.itu.edu.tr/en/student [URL 🔗](https://haberler.itu.edu.tr/en/student)

- 36. BlueRoboticsStore - AQUA Exploración, https://aquaexploracion.com/rov/blueroboticsstore/ [URL 🔗](https://aquaexploracion.com/rov/blueroboticsstore/)

- 37. ROV BlueROV2 Blueye X3 M2 PRO Revolution Pivot Photon ROV-500 ROV-3000 Fifish Pro W6 SRV-8 Falcon Chinook Fusion Defender Pro5 I - Unmanned Systems Technology, [URL 🔗](https://aquaexploracion.com/rov/blueroboticsstore/)

https://www.unmannedsystemstechnology.com/wp-content/uploads/2021/05/Small -ROVs-Comparison.pdf [URL 🔗](https://www.unmannedsystemstechnology.com/wp-content/uploads/2021/05/Small-ROVs-Comparison.pdf)

| Kategori | Endüstri | Fiyat | Bütçe Dostu | Fiyat |
| --- | --- | --- | --- | --- |
|   | Standardı (En |   | (F/P) Alternatif |   |
|   | İyi Sonuç) |   |   |   |
| Yapay | NVIDIA Jetson | $899+ | NVIDIA Jetson | $249 - |
| Zeka | Orin NX |   | Orin Nano 8GB | $399 |
| İşlemcisi |   |   |   |   |
| Ağ | Blue Robotics | $185 | Standart OEM | Belirsiz |
| Anahtarı |   |   | Mini Switch |   |


| (Switch) | Ethernet Switch |   | (Modifiyeli) |   |
| --- | --- | --- | --- | --- |
| Hız ve | Water Linked | $8,710 - | Sadece gelişmiş | - |
| Konum | DVL A50 | $10,580 | IMU+Kamera |   |
| (DVL) |   |   | (Riskli) |   |
| Çevresel | Ping360 | $2,750 | Sonoptix ECHO | Modele |
| Engel | Scanning Sonar |   | Multibeam | Göre |
| Sonarı |   |   |   |   |
| Derinlik | Bar30 (MS5837) | $80 - $90 Bar30 MS5837 |   | $45 |
| Sensörü | Yüksek |   | (Bağımsız Satıcı) |   |
|   | Çözünürlüklü |   |   |   |
