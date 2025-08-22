---
layout: ../../layouts/BlogLayout.astro
title: iOS 26 Beta で Firestore 通信時に TCP接続エラー
description: Xcode 26
tags: ["Swift", "Firebase"]
time: 5
featured: true
timestamp: 2025-08-22T13:31:03+00:00
filename: ios26tcp
---

## 発生事象
- 実機：iPhone 16 / iOS 26 beta
- Firebase Auth は正常に動作しており、UID 取得は可能
- Firebase 側のセキュリティルールや UID 管理、Xcode 設定などを確認済み

### エラーログ

```log
nw_connection_get_connected_socket_block_invoke [C2] Client called nw_connection_get_connected_socket on unconnected nw_connection
nw_endpoint_flow_failed_with_error [C2 2404:6800:400a:80b::200a.443 failed parent-flow (unsatisfied (No network route), ipv4, dns)] already failing, returning
TCP Conn 0x1220041e0 Failed : error 0:50 [50]
```

--

## 結論
- 原因は iOS 26 における **Network / TLS スタックの初期化タイミングの変更**
- 処理の開始時に TLS 初期化が遅れると、Firestore の最初のリクエストが失敗するため、暫定対応としてアプリ起動時に TLS スタックを先に初期化する

```swift
import Network

@main
struct MyApp: App {
    init() {
        // TLS スタックを早期初期化
        _ = nw_tls_create_options()
    }

    var body: some Scene {
        WindowGroup {
            FooView()
        }
    }
}
```

---

## 解説
1. **TLS / Network フレームワークの仕様変更**
   - iOS 26 ではデフォルト TLS バージョンが 1.0 → 1.2 に変更
   - Firebase SDK は TLS 1.2 対応済みだが、初回接続時のスタック初期化が遅れると NWPath の確立に失敗する
   - その結果、nw_endpoint_flow_failed_with_error などのネットワークエラーが発生
2. **`nw_tls_create_options()` の役割**
   - Network フレームワークの TLS オプションを初期化
   - これにより TLS スタックが事前に準備され、Firestore の初回接続が確実に成功する
   - 同様の手法は iOS / tvOS のベータ版環境でも報告されており、暫定回避策として有効
  
### 補足

- iOS 26 では従来の TLS 初期化タイミングと異なるため、Firebase Firestore / Auth など TLS 必須の通信は初回接続が不安定になりやすい
- 特に Wi-Fi や VPN の設定がない場合でも、NWPath 確立の前に SDK がリクエストを送ると接続失敗になる

---

## まとめ
- iOS 26 / Xcode 26 環境で Firebase Firestore 初回接続が失敗する場合、**ネットワークスタック初期化が原因**
- 正式リリース版では Firebase SDK 側で対応されることが期待される

---

## 参考文献
1. [Qiita: Xcode 26, iOS 26 でアプリ起動時にクラッシュした話](https://qiita.com/KaitoMuraoka/items/b4253a78013614741257)
2. [Peer-to-peer connection issues on tvOS/iPadOS](https://developer.apple.com/forums/thread/797495)


