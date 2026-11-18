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
