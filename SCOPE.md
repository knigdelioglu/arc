# Arc — MVP Scope

## 1. Ürün tanımı

Arc, iPhone ve iPad üzerinde PDF/EPUB belgelerini Türkçe seslendiren, tamamen cihaz üzerinde çalışan bir sesli okuma uygulamasıdır.

MVP'nin temel teknik farkı, TTS modelini dinleme süresinin tamamı boyunca sürekli çalıştırmak yerine **ileriden üretim yapan adaptif ses cache'i** kullanmasıdır.

Ana TTS motoru: **Supertonic 3**  
Inference hedefi: **Core ML / Apple Neural Engine uyumlu yürütme**  
Playback: **AVFoundation tabanlı background audio**

---

## 2. MVP hedefleri

MVP aşağıdakileri yapabilmelidir:

1. Kullanıcı cihazından EPUB veya metin katmanlı PDF içe aktarabilmeli.
2. Belgeden okunabilir metin çıkarılmalı.
3. Başlık, bölüm, paragraf ve cümle sınırları mümkün olduğunca korunmalı.
4. Türkçe metin Supertonic 3 ile cihaz üzerinde seslendirilmelidir.
5. Üretilen ses küçük yerel segmentler halinde cache'e yazılmalıdır.
6. Kullanıcı ilk ses hazır olur olmaz dinlemeye başlayabilmelidir.
7. Uygulama ilerideki metni adaptif biçimde önceden üretmelidir.
8. Buffer yeterliyken TTS modeli aktif tutulmamalıdır.
9. Ekran kilitlendiğinde ve uygulama arka plana geçtiğinde ses oynatma devam etmelidir.
10. Kullanıcı ileri/geri sarabilmeli ve kaldığı yerden devam edebilmelidir.
11. Uygulama internet bağlantısı olmadan çalışabilmelidir.
12. Kitap metni ve oluşturulan ses cihaz dışına gönderilmemelidir.

---

## 3. MVP dışı

Aşağıdakiler ilk sürümün parçası değildir:

- Android desteği
- bulut TTS
- hesap / kullanıcı girişi
- cloud sync
- sosyal özellikler
- kitap mağazası
- DRM kırma veya korumalı EPUB/PDF desteği
- taranmış PDF OCR
- DOCX desteği
- web sayfasından makale çekme
- LLM ile özetleme, soru-cevap veya içerik dönüştürme
- voice cloning
- özel kullanıcı sesi eğitme
- çok kullanıcılı profil sistemi

Bunlar sonraki fazlarda ayrıca değerlendirilir.

---

## 4. Platform kapsamı

### MVP

- iOS
- iPadOS

### İlk geliştirme hedefi

- modern Neural Engine içeren iPhone/iPad cihazları

Uygulama yalnız ANE'ye sabitlenmemelidir. Core ML yürütme politikası ve gerçek cihaz uyumluluğu ölçülerek uygun compute-unit fallback'i bulunmalıdır. ANE kullanımı hedef olsa da belirli bir katmanın ANE üzerinde çalışacağı varsayımı kodun iş mantığına gömülmemelidir.

---

## 5. Belge desteği

### EPUB

MVP gereksinimleri:

- EPUB paketini açma
- manifest/spine sırasını takip etme
- XHTML içeriğini metne dönüştürme
- başlık/bölüm yapısını koruma
- dipnot, menü, tekrar eden navigasyon ve anlamsız markup'ı mümkün olduğunca temizleme

### PDF

MVP:

- PDFKit ile metin katmanlı PDF okuma
- sayfa sırasını koruma
- tekrar eden header/footer temizliği için temel heuristics
- kelime bölünmelerini ve satır sonlarını normalize etme

MVP'de taranmış/görüntü tabanlı PDF zorunlu değildir.

---

## 6. Metin pipeline'ı

```text
Document
  → extractor
  → structural parser
  → text normalizer
  → chapter/paragraph model
  → sentence-aware segmenter
  → render queue
```

Normalizer aşağıdaki sorunları ele almalıdır:

- gereksiz satır sonları
- PDF kaynaklı kelime bölünmeleri
- ardışık whitespace
- sayfa numaraları
- tekrar eden header/footer
- bozuk Unicode
- Türkçe noktalama
- kısaltmaların cümle bölme hataları

Metin normalizasyonu deterministik olmalıdır. Aynı belge ve aynı uygulama sürümünde aynı normalize edilmiş metin üretilmelidir; bu cache anahtarlarının kararlı olmasını sağlar.

---

## 7. TTS katmanı

Uygulama çekirdeği Supertonic'e doğrudan bağımlı olmamalıdır.

Önerilen sınır:

```swift
protocol SpeechSynthesizing {
    func prepare() async throws
    func synthesize(_ request: SpeechRequest) async throws -> SpeechAudio
    func releaseResources() async
}
```

İlk implementasyon:

```text
Supertonic3CoreMLSynthesizer
```

Bu ayrım sayesinde model dönüştürme biçimi, Core ML graph yapısı veya gelecekteki başka bir local TTS backend'i uygulamanın geri kalanını değiştirmeden güncellenebilir.

### TTS gereksinimleri

- Türkçe dil kodu
- seçilebilir voice/style
- deterministik veya yeterince kararlı çıktı
- model sürüm bilgisinin erişilebilir olması
- cancellation desteği
- thermal/memory pressure durumunda güvenli iptal
- sentez metriği toplama:
  - giriş karakter/token miktarı
  - üretilen ses süresi
  - sentez süresi
  - RTF
  - peak memory mümkünse

---

## 8. Render-ahead scheduler

MVP'nin kritik bileşenidir.

### Hedef davranış

Model sürekli açık kalmaz.

```text
buffer düşük
   ↓
model yükle
   ↓
ilerideki segmentleri üret
   ↓
high-water mark'e ulaş
   ↓
model kaynaklarını bırak
   ↓
cache'ten oynat
   ↓
buffer yeniden düşük
   ↓
tekrar et
```

### İlk tuning değerleri

Başlangıçta:

- startup için ilk oynatılabilir pencere: yaklaşık 30–60 saniye
- high-water target: yaklaşık 10–15 dakika
- low-water target: yaklaşık 4–5 dakika

Bunlar sabit ürün değerleri değildir. Scheduler cihazın gerçek performansını öğrendikçe uyarlanmalıdır.

### Dinamik eşik

Scheduler şu değeri tahmin etmelidir:

```text
estimatedGenerationTime = requestedAudioDuration × measuredRTF
```

Low-water eşiği için başlangıç formülü:

```text
lowWater >= estimatedGenerationTime(nextFill) × safetyFactor + fixedMargin
```

Önerilen başlangıç:

```text
safetyFactor = 2.0
fixedMargin = 30–60 s
minimumLowWater = 120 s
```

Amaç, thermal throttling veya beklenmedik yavaşlama olduğunda bile playback underrun yaşamamaktır.

### Önemli örnek

RTF `0.15` ise:

- 20 dakika ses ≈ 3 dakika sentez
- yeni üretime 19. dakikada başlamak güvenli değildir

Bu nedenle Arc sabit “19. dakika” gibi tetikleyiciler kullanmayacaktır.

---

## 9. Sentez batch'i ile cache segmenti ayrı kavramlardır

Arc 10 dakikalık bir pencereyi tek üretim işi olarak ele alabilir; ancak bunu tek 10 dakikalık dosya halinde saklamak zorunda değildir.

Öneri:

- scheduler batch: birkaç dakika
- text segment: cümle/paragraf sınırları
- persistent audio segment: yaklaşık 30–90 saniye

Avantajları:

- hızlı seek
- küçük invalidation alanı
- voice değişiminde kısmi yeniden üretim
- bozuk/yarım dosyada daha az kayıp
- kaldığı yerden devamın kolaylaşması

---

## 10. Audio cache

### Dosya biçimi

MVP için hedef:

- M4A container
- AAC tabanlı konuşma odaklı encoding
- mono

Ham WAV yalnız debug/benchmark amacıyla kullanılabilir; kalıcı cache formatı olmamalıdır.

### Cache key

Her segment en az şu girdilerle kimliklendirilmelidir:

```text
documentContentHash
chapterID
segmentID
normalizedTextHash
language
voiceStyleID
modelVersion
synthesisParameters
cacheFormatVersion
```

### Cache invalidation

Şunlardan biri değişirse ilgili segment yeniden üretilir:

- metin
- voice/style
- model sürümü
- sentez parametreleri
- cache format sürümü

### Cache politikası

- aktif kitap cache'i korunur
- eski kitapların cache'i LRU benzeri politika ile temizlenebilir
- kullanıcı isterse bir kitabın tamamını önceden seslendirebilir
- kullanıcı cache'i manuel temizleyebilir
- cache temizlenmesi kitap/progress verisini silmemelidir

---

## 11. Playback engine

Playback TTS modelinden bağımsız çalışmalıdır.

Gereksinimler:

- gapless veya kullanıcı tarafından fark edilmeyecek segment geçişi
- play / pause
- ±15/30 saniye seek
- bölüm atlama
- playback rate
- interruption recovery
- Bluetooth/kulaklık route değişimleri
- telefon görüşmesi sonrası uygun resume davranışı
- lock-screen controls
- Control Center controls
- Now Playing metadata
- elapsed time ve chapter bilgisi

Tek bir uzun ses dosyasına bağımlı bir playback tasarımı yapılmamalıdır.

---

## 12. Background davranışı

### Zorunlu

- `AVAudioSession.Category.playback`
- Background Modes → Audio
- ekran kilitliyken oynatma

### Mimari kural

Arka planda sınırsız ML inference süresi **garanti kabul edilmemelidir**.

Arc'ın devamlılığı şu sırayla sağlanır:

1. önceden oluşturulmuş buffer,
2. aktif medya playback session'ı,
3. uygun olduğunda opportunistic background processing.

BackgroundTasks API'leri cache'i genişletmek veya önceden render etmek için değerlendirilebilir, ancak sistemin bu işleri istenen saniyede başlatacağı varsayılmamalıdır.

Playback'in devam etmesi yalnız BGProcessingTask'e bağlanmamalıdır.

---

## 13. Model lifecycle

TTS kaynakları gerektiğinde yüklenmeli, buffer yeterli olduğunda bırakılabilmelidir.

`releaseResources()` sonrası hedef:

- aktif synthesis task kalmaması
- büyük tensor/intermediate referanslarının tutulmaması
- MLModel graph referanslarının uygulama tarafından gereksiz tutulmaması
- cache playback'in modelden bağımsız devam etmesi

Not: iOS/Core ML bazı compiled model sayfalarını veya sistem cache'lerini kendi politikasıyla tutabilir. “Modeli kapat” ifadesi işletim sisteminin tüm model belleğini anında sıfırlayacağı garantisi anlamına gelmez.

Modeli her 30 saniyede yükleyip boşaltmak da verimsiz olabilir. Bu yüzden lifecycle, birkaç dakikalık batch üretimi etrafında tasarlanmalıdır.

---

## 14. Thermal ve enerji politikası

Scheduler aşağıdakileri izlemelidir:

- `ProcessInfo.processInfo.thermalState`
- Low Power Mode
- battery state mümkün olduğunda
- sentez RTF trendi

Önerilen davranış:

### normal

Hedef buffer'ı doldur.

### serious thermal state

- batch boyutunu küçült
- high-water hedefini azalt
- gereksiz pre-render işlerini durdur

### critical

- zorunlu olmayan sentezi durdur
- hazır cache'ten oynatmaya devam et

MVP'de pil yüzdesine dayalı agresif otomasyon şart değildir; önce gerçek cihaz benchmark'ı yapılmalıdır.

---

## 15. Veri modeli

En az:

### Book

- id
- title
- author
- sourceType
- sourceFingerprint
- importedAt
- cover reference

### Chapter

- id
- bookID
- order
- title
- normalizedText

### TextSegment

- id
- chapterID
- order
- text
- textHash
- estimatedStart

### AudioSegment

- id
- textSegment range
- cacheKey
- localURL
- duration
- synthesisDuration
- modelVersion
- voiceStyleID
- status

### ReadingProgress

- bookID
- chapterID
- segmentID
- offset
- updatedAt

---

## 16. Önerilen modüller

```text
ArcApp
│
├── Library
│   ├── BookRepository
│   └── ProgressRepository
│
├── Documents
│   ├── DocumentImporter
│   ├── EPUBExtractor
│   ├── PDFExtractor
│   ├── TextNormalizer
│   └── TextSegmenter
│
├── Speech
│   ├── SpeechSynthesizing
│   ├── Supertonic3CoreMLSynthesizer
│   ├── RenderScheduler
│   ├── SynthesisMetrics
│   └── ModelLifecycleController
│
├── Audio
│   ├── AudioCache
│   ├── AudioEncoder
│   ├── PlaybackEngine
│   └── NowPlayingController
│
└── UI
    ├── Library
    ├── Reader
    ├── Player
    └── Settings
```

---

## 17. Concurrency kuralları

- aynı audio segment iki kez paralel üretilmemeli
- render queue tek authoritative scheduler tarafından yönetilmeli
- kullanıcı seek yaptığında artık gereksiz olan düşük öncelikli işler cancel edilebilmeli
- kullanıcı başka bölüme atlarsa yeni konum öncelik kazanmalı
- model lifecycle ile synthesis task'ları yarış durumuna girmemeli
- yarım üretilmiş dosya final cache dosyası olarak işaretlenmemeli

Dosya yazımı:

```text
segment.tmp
   ↓ successful encode + fsync/close
atomic rename
   ↓
segment.m4a
```

---

## 18. Seek davranışı

Kullanıcı henüz render edilmemiş bir konuma atlarsa:

1. mevcut gereksiz prefetch işi iptal edilir veya düşük önceliğe alınır,
2. hedef konumun küçük startup segmenti yüksek öncelikle üretilir,
3. playback başlar,
4. yeni konumdan ileri buffer doldurulur.

Kullanıcı hiçbir zaman tüm bölümün üretilmesini beklemek zorunda bırakılmamalıdır.

---

## 19. Offline model dağıtımı

MVP'de iki seçenek teknik olarak değerlendirilecektir:

### A — model uygulamayla paketli

Artı:
- ilk açılışta doğrudan offline

Eksi:
- uygulama boyutu büyür

### B — ilk kullanımda model indir

Artı:
- App Store binary küçülür

Eksi:
- ilk model kurulumu internet ister

Ürün kararı model paket boyutu, lisans ve App Store dağıtım davranışı doğrulandıktan sonra verilir.

Model dosyaları Git geçmişine doğrudan eklenmemelidir.

---

## 20. Gizlilik ve güvenlik

- belge içeriği varsayılan olarak yalnız app sandbox'ında işlenir
- cloud endpoint zorunlu değildir
- oluşturulan audio cache paylaşılmaz
- debug log'larında kitap metni tutulmamalıdır
- crash/analytics entegrasyonu eklenirse kitap içeriği ve dosya yolları redakte edilmelidir

---

## 21. MVP UI

### Library

- kitap ekle
- kitap listesi
- kapak/başlık/yazar
- ilerleme
- kitap sil

### Reader / Player

- başlık ve bölüm
- play/pause
- ileri/geri sar
- bölüm seç
- hız
- voice/style
- kalan/ilerleme bilgisi
- buffer/render durumu yalnız gerektiğinde sade biçimde gösterilebilir

### Settings

- varsayılan voice
- playback rate
- cache limiti
- cache temizle
- model bilgisi
- gizlilik bilgisi

---

## 22. Ölçüm ve benchmark

MVP geliştirilirken gerçek cihazda şu değerler kaydedilmelidir:

- model load süresi
- time-to-first-playable-audio
- RTF
- dakika ses başına enerji etkisi
- active synthesis memory
- model release sonrası memory
- thermal state değişimleri
- 30/60/120 dakikalık playback'te underrun sayısı
- cache disk kullanımı

Bu metrikler görülmeden 10/15/20 dakikalık buffer değerleri kalıcı ürün kararı yapılmamalıdır.

---

## 23. MVP kabul kriterleri

MVP tamamlandı sayılabilmesi için:

- [ ] EPUB içe aktarılıyor ve doğru sırada okunuyor.
- [ ] Metin katmanlı PDF içe aktarılıyor ve okunuyor.
- [ ] Türkçe Supertonic 3 Core ML sentezi cihaz üzerinde çalışıyor.
- [ ] İlk ses hazır olduğunda tüm kitabı beklemeden playback başlıyor.
- [ ] Render-ahead scheduler cache'i otomatik dolduruyor.
- [ ] Buffer yeterliyken synthesis/model kaynakları bırakılabiliyor.
- [ ] Cache'ten playback TTS motorundan bağımsız devam ediyor.
- [ ] Ekran kilitliyken playback devam ediyor.
- [ ] Lock-screen medya kontrolleri çalışıyor.
- [ ] Seek sonrası hedef bölge öncelikli render ediliyor.
- [ ] Uygulama kapatılıp açıldığında okuma konumu korunuyor.
- [ ] İnternet kapalıyken daha önce kurulmuş modelle belge okunabiliyor.
- [ ] Yarım/bozuk cache segmenti playback kuyruğuna girmiyor.
- [ ] Voice/model değişimi doğru cache invalidation oluşturuyor.
- [ ] En az bir gerçek iPhone üzerinde 60 dakikalık kesintisiz playback testi tamamlanıyor.

---

## 24. Sonraki faz adayları

- Vision OCR ile taranmış PDF
- DOCX / TXT / Markdown
- Share Extension
- Siri / App Intents
- CarPlay değerlendirmesi
- tüm kitabı şarjdayken pre-render etme
- otomatik cache boyutu yönetimi
- iCloud ile yalnız progress sync
- farklı local TTS backend'leri
- Android sürümü

---

## 25. Mimari karar özeti

Arc'ın temel prensibi:

> **Sentez ile playback birbirinden ayrıdır.**

Supertonic 3 sesi üretir ve cache'e yazar. Kullanıcı mümkün olduğunca önceden üretilmiş ses segmentlerini dinler. Model yalnız buffer ihtiyacı olduğunda devreye girer.

Bu yaklaşım MVP'nin enerji, RAM, background dayanıklılığı ve seek davranışı açısından varsayılan mimarisidir.
