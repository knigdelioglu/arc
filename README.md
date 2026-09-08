# Arc

**Arc**, PDF ve EPUB gibi belgeleri tamamen cihaz üzerinde Türkçe seslendirmek için geliştirilen, gizlilik odaklı bir iOS/iPadOS sesli okuma uygulamasıdır.

Temel TTS motoru **Supertonic 3** olacaktır. Model, mümkün olan Apple cihazlarında **Core ML** üzerinden çalıştırılacak ve hesaplama önceliği Neural Engine uyumlu yürütme yoluna verilecektir. Metin veya kitap içeriği seslendirme amacıyla bir sunucuya gönderilmez.

## Proje hedefi

Arc'ın amacı bir metni modelden gerçek zamanlı olarak her saniye üretmek değil, **ileriden ses üretip yerel olarak önbelleğe almak** ve oynatmayı bu cache üzerinden sürdürmektir.

Bu yaklaşımın hedefleri:

- ekran kilitliyken kesintisiz dinleme,
- TTS modelini sürekli RAM'de ve aktif tutmama,
- daha düşük pil tüketimi,
- ileri/geri sarma ve kaldığı yerden devam etme,
- internet bağlantısı olmadan çalışma,
- PDF/EPUB içeriğini cihazdan çıkarmama.

## Temel mimari

```text
PDF / EPUB
    │
    ▼
Document Importer
    │
    ▼
Text Extraction
    │
    ▼
Normalization + Chapter/Paragraph Segmentation
    │
    ▼
Render Scheduler
    │
    ├── buffer yeterli ────────────────┐
    │                                  │
    └── buffer düşük                   │
           │                           │
           ▼                           │
   Supertonic 3 / Core ML              │
           │                           │
           ▼                           │
   küçük ses segmentleri               │
           │                           │
           ▼                           │
      Local Audio Cache ◄──────────────┘
           │
           ▼
       Playback Engine
           │
           ├── lock screen
           ├── background audio
           ├── seek / resume
           └── Now Playing controls
```

## Neden anlık TTS yerine render-ahead cache?

Supertonic 3 gerçek zamandan hızlı sentez yapabildiği için modelin dinleme süresinin tamamı boyunca aktif kalmasına gerek yoktur.

Arc'ın varsayılan stratejisi:

1. Kullanıcı okumayı başlattığında ilk oynatılabilir ses mümkün olduğunca hızlı üretilir.
2. Oynatma başladıktan sonra uygulama ilerideki bölümleri toplu olarak üretir.
3. Yeterli ses cache'e alındığında Core ML model referansları serbest bırakılır.
4. Oynatma yalnızca yerel ses dosyalarından devam eder.
5. Cache belirlenen alt eşiğe indiğinde model yeniden yüklenir ve buffer tekrar doldurulur.
6. Sonraki ses üretimi tamamlandığında model yeniden bırakılır.

### Sabit 20 dakika yerine adaptif buffer

Örneğin model bir saatlik sesi 9 dakikada üretiyorsa RTF yaklaşık `0.15` olur. Bu durumda 20 dakikalık yeni ses yaklaşık 3 dakikada üretilebilir. Dolayısıyla 20 dakikalık cache'in 19. dakikasında yeni üretime başlamak geç kalabilir.

Bu nedenle scheduler sabit bir dakika kullanmayacaktır. Karar şu verilere göre alınacaktır:

- cihazda ölçülen gerçek sentez hızı,
- mevcut oynatma hızı,
- cache'te kalan ses süresi,
- üretilecek hedef buffer,
- thermal state ve düşük güç modu,
- önceki sentezlerin süreleri.

İlk tasarım hedefi:

- **high-water mark:** yaklaşık 10–15 dakika hazır ses,
- **low-water mark:** yaklaşık 4–5 dakika hazır ses,
- low-water'a gelindiğinde high-water seviyesine kadar yeni ses üretme,
- gerçek cihaz ölçümlerine göre bu değerleri otomatik uyarlama.

Bu değerler ürün davranışı değil, başlangıç tuning değerleridir.

## Cache biçimi

Uzun tek bir WAV dosyası oluşturmak yerine küçük, adreslenebilir ses segmentleri tutulacaktır.

Önerilen yapı:

- sentez batch'i: birkaç dakikalık metin,
- fiziksel cache segmenti: yaklaşık 30–90 saniye,
- codec: konuşma için uygun AAC/M4A,
- mono çıkış,
- segmentler arasında kesintisiz oynatma.

Ham PCM/WAV cache kullanılmamalıdır. 44.1 kHz, 16-bit mono PCM yaklaşık 88.2 KB/s tüketir; 20 dakika yaklaşık 106 MB'a ulaşır. 64 kbps AAC ise aynı 20 dakikayı yaklaşık 9.6 MB civarında tutar.

Her cache girdisi şu parametrelerden türetilen kararlı bir anahtara sahip olmalıdır:

```text
documentHash
chapter/segment identifier
normalizedTextHash
modelVersion
voice/style
language
synthesis settings
cache format version
```

Metin, model, ses veya sentez ayarı değişirse yalnız etkilenen segmentler geçersiz sayılır.

## Background ve ekran kilidi

Arc bir medya oynatma uygulaması gibi davranacaktır:

- `AVAudioSession` kategorisi `.playback`,
- Background Modes → Audio,
- lock-screen / Control Center kontrolleri,
- Now Playing metadata,
- kesinti ve kulaklık/Bluetooth route değişikliklerinin yönetimi.

Önemli tasarım kuralı: **uygulamanın kesintisiz çalışması, iOS'un keyfî arka plan compute süresi vermesine bağlı olmamalıdır.** Hazır ses buffer'ı bu nedenle mimarinin temel parçasıdır. Background processing fırsatları ek optimizasyon olarak kullanılabilir; oynatmanın tek dayanağı değildir.

## MVP dosya desteği

İlk sürüm:

- EPUB
- metin katmanı bulunan PDF

Sonraki sürümler:

- TXT / Markdown / DOCX
- taranmış PDF için Vision OCR
- Share Sheet ile uygulamaya gönderme

## Teknoloji yönü

- Swift
- SwiftUI
- Swift Concurrency
- Core ML
- AVFoundation / AVAudioSession
- PDFKit
- MediaPlayer / Now Playing
- Supertonic 3

Supertonic çalışma katmanı uygulamanın geri kalanından bir protokol ile ayrılacaktır; böylece Core ML model paketi veya inference implementasyonu değiştirilebilir.

## Gizlilik

Arc'ın temel çalışma modu tamamen yereldir:

- kitap içeriği sunucuya gönderilmez,
- TTS yerel çalışır,
- oluşturulan ses yerel cache'te tutulur,
- telemetry/analytics MVP'nin zorunlu parçası değildir.

## Durum

Proje başlangıç aşamasındadır.

- Ayrıntılı MVP kapsamı ve kabul kriterleri: [`SCOPE.md`](SCOPE.md)
- Fazlara ayrılmış geliştirme planı: [`ROADMAP.md`](ROADMAP.md)
