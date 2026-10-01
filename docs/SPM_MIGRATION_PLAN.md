# payjp-flutter-plugin Swift Package Manager 対応方針

作成日: 2026-09-16

## 1. 背景

- Flutter 3.44 以降、iOS/macOS ネイティブ依存管理のデフォルトが **Swift Package Manager (SPM)** に変更された
- CocoaPods は当面サポートされるが、CocoaPods 中央リポジトリは **2026-12-02 に read-only 化**予定
- Flutter プラグインは当面 **SPM と CocoaPods の両対応が必須**
- iOS SDK（[payjp/payjp-ios](https://github.com/payjp/payjp-ios)）は既に SPM 対応済み

参考: [Flutter 公式ガイド (for plugin authors)](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)

## 2. 現状分析

### 2.1 プラグイン構成
- `ios/Classes/` に Swift + Objective-C が混在
  - **Swift**: `SwiftPayjpFlutterPlugin.swift`, `ApplePayModule.swift`, `CardFormModule.swift`, `ChannelMethod.swift`, `PayjpPluginConstant.swift`, `TokenStoringError.swift`
  - **Obj-C**: `PayjpFlutterPlugin.{h,m}`, `PayjpAppDelegateInterceptor.{h,m}`
- Obj-C 側は Swift 側を `#import <payjp_flutter/payjp_flutter-Swift.h>` で参照している
- 依存: `PAYJPFlutterCore ~> 2.2.3` / `GoogleUtilities/AppDelegateSwizzler ~> 8` / `Flutter`
- `pubspec.yaml` の Flutter バージョン制約: `>=3.38.1`
- iOS deployment target: `12.0`

### 2.2 iOS SDK の SPM 対応状況
- `Package.swift` は **`PAYJP` product のみ**を公開（`PAYJPFlutterCore` は SPM 未公開）
- SPM 側 iOS 最低 deployment target: `iOS 10.0`
- プラグインコードは元々 `import PAYJP` を使用しており、SPM 移行後もソースコード変更は不要

### 2.3 SPM 対応で発生する制約
- SPM は 1 target = 1 言語が原則。**Swift/Obj-C 混在の解消が必要**
- `Package.swift` は静的宣言のみ。`sdkconfig.json` の動的読み込みは不可

## 3. 対応方針

### 3.1 Obj-C コードの Swift 化（A 案）

Swift 統一で 1 target 化する。

| ファイル | 対応 |
|---|---|
| `PayjpFlutterPlugin.{h,m}` | 削除。Swift 側 `SwiftPayjpFlutterPlugin` に `@objc` + `FlutterPlugin` 準拠で登録処理を集約 |
| `PayjpAppDelegateInterceptor.{h,m}` | Swift 化。`GoogleUtilities/GULAppDelegateSwizzler` は Swift からも呼び出し可能 |

### 3.2 ディレクトリ構成

```
ios/
├── payjp_flutter/                             # 新規
│   ├── Package.swift                          # 新規
│   └── Sources/
│       └── payjp_flutter/
│           ├── SwiftPayjpFlutterPlugin.swift
│           ├── PayjpAppDelegateInterceptor.swift   # Obj-C から書き換え
│           ├── ApplePayModule.swift
│           ├── CardFormModule.swift
│           ├── ChannelMethod.swift
│           ├── PayjpPluginConstant.swift
│           └── TokenStoringError.swift
├── Classes/                                   # 削除
└── payjp_flutter.podspec                      # パス修正して維持
```

### 3.3 Package.swift（想定）

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "payjp_flutter",
    platforms: [
        .iOS("12.0")
    ],
    products: [
        .library(name: "payjp-flutter", targets: ["payjp_flutter"])
    ],
    dependencies: [
        .package(name: "FlutterFramework", path: "../FlutterFramework"),
        .package(url: "https://github.com/payjp/payjp-ios.git", from: "2.2.3"),
        .package(url: "https://github.com/google/GoogleUtilities.git", from: "8.0.0")
    ],
    targets: [
        .target(
            name: "payjp_flutter",
            dependencies: [
                .product(name: "FlutterFramework", package: "FlutterFramework"),
                .product(name: "PAYJP", package: "payjp-ios"),
                .product(name: "GULAppDelegateSwizzler", package: "GoogleUtilities")
            ]
        )
    ]
)
```

### 3.4 podspec の維持（後方互換）

- **ソースコードの実体は 1 セット**（`ios/payjp_flutter/Sources/payjp_flutter/`）で SPM/Cocoapods 双方から参照する
- `ios/payjp_flutter.podspec` は削除せず、`source_files` のパスを新配置に書き換えて維持

```ruby
# 変更前
s.source_files = 'Classes/**/*'
s.public_header_files = 'Classes/**/*.h'

# 変更後
s.source_files = 'payjp_flutter/Sources/payjp_flutter/**/*.swift'
# public_header_files は Obj-C 廃止により不要
```

- 参照関係のまとめ

| 経路 | エントリポイント | ソース参照先 |
|---|---|---|
| SPM | `ios/payjp_flutter/Package.swift` | `Sources/payjp_flutter/*.swift` |
| Cocoapods | `ios/payjp_flutter.podspec` | `payjp_flutter/Sources/payjp_flutter/*.swift` |

- Cocoapods 版の依存 `PAYJPFlutterCore` は現状維持（軽量版のため）
- SPM 側は iOS SDK の `PAYJP` product を利用（`PAYJPFlutterCore` は SPM 未公開のため）
- `ios/Classes/` は空になるため削除する

### 3.5 依存バージョン管理

- `sdkconfig.json` は **CocoaPods と Android 用として利用継続**
- `Package.swift` は依存バージョンをハードコード
- iOS SDK 更新時は **`sdkconfig.json` と `Package.swift` の 2 箇所を更新**する運用

### 3.6 Flutter SDK バージョン制約

- `pubspec.yaml` の `flutter: ">=3.38.1"` は**維持**（利用者影響を最小化）
- 例外: `example/` は Flutter 3.44+ を要求（SPM 動作検証のため）
- 動作モード:
  - Flutter 3.38.1〜3.43 の利用者 → CocoaPods 経由（Package.swift は無視される）
  - Flutter 3.44+ の利用者 → SPM 経由（デフォルト）

### 3.7 iOS deployment target

- `ios.deployment_target = 12.0` を**維持**
- iOS SDK 側の SPM は iOS 10.0+ を要求するのみで、上限制約なし
- 利用者アプリの Deployment Target 要件に変更なし

## 4. 影響評価

### 4.1 プラグイン利用者への影響

| 項目 | 影響 |
|---|---|
| Flutter SDK バージョン | **影響なし**（`>=3.38.1` 維持） |
| iOS Deployment Target | **影響なし**（12.0 維持） |
| API 変更 | **なし**（Dart 側 API は変わらない） |
| ビルドサイズ | SPM 利用時のみ、`PAYJPFlutterCore`（軽量版）ではなく `PAYJP`（フル版）を含むため微増の可能性 |

### 4.2 リスク・懸念

- **iOS SDK の SPM に `PAYJPFlutterCore` product が未公開**
  - 対応 A（本方針）: SPM 版はフル `PAYJP` product を使う → バイナリサイズ増の可能性
  - 対応 B（将来検討）: iOS SDK 側に `PAYJPFlutterCore` product 追加を依頼
- SPM/CocoaPods 双方の維持で **2 箇所（Package.swift, podspec）の同期作業**が発生
- Swift/Obj-C 変換で挙動差異が発生しないよう、`GoogleUtilities/AppDelegateSwizzler` の URL スキーム処理を重点的にテスト必要

## 5. 実装ステップ

1. **Obj-C → Swift 書き換え**
   - `PayjpFlutterPlugin.{h,m}` 削除、Swift 側に統合
   - `PayjpAppDelegateInterceptor.{h,m}` を Swift 化
2. **ディレクトリ再配置**
   - `ios/Classes/*.swift` を `ios/payjp_flutter/Sources/payjp_flutter/` へ移動
3. **Package.swift 作成**
4. **podspec 更新**
   - `source_files`, `public_header_files` のパス修正
5. **`.gitignore` 更新**
   - `.build/`, `.swiftpm/` 追加
6. **example アプリの Flutter バージョン引き上げ + 動作検証**
   - SPM 有効: `flutter run` / Xcode で Package Dependencies 確認
   - SPM 無効: `flutter config --no-enable-swift-package-manager` で従来通り動作確認
   - `pod lib lint ios/payjp_flutter.podspec` で podspec 妥当性確認
7. **README / CHANGELOG 更新**

## 6. 未確定事項・今後の判断

- iOS SDK 側に `PAYJPFlutterCore` の SPM product 追加を依頼するか（バイナリサイズ影響が有意な場合）
- macOS 対応の要否（現状 iOS のみサポート、変更予定なし）

## 7. 参考リンク

- [Flutter Swift Package Manager for plugin authors](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)
- [payjp/payjp-ios](https://github.com/payjp/payjp-ios)
- [Apple: PackageDescription API](https://developer.apple.com/documentation/packagedescription)
