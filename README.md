
<div align="center">

# WordKit

Modern Swift ortamları için **async/await** tabanlı, test edilebilir ve genişletilebilir sözlük/kelime verisi istemcisi.

[![Swift Package Manager](https://img.shields.io/badge/SPM-compatible-success)](#installation)
[![Platforms](https://img.shields.io/badge/platforms-iOS%2013%2B%20%7C%20macOS%2011%2B%20%7C%20tvOS%2014%2B%20%7C%20watchOS%208%2B-informational)](#requirements)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

</div>

---

## İçindekiler
- [Özet](#özet)
- [Mimari ve Tasarım Kararları](#mimari-ve-tasarım-kararları)
- [Gereksinimler](#gereksinimler)
- [Kurulum (Swift Package Manager)](#kurulum-swift-package-manager)
- [Hızlı Başlangıç](#hızlı-başlangıç)
- [API Referansı](#api-referansı)
  - [WordAPI](#wordapi)
  - [Modeller](#modeller)
  - [Hata Modeli](#hata-modeli)
  - [Dil Desteği](#dil-desteği)
  - [Yapılandırma](#yapılandırma)
- [Test Çalıştırma](#test-çalıştırma)
- [DocC Dokümantasyonu](#docc-dokümantasyonu)
- [Sürümleme ve Yol Haritası](#sürümleme-ve-yol-haritası)
- [Katkı Rehberi](#katkı-rehberi)
- [Lisans](#lisans)

---

## Özet
**WordKit**, sözlük servislerinden kelime girişlerini almak için yalın bir istemci sunar. Odak noktaları:
- **Async/Await** destekli modern API yüzeyi
- **Bağımlılıksız** ve hafif
- **Test edilebilir mimari** (enjekte edilebilir `URLSession`, mock’lar)
- **Anlamlı hata modeli** ve tip-güvenli dönüşler

> Not: Depo MIT lisanslıdır. Ayrıntı için `LICENSE` dosyasına bakın.

---

## Mimari ve Tasarım Kararları
- **Katmanlı yapı:**  
  `WordAPI` (public yüz) → `APIClient` (istek/yanıt ayrıştırma) → Modeller (`WordEntry`, `Meaning`, `Definition`, `Phonetic`, `APIError`).
- **Bağımlılık enjeksiyonu:**  
  Testlerde özelleştirilmiş `URLSession` (ör. `MockURLProtocol`) ile deterministik ağ senaryoları.
- **Sağlam hata ayrımı:**  
  2xx başarı, **404** için alan-özel hata (`wordNotFound`), diğer durumlar için **genel istek hatası**.

---

## Gereksinimler
- **Swift:** 5.9+ (önerilir)  
- **Xcode:** 15+  
- **Platformlar (öneri):** iOS 13+, macOS 11+, tvOS 14+, watchOS 8+

> Minimum sürümler projeye göre gevşetilebilir/sıkılaştırılabilir.

---

## Kurulum (Swift Package Manager)

### Xcode
1. **File → Add Packages...**
2. URL alanına: `https://github.com/CanSagnak1/WordKit.git`
3. Uygun **branch/tag** seçin ve ekleyin.

### Package.swift
```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "YourApp",
    platforms: [
        .iOS(.v13), .macOS(.v11), .tvOS(.v14), .watchOS(.v8)
    ],
    dependencies: [
        .package(url: "https://github.com/CanSagnak1/WordKit.git", from: "0.1.0")
    ],
    targets: [
        .target(
            name: "YourApp",
            dependencies: [
                .product(name: "WordAPI", package: "WordKit")
            ]
        )
    ]
)
````

---

## Hızlı Başlangıç

```swift
import WordAPI

let api = WordAPI() // varsayılan yapılandırma

Task {
    do {
        let entries = try await api.fetch(word: "serendipity", language: .en)
        // entries: [WordEntry]
        if let first = entries.first {
            print("Word:", first.word)
            print("Meanings:", first.meanings.map(\.partOfSpeech))
        }
    } catch {
        // Aşağıdaki Hata Modeli bölümüne bakın
        print("Fetch failed:", error)
    }
}
```

---

## API Referansı

### WordAPI

`WordAPI`, kelime girişlerini getirmek için ana geçittir.

```swift
public final class WordAPI {
    /// Varsayılan `URLSessionConfiguration` ve endpoint ile kurulum.
    public init(configuration: URLSessionConfiguration = .default,
                baseURL: URL? = nil)

    /// Belirtilen dilde bir kelime için sözlük girdilerini döndürür.
    public func fetch(word: String,
                      language: Language,
                      timeout: TimeInterval? = nil) async throws -> [WordEntry]
}
```

**Özellikler**

* `baseURL` özelleştirilebilir (test/alternatif endpoint’ler için).
* `timeout` hem `URLRequest` seviyesinde hem de `Task` tabanlı olarak uygulanabilir.
* Varsayılan `Accept: application/json` başlığı set edilir.

### Modeller

```swift
public struct WordEntry: Codable, Identifiable {
    public let id: String                // deterministik kimlik
    public let word: String
    public let phonetics: [Phonetic]
    public let meanings: [Meaning]
}

public struct Phonetic: Codable, Identifiable {
    public let text: String?
    public let audio: String?
    public let sourceUrl: String?
    public var id: String { ... }        // içerik-tabanlı kimlik
}

public struct Meaning: Codable, Identifiable {
    public let partOfSpeech: String
    public let definitions: [Definition]
    public var id: String { ... }        // içerik-tabanlı kimlik
}

public struct Definition: Codable, Identifiable {
    public let definition: String
    public let example: String?
    public let synonyms: [String]?
    public let antonyms: [String]?
    public var id: String { ... }
}

public struct APIError: Codable, Equatable {
    public let title: String
    public let message: String
    public let resolution: String?
}
```

> **Identifiable Notu:** UUID üretmek yerine **içerik-tabanlı** deterministik ID’ler kullanılır; bu sayede diffable veri kaynaklarında kararlılık sağlanır.

### Hata Modeli

```swift
public enum NetworkError: Error, Equatable {
    case invalidURL
    case networkError(underlying: Error)
    case requestFailed(statusCode: Int, body: Data?)
    case decodingError(underlying: Error)
    case wordNotFound(APIError)
}
```

**Yakalama Örneği**

```swift
do {
    let entries = try await api.fetch(word: "kitten", language: .en)
} catch let error as NetworkError {
    switch error {
    case .wordNotFound(let apiError):
        print("Not found:", apiError.message)
    case .requestFailed(let code, _):
        print("HTTP \(code)")
    default:
        print("General error:", error)
    }
} catch {
    print("Unexpected:", error)
}
```

### Dil Desteği

```swift
public enum Language: String, Codable, CaseIterable {
    case en, tr, es, de, fr, it, ptBR, ru, ja, ko, zh
    // Servis desteğine göre güncellenebilir.
}
```

> **İpucu:** Üçüncü taraf API’lerin kabul ettiği dil kodları servis dokümantasyonuna göre **birebir** eşlenmelidir. `Language` enum’unu servis matrisine göre genişletin veya `unknown(rawValue:)` stratejisi kullanın.

### Yapılandırma

```swift
var config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 15
let api = WordAPI(configuration: config,
                  baseURL: URL(string: "https://api.dictionaryapi.dev/api/v2/entries")!)
```

**Gelişmiş Ayarlar**

* **Başlıklar:** `User-Agent`, `Accept-Language` eklemek isteyebilirsiniz.
* **Retry politikası:** 5xx durumları için basit üssel geri çekilmeli (exponential backoff) bir strateji eklenebilir.
* **İptal Edilebilirlik:** `Task`’i saklayarak iptal (`task.cancel()`) uygulanabilir.

---

## Test Çalıştırma

Projede birim testler `XCTest` ile yazılmıştır (mock URL protokolü ve fixture’lar).

```bash
swift test
# veya Xcode üzerinden Product > Test
```

**Önerilen Ek Testler**

* Zaman aşımı ve iptal senaryoları
* 5xx durumlarında retry
* Dil varyantları (`pt-BR` gibi) için doğru URL oluşumu
* Başlıklar (`Accept`, `User-Agent`) set edildi mi?

---

## DocC Dokümantasyonu

DocC üretimi için:

```bash
xcodebuild docbuild \
  -scheme WordAPI \
  -destination 'generic/platform=iOS'
```

Ardından `.doccarchive`’ı Xcode Organizer üzerinden veya statik web dağıtımıyla yayınlayabilirsiniz.

---

## Sürümleme ve Yol Haritası

**SemVer** takip edilir: `MAJOR.MINOR.PATCH`

**Planlananlar**

* [ ] 5xx retry ve özelleştirilebilir `RetryPolicy`
* [ ] Yerleşik basit önbellek katmanı
* [ ] Gelişmiş `Language` matrisi ve doğrulama
* [ ] Örnek iOS demo uygulaması
* [ ] DocC içinde rehber (How-to) makaleleri

---

## Katkı Rehberi

Katkılar memnuniyetle karşılanır:

1. Issue açın (hata/öneri).
2. Fork → feature branch → PR.
3. PR’larda test eklemeyi ve **SwiftLint**/format kontrolünü (varsa) geçmeyi unutmayın.

---

## Lisans

Bu proje **MIT** lisanslıdır. Ayrıntılar için [LICENSE](LICENSE) dosyasına bakın.

