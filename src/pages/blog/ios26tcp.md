---
layout: ../../layouts/BlogLayout.astro
title: iOS 26 Beta で Firestore 通信時に nw_connection_get_connected_socket_block_invoke
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
- Firestore への接続が失敗して `InventoryView` が永遠に Loading のまま
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
- 処理の開始時に TLS 初期化が遅れると、Firestore の最初のリクエストが失敗

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
   - Firebase SDK 自体は TLS 1.2 対応済みだが、初回接続時のスタック初期化が遅いと失敗する
2. **`nw_tls_create_options()` の役割**
   - Network フレームワークの TLS オプションを初期化
   - これにより TLS スタックが先に準備され、Firebase の最初の接続が成功する

---

## まとめ
- iOS 26 / Xcode 26 環境で Firebase Firestore 初回接続が失敗する場合、**ネットワークスタック初期化が原因**
- 正式リリース版では Firebase SDK 側で対応されることが期待される
