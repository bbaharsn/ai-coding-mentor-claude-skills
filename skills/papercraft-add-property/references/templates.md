# Şablonlar

Gerçek örnekler: `Models/ButtonStyle/ButtonFill.swift`, `Views/Canvas/ButtonElementView.swift`, `Views/Editor/ButtonPropertiesSheet.swift`, `PaperCraftTests/ButtonStyleTests.swift`. Şablon ile kod ayrışırsa kodu esas al.

## 1. Seçenek enum'u — `Models/<Grup>Style/<Ad>.swift`

> Örnek sadece kalıbı göstermek için. Gerçekte TextField'ın JSON'unda zaten `"style": "roundedBorder"` var ve `ElementView` bunu henüz okumuyor (sabit `.roundedBorder`). Yeni alan açmadan önce böyle mevcut bir alanı tiplendirmek daha doğru olabilir — kullanıcıya sor.

```swift
//
//  TextFieldBorder.swift
//  PaperCraft
//

import Foundation

/// Metin kutusunun kenarlığı.
nonisolated enum TextFieldBorder: String, Codable, CaseIterable {
    case roundedBorder
    case underline
    case none

    /// Özellikler sayfasında görünen ad.
    var title: String {
        switch self {
        case .roundedBorder: "Çerçeve"
        case .underline: "Alt çizgi"
        case .none: "Yok"
        }
    }

    /// Tanımadığımız bir değer gelirse (ör. AI uydurduysa) çökmek yerine `.roundedBorder` olur.
    init(from decoder: Decoder) throws {
        let rawValue = try decoder.singleValueContainer().decode(String.self)
        self = TextFieldBorder(rawValue: rawValue) ?? .roundedBorder
    }
}
```

## 2. `ElementProperties` alanı

```swift
    // MARK: - Metin kutusu görünümü

    var border: TextFieldBorder?
```

## 3. Katalog JSON

```json
"defaultProperties": {
  "placeholder": "Placeholder",
  "border": "roundedBorder"
}
```

## 5. Sayfadaki kontrol

```swift
Section("Kenarlık") {
    segmented(binding(\.border, default: .roundedBorder), title: \.title)
}
```

`binding` ve `segmented` yardımcıları `ButtonPropertiesSheet`'te. İkinci bir sayfa yazılırken bunlar kopyalanacaksa ortak bir yere (ör. `ElementPropertiesForm` ya da bir `extension`) taşımayı düşün.

## 6. Testler

```swift
@Test func catalogTextFieldHasBorderDefault() throws {
    let field = try #require(try ComponentCatalog.load().makeElement(of: .textField))
    #expect(field.properties.border == .roundedBorder)
}

@Test func unknownBorderFallsBack() throws {
    let json = #"{ "border": "zigzag" }"#
    let properties = try JSONDecoder().decode(ElementProperties.self, from: Data(json.utf8))
    #expect(properties.border == .roundedBorder)
}
```

## 7. Doğrulama komutları

```bash
# Değişkenler
SIM=34B676F8-0333-49D3-BF23-4161C1B667EE   # iPhone 13; yoksa: xcrun simctl list devices available
D=<scratchpad>/dd                           # derived data, proje dışında

# iOS testleri
rm -rf $D/Logs/Test
xcodebuild test -project PaperCraft.xcodeproj -scheme PaperCraft \
  -destination "id=$SIM" -derivedDataPath $D 2>&1 | grep -E "error:|\*\* TEST"
xcrun xcresulttool get test-results summary \
  --path $(ls -td $D/Logs/Test/*.xcresult | head -1) | grep -E '"(passedTests|failedTests)"'

# macOS derlemesi
xcodebuild build -project PaperCraft.xcodeproj -scheme PaperCraft \
  -destination 'platform=macOS' -derivedDataPath $D 2>&1 | grep -E "error:|\*\* BUILD"

# Simülatörü tekrar aç + uygulamayı derle
xcrun simctl boot $SIM; xcrun simctl bootstatus $SIM -b
xcodebuild build -project PaperCraft.xcodeproj -scheme PaperCraft \
  -destination "id=$SIM" -derivedDataPath $D 2>&1 | grep -E "error:|\*\* BUILD"
# Sonra simülatör aracıyla: launch $D/Build/Products/Debug-iphonesimulator/PaperCraft.app
```

Yakın çekim gerekirse: `xcrun simctl io $SIM screenshot full.png` + `sips -c <h> <w> --cropOffset <y> <x> full.png --out crop.png`.
