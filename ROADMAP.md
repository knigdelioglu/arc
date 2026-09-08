# Arc — Development Roadmap

Bu roadmap, Arc'ın MVP'ye mümkün olan en düşük teknik riskle ulaşması için hazırlanmıştır. Fazlar takvim tarihine değil **çıkış kriterlerine** bağlıdır. Bir fazın kritik kabul kriteri karşılanmadan sonraki faz, önceki varsayımı gizleyecek ölçüde büyütülmemelidir.

Arc'ın temel ürün ilkesi:

> **Önce üret, yerel cache'e yaz, cache'ten oynat; TTS modelini yalnız gerektiğinde çalıştır.**

Ana hedef platformlar: **iOS / iPadOS**  
TTS: **Supertonic 3**  
Inference: **Core ML, ANE öncelikli fakat ölçümle doğrulanan compute-unit seçimi**  
Playback: **AVFoundation / background audio**

---

## Yol haritası özeti

```text
R0  Teknik risk doğrulama
 ↓
R1  Uygulama çekirdeği ve proje iskeleti
 ↓
R2  Supertonic 3 Core ML dikey dilimi
 ↓
R3  Segmentleme + kalıcı audio cache
 ↓
R4  Adaptif render-ahead scheduler
 ↓
R5  EPUB pipeline
 ↓
R6  PDF pipeline
 ↓
R7  Player + background / lock-screen deneyimi
 ↓
R8  Library + progress + ürün UI
 ↓
R9  Dayanıklılık, enerji ve performans tuning
 ↓
R10 MVP release candidate
```

---

# R0 — Teknik risk doğrulama

## Amaç

Uygulamanın geri kalanını yazmadan önce Arc'ın üç temel varsayımını gerçek cihazda doğrulamak:

1. Supertonic 3 Türkçe sesi Core ML üzerinden güvenilir biçimde üretilebiliyor mu?
2. Seçilen graph gerçekten hedef Apple donanımında yeterince hızlı ve enerji verimli mi?
3. Hazır ses cache'ten oynatılırken ekran kilitli / uygulama arka planda kesintisiz medya deneyimi sağlanabiliyor mu?

## İşler

- Minimal Swift/iOS spike proje oluştur.
- Supertonic 3 model varlıklarını Core ML pipeline'ına bağla.
- Türkçe sabit test metni oluştur.
- Aşağıdaki compute politikalarını ölç:
  - `.all`
  - `.cpuAndNeuralEngine`
  - gerekiyorsa `.cpuAndGPU`
- Model yükleme süresini ölç.
- İlk ses oluşma süresini ölç.
- 1, 5 ve 20 dakikalık eşdeğer metinlerde sentez ölçümü yap.
- RTF hesapla:

```text
RTF = synthesisDuration / generatedAudioDuration
```

- Peak memory / memory footprint gözlemi yap.
- thermal state değişimini kaydet.
- sentez bittikten sonra model referanslarını bırak ve bellek davranışını gözlemle.
- M4A'ya yazılmış örnek sesi AVFoundation ile oynat.
- ekranı kilitle ve playback'in devam ettiğini doğrula.

## Ölçüm kaydı

Her benchmark en az şunları kaydetmeli:

```text
device
OS version
app build
model version
compute units
text length
generated audio duration
model load time
synthesis time
RTF
memory before/load/peak/after release
thermal state before/after
Low Power Mode
```

## Kritik not

ANE hedefimizdir, fakat **"Core ML kullanıyor = ANE üzerinde çalışıyor" varsayımı yapılmayacaktır**. Gerçek compute yerleşimi ve performans ölçülmeden mimariye ANE garantisi yazılmamalıdır.

## Çıkış kriteri

- Türkçe ses deterministik biçimde üretilebiliyor.
- Üretilen ses dosyaya yazılıp tekrar oynatılabiliyor.
- Gerçek cihaz RTF ölçülebiliyor.
- TTS playback thread'inden bağımsız.
- Ekran kilitliyken cache'ten playback devam ediyor.
- Kullanılacak Core ML model paketi / conversion yolu kararlaştırılmış.

---

# R1 — Uygulama çekirdeği ve proje iskeleti

## Amaç

TTS, belge işleme, cache ve player'ı birbirinden bağımsız geliştirebileceğimiz temel mimariyi oluşturmak.

## İşler

- SwiftUI uygulama projesi.
- Swift Concurrency tabanlı task modeli.
- Ana modül sınırları:

```text
Library
Documents
Speech
Audio
UI
Infrastructure
```

- `SpeechSynthesizing` protokolünü tanımla.
- `PlaybackEngine` arayüzünü tanımla.
- `AudioCache` arayüzünü tanımla.
- `DocumentExtracting` arayüzünü tanımla.
- uygulama veri dizinlerini ayır:
  - metadata
  - imported documents
  - generated audio cache
  - temporary synthesis files
- temel logging / debug metrics altyapısı.
- model ve cache format sürümlerini merkezi olarak tanımla.

## Çıkış kriteri

- Uygulama boş library ekranıyla açılıyor.
- Modüller arasında doğrudan gereksiz bağımlılık yok.
- TTS backend'i değiştirilse player ve document pipeline değişmek zorunda değil.
- Cache silinse kitap metadata/progress kaybolmuyor.

---

# R2 — Supertonic 3 Core ML dikey dilimi

## Amaç

Uygulama içinde gerçek Türkçe metni üretip oynatan ilk uçtan uca TTS yolunu tamamlamak.

## İşler

- `Supertonic3CoreMLSynthesizer` implementasyonu.
- model `prepare()` / `releaseResources()` lifecycle.
- Türkçe dil seçimi.
- mevcut Supertonic voice/style seçeneklerinin modellenmesi.
- sentence/paragraph input kabulü.
- cancellation desteği.
- synthesis metrics.
- PCM → kalıcı audio encoding yolu.
- tek bir örnek metni:

```text
text
 → Supertonic
 → PCM/audio buffer
 → AAC/M4A
 → local file
 → PlaybackEngine
```

akışından geçir.

## Çıkış kriteri

- Ağ kapalıyken Türkçe sentez çalışıyor.
- Aynı request yanlışlıkla paralel iki kez çalıştırılmıyor.
- sentez iptal edilebiliyor.
- model yükleme / bırakma tekrarlanabiliyor.
- üretilen M4A tekrar açılıp oynatılabiliyor.
- RTF her sentez için kaydediliyor.

---

# R3 — Metin segmentleme ve kalıcı audio cache

## Amaç

Uzun metni tek dev ses dosyası yerine güvenli, adreslenebilir ve yeniden kullanılabilir parçalara dönüştürmek.

## İşler

- sentence-aware segmenter.
- paragraph sınırlarını mümkün olduğunca koru.
- fiziksel audio segment hedefi: başlangıçta yaklaşık 30–90 saniye.
- segment metadata modeli.
- deterministic cache key.
- atomic file write:
  - önce temporary file
  - encoding tamamlanınca final path'e atomik taşıma
- yarım / bozuk sentez dosyalarını cache'e kabul etme.
- cache hit / miss.
- cache invalidation:
  - text hash
  - model version
  - voice/style
  - synthesis settings
  - cache format version
- LRU benzeri temizlik altyapısı.

## Çıkış kriteri

- aynı segment ikinci kez istenince yeniden sentezlenmeden cache'ten geliyor.
- yarım synthesis kalıcı valid cache olarak görünmüyor.
- voice/model değişince yalnız ilgili segmentler invalid oluyor.
- cache manuel temizlenebiliyor.
- kitap/progress metadata cache silinmesinden etkilenmiyor.

---

# R4 — Adaptif render-ahead scheduler

## Amaç

Arc'ın ayırt edici mimarisini tamamlamak: modeli sürekli çalıştırmadan kesintisiz ses üretmek.

## Başlangıç politikası

İlk tuning değerleri:

```text
startup playable buffer: 30–60 s
high-water target:       10–15 min
low-water target:        4–5 min
```

Bunlar sabit ürün kuralları değildir.

## Scheduler girdileri

- mevcut playback position
- cache'te hazır ileri ses süresi
- son sentezlerin moving-average RTF değeri
- playback rate
- thermal state
- Low Power Mode
- pending synthesis işleri
- kullanıcının seek / chapter değişimi

## Dinamik hesap

```text
estimatedGenerationTime = targetAudioToGenerate × measuredRTF

safeLowWater = max(
    minimumLowWater,
    estimatedGenerationTime × safetyFactor + fixedMargin
)
```

Başlangıç:

```text
safetyFactor = 2.0
fixedMargin = 30–60 s
```

## İşler

- tek authoritative `RenderScheduler`.
- high/low water state machine.
- synthesis queue prioritization.
- playback'e en yakın eksik segment en yüksek öncelik.
- seek durumunda eski ileri işler cancel / deprioritize.
- bölüm atlamasında yeni konum öncelik kazanır.
- model yalnız aktif batch için hazırlanır.
- high-water dolunca synthesis durur ve kaynaklar bırakılır.
- underrun telemetry.
- thermal state ciddi olduğunda batch küçültme.
- critical state'te zorunlu olmayan pre-render'ı durdurma.

## Çıkış kriteri

- en az 60 dakikalık sentetik/uzun metin testinde playback underrun olmuyor.
- model dinleme süresinin tamamı boyunca aktif kalmıyor.
- scheduler gerçek ölçülen RTF'ye göre tetik noktasını değiştiriyor.
- seek sonrası artık gereksiz eski bölümün üretimine devam etmiyor.
- playback hazır cache'ten TTS'den bağımsız devam ediyor.

---

# R5 — EPUB pipeline

## Amaç

Gerçek bir EPUB kitabını import edip bölüm yapısını koruyarak Arc metin modeline dönüştürmek.

## İşler

- Files picker ile EPUB import.
- EPUB container açma.
- OPF manifest / spine işleme.
- XHTML → text.
- başlık ve bölüm sırasını koruma.
- navigation / menu tekrarlarını temizleme.
- CSS/script/markup artıklarını çıkarma.
- footnote davranışını belirleme.
- Unicode ve whitespace normalization.
- Türkçe cümle segmentleme edge-case'leri.
- kitap başlığı / yazar / kapak metadata'sı mümkün olduğunda alma.
- source fingerprint oluşturma.

## Çıkış kriteri

- farklı yapıda en az birkaç EPUB doğru sırayla açılıyor.
- bölüm sırası spine ile uyumlu.
- menü/navigation metinleri gereksiz biçimde okunmuyor.
- aynı EPUB yeniden import edildiğinde kararlı normalized text/hash elde ediliyor.
- EPUB → segment → cache → playback uçtan uca çalışıyor.

---

# R6 — PDF pipeline

## Amaç

Metin katmanlı PDF'leri seslendirmek.

## İşler

- Files picker ile PDF import.
- PDFKit text extraction.
- sayfa sırasını koruma.
- satır sonu normalization.
- bölünmüş kelimeleri birleştirme heuristics.
- tekrar eden header/footer tespiti.
- sayfa numarası temizleme.
- çok kolonlu / karmaşık PDF'lerde güvenli fallback davranışı.
- extraction kalitesini kullanıcıya gerektiğinde gösterecek hata durumu.

## MVP sınırı

Görüntü/tarama tabanlı PDF için OCR bu fazda zorunlu değildir.

## Çıkış kriteri

- normal metin PDF'lerde okuma sırası kabul edilebilir.
- gereksiz sayfa başlığı/numara tekrarları temel örneklerde temizleniyor.
- extraction başarısızsa sessizce bozuk TTS üretmek yerine kullanıcıya durum bildiriliyor.
- PDF → segment → cache → playback uçtan uca çalışıyor.

---

# R7 — Player, background audio ve lock screen

## Amaç

Arc'ı gerçek bir audiobook/media player gibi davranır hale getirmek.

## İşler

- `AVAudioSession.Category.playback`.
- Background Modes → Audio.
- play / pause.
- ±15 veya ±30 saniye seek.
- chapter next / previous.
- playback rate.
- gapless veya fark edilmeyecek segment transition.
- Now Playing metadata.
- lock-screen controls.
- Control Center controls.
- interruption handling.
- telefon görüşmesi sonrası uygun resume.
- kulaklık / Bluetooth route değişimi.
- audio route disconnect davranışı.
- player position → ReadingProgress persistence.

## Kritik test

Aşağıdaki senaryo gerçek cihazda çalışmalı:

```text
kitabı başlat
→ yeterli cache üret
→ ekranı kilitle
→ TTS gerekmediği dönemde cache'ten oynat
→ interruption yaşa
→ geri dön
→ doğru konumdan devam et
```

## Çıkış kriteri

- ekran kapalıyken uzun süre playback kesilmiyor.
- segment sınırları normal dinlemede fark edilmiyor.
- Control Center / lock screen komutları doğru çalışıyor.
- uygulama tekrar açıldığında doğru konumdan devam ediyor.

---

# R8 — Library, progress ve ürün UI

## Amaç

Teknik prototipi günlük kullanılabilir uygulamaya dönüştürmek.

## Ekranlar

### Library

- kitap kapağı
- başlık
- yazar
- ilerleme
- son dinleme zamanı
- import
- kitap silme
- audio cache temizleme

### Book / Reader

- bölüm listesi
- mevcut bölüm
- metin konumu
- cache / hazırlanıyor durumu gerektiği kadar sade gösterim

### Player

- play/pause
- seek
- bölüm
- hız
- voice/style
- sleep timer post-MVP'ye bırakılabilir

### Settings

- varsayılan voice/style
- playback rate
- cache limiti
- diagnostic/benchmark bilgileri için debug build seçeneği

## Çıkış kriteri

- kullanıcı uygulamayı debug araçlarına ihtiyaç duymadan kullanabiliyor.
- import → play akışı açık ve kısa.
- sentez teknik ayrıntıları normal kullanıcıyı gereksiz yere meşgul etmiyor.
- kitap kapatılıp açıldığında ilerleme korunuyor.

---

# R9 — Dayanıklılık, enerji ve performans tuning

## Amaç

Mimariyi gerçek kullanım koşullarında doğrulamak ve scheduler tuning'ini veriye dayalı yapmak.

## Test matrisi

- kısa öykü
- 1 saat+ metin
- 8–15 saat eşdeğer uzun kitap
- çok sayıda kısa bölüm
- tek dev bölüm
- hızlı ileri seek
- bölüm atlama
- voice değiştirme
- cache dolması
- düşük disk alanı
- Low Power Mode
- thermal `.serious`
- memory pressure
- telefon görüşmesi / interruption
- Bluetooth bağlanma / ayrılma
- uygulamanın background/foreground geçişleri

## Enerji ölçümü

Özellikle karşılaştır:

1. anlık/continuous synthesis prototipi
2. render-ahead cache scheduler

Aynı metin, aynı cihaz, aynı ses seviyesi, mümkün olduğunca benzer koşullar kullanılmalı.

Ölç:

- 1 saatlik dinleme başına batarya farkı
- toplam synthesis active time
- model load sayısı
- ortalama RTF
- thermal state
- underrun sayısı
- cache I/O

## Tuning hedefi

Modeli çok sık yükleyip boşaltmak ile çok uzun süre bellekte tutmak arasında optimum noktayı gerçek ölçümlerle bul.

## Çıkış kriteri

- uzun dinlemede playback underrun kabul edilemez düzeyde değil.
- scheduler gereksiz model thrash oluşturmuyor.
- thermal throttling altında güvenli biçimde adapte oluyor.
- cache büyümesi kontrol altında.
- crash / yarım dosya sonrası cache kendini toparlayabiliyor.

---

# R10 — MVP Release Candidate

## MVP özellik seti

- iPhone / iPad
- tamamen local kullanım
- Supertonic 3
- Core ML inference
- Türkçe TTS
- EPUB
- metin katmanlı PDF
- render-ahead adaptive cache
- M4A/AAC local audio cache
- background / lock-screen playback
- seek / chapter navigation
- playback speed
- reading progress
- basic library
- cache management

## Release blocker kriterleri

Aşağıdakilerden biri varsa MVP release edilmez:

- normal bir EPUB/PDF kullanımında sık playback underrun
- ekran kilitliyken playback'in güvenilmez olması
- cache corruption'ın uygulamayı kullanılamaz hale getirmesi
- kitap içeriğinin cihaz dışına çıkması
- model/backend hatasının sessiz veri kaybı oluşturması
- progress'in sık kaybolması
- disk cache'in limitsiz büyümesi
- desteklenen cihazlarda sistematik crash / OOM

## Release öncesi doğrulama

- temiz kurulum
- upgrade/migration senaryosu
- airplane mode
- düşük disk alanı
- uzun kitap
- en az iki farklı performans sınıfında gerçek Apple cihazı
- privacy manifest / App Store gereksinimleri
- third-party model/license attribution kontrolü

---

# MVP sonrası

Öncelik gerçek kullanıcı verisine göre belirlenecek.

## R11 — OCR

- Vision tabanlı taranmış PDF OCR
- sayfa OCR cache
- layout-aware text ordering

## R12 — Daha fazla belge kaynağı

- TXT
- Markdown
- DOCX
- Share Sheet
- Files / Open In entegrasyonu iyileştirmeleri

## R13 — Offline pre-render modu

Kullanıcı isterse:

> "Bu kitabın tamamını hazırla"

komutuyla cihaz uygun koşullardayken bütün kitabı önceden seslendirebilir.

Dikkat: iOS background execution garantilerine dayanılmamalı; uygulama foreground, şarj durumu ve sistemin verdiği background fırsatları birlikte değerlendirilmelidir.

## R14 — Gelişmiş dinleme

- sleep timer
- bookmark
- favori bölüm
- chapter-level cache controls
- silence trimming değerlendirmesi
- configurable seek intervals

## R15 — TTS backend evrimi

- yeni Supertonic sürümleri
- kalite/performance karşılaştırması
- alternatif tamamen local backend'ler

`SpeechSynthesizing` sınırı sayesinde bu değişikliklerin document/player katmanına yayılmaması hedeflenir.

---

# Geliştirme öncelik kuralları

Bir özellik aşağıdaki sıraya göre değerlendirilir:

1. **Playback güvenilirliği**
2. **Türkçe TTS doğruluğu ve doğallığı**
3. **Veri/cache bütünlüğü**
4. **Enerji ve thermal davranış**
5. **Startup latency**
6. **Disk kullanımı**
7. **UI polish**
8. yeni özellikler

Arc'ın temel fonksiyonu çalışmıyorsa kozmetik özellik eklenmemelidir.

---

# Definition of Done

Bir roadmap maddesi yalnız kod yazıldığı için tamamlanmış sayılmaz.

İlgili olduğu yerde:

- gerçek cihazda doğrulanmalı,
- hata yolu tanımlanmalı,
- cancellation davranışı test edilmeli,
- concurrency race oluşturmamalı,
- kalıcı veri/cache migration etkisi düşünülmeli,
- metric/log ile gözlemlenebilir olmalı,
- enerji/thermal etkisi kritikse ölçülmeli,
- kullanıcı verisini cihaz dışına göndermemeli.

---

# İlk uygulanacak sıra

İlk geliştirme döngüsünde yalnız şu hedeflere odaklan:

```text
1. Minimal iOS app
2. Supertonic 3 Core ML'i gerçek cihazda çalıştır
3. Türkçe sabit metin sentezle
4. Çıktıyı M4A dosyasına yaz
5. Dosyayı modelden bağımsız oynat
6. Ekranı kilitle ve playback'i doğrula
7. RTF / memory / thermal benchmark kaydet
8. Sonuca göre Core ML backend ve scheduler başlangıç değerlerini kesinleştir
```

Bu doğrulanmadan EPUB parser, kapsamlı library UI veya gelişmiş player özelliklerine büyük yatırım yapılmamalıdır.
