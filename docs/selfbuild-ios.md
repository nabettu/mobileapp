# 自分の iPhone 用にセルフビルドする (フォーク nabettu/mobileapp の selfbuild-ios ブランチ)

公式手順 (README の iOS 節) からの変更点だけを書く。

- Bundle ID は `com.nabettu.pebbleself`、Team は CROSSEDGE (`2KDR72P2RA`)。`BUNDLE_ID` は `Config.xcconfig` だけでなく `project.pbxproj` の target 設定にも直書きされているので両方変える
- entitlements は HealthKit だけに減らした (Apple Sign In・Push・Associated Domains は公式の Team / ドメインでしか使えない)
- Firebase は `GoogleService-Info-dummy.plist` をコピーして使う (git 管理外)。ログイン・クラウドのバックアップ・Nenya (公式のクラウド AI) は動かない
- Xcode に Apple ID が無いので、署名は ASC の API キーで取る:
  `xcodebuild ... -allowProvisioningUpdates -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_A2M34D6GQG.p8 -authenticationKeyID A2M34D6GQG -authenticationKeyIssuerID <ISSUER>`
- ビルド: `./gradlew podInstall` (約 6 分) → `xcodebuild -workspace iosApp/iosApp.xcworkspace -scheme iosApp -destination 'generic/platform=iOS' -derivedDataPath build-ios build`
- 入れる: `xcrun devicectl device install app --device <UDID> build-ios/Build/Products/Debug-iphoneos/Pebble.app`

## Index 01 (リング) について読んだこと (2026-11-18)

- ペアリングは `libindex/.../IndexPairing.kt` (`RealIndexPairing`)。BLE の bond を作り、成功したらリング内の録音を消してから「ペア済み」にする。Firebase のログインは要らない (`experimental/.../RingPairing.kt` の uid 必須の旧経路はどこからも呼ばれていない)
- 文字起こしは `HybridTranscriptionService`。端末内の Cactus モデル (LocalOnly) なら公式のクラウド無しで動く。Wispr / Kirinki はトークンが要る
- メモの AI 処理は Nenya (公式クラウド、Firebase の ID トークンで認証) → 失敗時は端末内の Cactus に落ちる (`AgentFactory`)

## ペアリングと音声取得だけを別アプリに移せるか (2026-11-18 調査)

結論: 移せる。中核は公開バイナリのライブラリなので、自前のつなぎコードは数百行で済む見込み。

- リングとの通信 (スキャン・接続・録音の転送・デコード) は全部 `io.github.coredevices.haversine:haversine` (Maven Central、版 `f3b7a61`、**Apache-2.0**) の中。
  ソース (github.com/coredevices/haversine-kmp) は非公開。iOS 版は Swift 製の静的ライブラリ `libHaversineSatelliteLibrary.a` を cinterop で包んだ KMP ライブラリ
  - 依存は kermit・ktor-client・coroutines・kotlinx-io・serialization だけ (Firebase・Room・Compose は要らない)
  - Swift から直接 `.a` を使うのはヘッダ / swiftmodule が同梱されていないので難しい。**小さな KMP の共有モジュールを作って framework にし、Swift アプリから呼ぶ**のが現実的
- 自前で用意するもの (アプリ側の実装を見て数えた):
  - `KMPHaversineSatelliteManager(pairedSatelliteIdProvider, debugDelegate, hacksDelegate, collectionIndexStorage, hwVersion = 11 to 0, scope, updateJsonProvider)` を作る (`experimental/src/iosMain/.../ringModule.ios.kt`)
  - `CollectionIndexStorage` (どの番号まで受け取ったか。StateFlow 1 つと setter)・`KMPHaversineHacksDelegate` (2 メソッド)・`KMPHaversineDebugDelegate` (3 メソッド、空でよい)
  - ペアリング: iOS は暗号化が要る特性 (サービス `607B5C9B-...`, `DAAD3D52-...`) に書いて OS に bond を作らせる (`libindex/.../IndexPairing.ios.kt`、kable を使用)。成功後に `eraseCollections()`。ファーム 3.62.0 未満は `programSatelliteWithUserID` も要る
  - 受信: `startScanning()` の Flow で `SatelliteStatus.Transferring` → `TransferStatus.TransferComplete` に `samples` (ShortArray の PCM) と `sampleRate` が来る。アプリは `removeDCBias` してから 16kHz に resample している (`RingSync.kt`)
  - 背景で受け取り続けるには `bluetooth-central` のバックグラウンドモードが要る
- ライセンス: haversine は Apache-2.0 なので自由に使える。**mobileapp のコード (libindex・RingSync 等) を写すと新アプリも GPLv3** になる (App Store 配布は README の追加許諾で MPL-2.0 可)。避けるなら haversine の API だけ見て書き直す
- 未確認: OTA の取得先 (`updateJsonProvider` は null 返しで動くか)・ペアリングキーのハッシュ確認 (`RingPairingKeyMonitor`) を省いて問題ないか・リングと実機で試していない
