---
name: papercraft-add-property
description: PaperCraft editöründe bir canvas elemanına (Button, TextField, Image, Switch, Slider, Text, NavigationBar, TabBar) yeni düzenlenebilir özellik eklemenin adım adım iş akışı — model enum'u, ElementProperties alanı, katalog JSON varsayılanı, Özellikler sayfası kontrolü, canvas çizimi, testler ve simülatörde doğrulama. Kullanıcı bir elemanın rengini/şeklini/stilini/boyutunu/ikonunu/metnini değiştirilebilir yapmak istediğinde, "X'e şu özelliği ekleyelim", "TextField'ın kenarlığını değiştirebilsin", "Switch'in rengi seçilebilsin", "diğer elemanlara geçelim" ya da bir eleman için ilk kez Özellikler sayfası yapılacağında bu skill'i kullan — kullanıcı "özellik" kelimesini kullanmasa bile.
---

# PaperCraft: elemana yeni özellik ekleme

Butonun şekil, dolgu, renk, ikon ve boyut özellikleri bu akışla eklendi. Aynı kalıbı korumak önemli: kullanıcı projeyi öğrenerek geliştiriyor, her özelliğin aynı yerlerde aynı biçimde durması kodu okunur tutuyor ve AI'ın ileride üreteceği JSON ile uyumu koruyor.

Kod şablonları için `references/templates.md` dosyasını oku.

## Başlamadan önce

1. Kullanıcıyla **ne değişecek** konusunda anlaş: hangi eleman, hangi seçenekler, varsayılan ne. Görsel bir seçimse (şekil, stil) beyin fırtınası/görsel taslak daha hızlı karar verdirir.
2. Mevcut koda bak — kalıp değişmiş olabilir:
   - `PaperCraft/Models/ElementProperties.swift` — alan zaten var mı? (`style`, `symbolName`, `text` gibi genel alanlar birden çok tür tarafından paylaşılır; yenisini açmadan önce mevcut alan işe yarıyor mu bak.)
   - `PaperCraft/Resources/Catalog/ComponentCatalog.json` — elemanın `defaultProperties`'i
   - `PaperCraft/Views/Canvas/ElementView.swift` — eleman nasıl çiziliyor
   - `PaperCraft/Views/Editor/*PropertiesSheet.swift` — o elemanın sayfası var mı

## Adımlar

### 1. Model (seçenekli özellikse)
- Yeni enum'u **kendi dosyasına** koy: `PaperCraft/Models/<Grup>Style/<Ad>.swift` (ör. `ButtonStyle/ButtonFill.swift`). Projede kural: her dosyada tek tip.
- `nonisolated enum ...: String, Codable, CaseIterable` — proje varsayılan olarak `MainActor` izolasyonlu; modeller her yerden kullanılabilsin diye `nonisolated`.
- Sayfada görünecek Türkçe ad için `var title: String`.
- **Bilinmeyen değerde çökmeyen** `init(from:)`: tanınmayan ham değer varsayılana düşer. Sebep: JSON ileride AI'dan gelecek ve uydurma değerler içerebilir; `ElementType` de aynı şeyi yapıyor.
- Basit değerler (metin, sayı, Bool, renk) için enum gerekmez. **Renkler hex `String`** olarak saklanır ("#AC5135"); `Color(hexString:)` ve `.hexString` zaten var.

### 2. `ElementProperties`
- Opsiyonel alan ekle (`var shape: ButtonShape?`). Hepsi opsiyonel, çünkü her tür sadece kendi alanlarını kullanır ve eski kayıtlar bu alan olmadan da açılabilmeli.
- İlgili türün `// MARK:` bölümüne koy, kısa Türkçe doc yorumu yaz.

### 3. Katalog varsayılanı
- `ComponentCatalog.json` içinde o elemanın `defaultProperties`'ine varsayılan değeri ekle. Alan adı Swift'teki property adıyla **birebir** aynı olmalı (decode bu sayede çalışıyor).

### 4. Canvas'ta çizim
- Eleman `ElementView` içinde tek satırdan büyük çiziliyorsa kendi görünümüne taşı: `Views/Canvas/<Tür>ElementView.swift` (ör. `ButtonElementView`). `ElementView`'daki `case` sadece onu çağırır.
- Özellik `nil` ise katalogdaki varsayılanla aynı görünmeli (`properties.shape ?? .rounded`).
- Boyutla ilgili çizimlerde `.aspectRatio(nil, ...)` kullanma — içeriğin kendi oranına küçülür ve eleman çerçevesini doldurmaz (butonda yaşandı). Oran gerekiyorsa sadece o durumda uygula.

### 5. Özellikler sayfası
- Sayfanın çerçevesi (başlık, "Bitti", `Form`, `Theme.panel`) ortak: `Views/Editor/ElementPropertiesSheet.swift`. Elemanın sayfası yoksa:
  1. Sadece `Section`'ları içeren `Views/Editor/<Tür>PropertiesSheet.swift` yaz (`TextFieldPropertiesSheet`'i örnek al).
  2. `ElementPropertiesSheet`'te `hasSheet(for:)`, `content` ve `title`'a yeni türü ekle.
  - Açılma kuralı değişmesin: **tek dokunuş seçer, çift dokunuş sayfayı açar** (`ContentView.isShowingProperties`). Kullanıcı bunu bilinçli seçti — tek dokunuşta sayfa açılıp canvas'ı küçültmesin.
- Ortak parçaları kullan, kopyalama:
  - `viewModel.binding(\.alan, default:)` (`EditorViewModel+Bindings.swift`) — değişiklik anında canvas'a yansır, "Kaydet" yok.
  - `SizeSection(viewModel:)` — genişlik × yükseklik.
  - `OptionPicker(selection:title:)` — enum seçenekleri (segmented).
  - `ColorSwatchRow` — renk; `SymbolPickerRow` — ikon.
- Metin → `TextField`; sayı → `.numbersAndPunctuation` klavyeli `TextField` (`numberPad`'de return yok, değer hiç uygulanmaz).
- Seçim çizgisinin şekli `SelectionOverlay.buttonShape`'te türe göre seçilir; elemanın kenarı kıvrımlıysa oraya da ekle.
- Bir değişiklik başka alanı da etkiliyorsa (daire şekli → çerçeve kare olur) generic binding yerine `EditorViewModel`'de ayrı bir metot yaz (`setSelectedButtonShape` gibi) ve onu test et.

### 6. Testler (Swift Testing)
Her yeni özellik için en az:
- Katalogdan üretilen elemanın varsayılanı doğru (`catalog.makeElement(of:)`)
- Bilinmeyen değer çökmeden varsayılana düşüyor (JSON'dan decode)
- ViewModel'e yeni metot eklendiyse onun davranışı (sınırlar dahil)

Testler `PaperCraftTests/` altında, `@MainActor struct ...Tests`. Point/normalize dönüşümü olan karşılaştırmalarda küçük tolerans kullan (`abs(a - b) < 0.001`).

### 7. Doğrulama
`references/templates.md` sonundaki komutlarla:
1. iOS simülatörde testleri çalıştır, geçen/kalan sayısını xcresult'tan oku.
2. **macOS için de derle** — proje native macOS'u da hedefliyor; `keyboardType`, `navigationBarTitleDisplayMode` gibi iOS'a özel API'ler `#if os(iOS)` içinde olmalı.
3. Uygulamayı simülatörde aç, elemanı seç, özelliği değiştir, ekran görüntüsüyle canvas'ı kontrol et.

Simülatör tuzakları (vakit kaybettirdi):
- `xcodebuild test` bittikten sonra simülatör kapanmış olabilir; `xcrun simctl boot` + `bootstatus -b` ile tekrar aç.
- Uygulama açıldıktan sonraki **ilk dokunuş** çoğu zaman boşa gidiyor; önce boş bir yere dokun.
- Animasyon bitmeden alınan ekran görüntüsü eski durumu gösterir; ~1 sn bekle.
- **Çift dokunuş** araçla güvenilir gönderilemiyor (iki ayrı çağrı arası çok uzun) — kullanıcıdan denemesini iste.
- Sayfanın tutamacına dokunmak detent'i değiştirir (yarımdan tam ekrana); dokunuş koordinatlarını buna göre seç.

### 8. Bitiş
- Yol haritasında (`docs/yol_haritasi.md`) ilgili madde varsa güncelle.
- Commit'i kullanıcı onaylamadan atma; onaylarsa Türkçe, maddeli bir mesaj yaz.
- Kullanıcıya kısa özet ver: ne eklendi, ne test edildi, neyi kendisinin denemesi gerekiyor.
