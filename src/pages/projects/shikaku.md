---
layout: ../../layouts/ProjectLayout.astro
title: 🫎 鹿く & ShikakuMLS 企画要旨
description: エンドツーエンド暗号対応コミュニケーションプラットフォームと端末完結MLSライブラリ
tags: ["Google Cloud", "CryptoKit", "E2EE", "MLS"]
timestamp: 2025-09-18T11:46:00+00:00
featured: false
filename: shikaku
---

<img width="1024" height="1024" alt="Generated Image September 19, 2025" src="https://github.com/user-attachments/assets/b1143078-0a46-4936-b9a7-3a8499c10892" />

## 1. プロジェクト概要

「鹿く」と「ShikakuMLS」は、セキュアで責任あるテキストコミュニケーションを実現する2つの主要コンポーネントです。

| 名称      | 役割・機能 |
| :-------- | :--------- |
| **鹿く** | ユーザーインターフェース（UI/UX）、メッセージのPush/Pull制御、SwiftDataによる暗号文・メタデータ管理、表示時復号、AIによるメッセージ裁定ロジックを提供するコミュニケーションプラットフォーム |
| **ShikakuMLS (OSS)** | 端末内完結型E2EEライブラリ。1:1通信向けDouble Ratchet、グループ通信向けMLSプロトコル、鍵管理、Forward Secrecy保証、暗号化・復号処理を提供するOSSライブラリ |

---

## 2. 主体

| 内部名称 | ユーザー向けラベル例 | 最大人数 | 用途 |
| -------- | ------------------ | -------- | ---- |
| **ユーザ** | 個人 | 1 | 1:1日次会話、デイログチケット管理 |
| **トループ (Troop)** | 小規模グループ | 14 | デイログチケット管理、日次雑談・プライベート用途 |
| **コホート (Cohort)** | チーム／プロジェクト | 500 | トピックチケット管理、中長期的議論・プロジェクト単位 |

---

## 3. チケット種類と運用

### デイログチケット

- 日次単位で会話を記録
- 当日に初回メッセージ送信時に自動起票
- 運用主体: ユーザ1人との1:1、またはトループ (最大14人)
- 当日分のみ書き込み可能（自動日次締め）
- 参加メンバー全員が閲覧可能

### トピックチケット

- 特定の議題・プロジェクトに紐づく
- 運用主体: コホート (最大500人)
- 日次締めなし（長期ディスカッション可）
- アクセス制御可能（特定人物、役職、全公開など）

---

## 4. UI設計方針

- ホームビュータブ構成

1. プロフィールと所属トループ一覧
2. デイログ（1:1・トループ会話のカレンダー表示）  
3. コホート別トピック（トピックチケット管理）  
4. 登録済みユーザ一覧
5. 設定

- デイログとトピックコホートのUIは明確に区別
- Push権/Pushライセンス制度およびAI裁定結果を自然に表示
- bot呼び出し権限はチケット作成時に設定可能

---

## 5. データ保存設計

| 保存対象 | 内容 | 形式 | TTL/期限 | 備考 |
| -------- | ---- | ---- | -------- | ---- |
| 暗号化メッセージ | AES-GCM暗号文 | バイナリ | 30〜90日 | 平文は表示時のみ復号、ビジネスプランで永続化オプション |
| メタデータ | UID、チケットID、送信日時、署名ハッシュ等 | JSON | 永続(WORM) | 検索・フィルタリング用途 |
| AI裁定ログ | 承認結果、逸脱スコア、推敲履歴 | JSON | 最大90日 | ビジネスプランでは永続化オプション |
| Push権履歴 | 回転情報、リマインドログ | JSON | 約90日 | 公平性UX維持、運用履歴の透明性確保 |
| タイムスタンプ証跡 | RFC3161準拠TS、Ed25519署名 | JSON/署名 | 永続(WORM) | メタデータと共にメッセージの非否認性保証 |

- SwiftDataには暗号化メッセージと検索・表示用メタデータを保存  
- 平文は端末メモリ上で復号、一時的に処理され永続化しない

---

## 6. 暗号化戦略

| 通信形式 | プロトコル | 鍵交換 | 対称暗号 | 署名・非否認性 |
| -------- | ---------- | ------ | -------- | -------------- |
| 1:1通信 | Double Ratchet | X25519 | AES-GCM | Ed25519 |
| グループ (3〜21人) | MLS (TreeKEM + Sender Key) | X25519 | AES-GCM | Ed25519 |

- メッセージ平文は表示時のみ端末で復号  
- MLSによりForward SecrecyとPost-Compromise Securityを保証  
- ShikakuMLS公開API経由で暗号化・復号処理を実行

---

## 7. ShikakuMLSの開発企画

- Swift + CryptoKitで実装、OSSとしてApache 2.0ライセンスで公開予定
- 1:1通信用 Double Ratchet、グループ通信用 MLS (RFC9420準拠) をサポート
- メッセージ平文を永続化しない設計
- Forward Secrecy、グループ加入/脱退後の鍵更新を自動管理

---

## 8. 技術スタック

| 層 | 技術 | 備考 |
| --- | --- | --- |
| クライアント | Swift 6, SwiftUI, CryptoKit, SwiftData | 最新Appleエコシステム |
| AI裁定 | Rakuten AI 2.0 mini (CoreML) | 端末内推論、低遅延・低コスト・プライバシー保護 |
| 暗号 | ShikakuMLS (Swift Package, CryptoKit) | 端末完結型E2EE、OSSライブラリ |
| サーバー | Google Cloud (Cloud Functions, Firestore, KMS/HSM) | メタデータ・証跡保存管理 |

---

## 9. 鹿く本体処理フロー

1. ユーザー登録・フレンド追加（公開鍵交換含む）  
2. チケット作成（デイログ or トピックチケット）  
3. メッセージ作成・AI裁定  
4. メッセージPush（shikakuMLS暗号化メッセージ + メタデータをCloud Run Functions経由で登録）  
5. 受信者Pull（暗号文とメタデータをSwiftDataに保存）  
6. 表示時復号（shikakuMLS復号）  
7. Push権回転・履歴記録  
8. タイムスタンプ証跡生成・保存（RFC3161準拠、Ed25519署名）

---

## 10. 今後の拡張

- 多言語対応UI/UX・AI裁定多言語化  
- 組織向けダッシュボード（品質・チケット消化状況可視化）  
- 文章力向上コンテンツ（AI裁定連動）  
- ShikakuMLS高度機能（群脱退時鍵更新自動検証、PCS保証強化）

## 11. 補遺

### 11.1 端末内AI裁定モデル実装例

```swift
import Foundation
import CoreML

final class LLMManager {
    static let shared = LLMManager()
    
    // Rakuten AI 2.0 mini用
    private let modelFileName = "rakutenai-2mini-quantized.mlmodelc"
    private let remoteModelURLString = "https://huggingface.co/Rakuten/RakutenAI-2mini-quantized/resolve/main/rakutenai-2mini-quantized.mlmodel"
    
    private var mlModel: MLModel?
    
    private var localURL: URL? {
        do {
            let fileManager = FileManager.default
            let appSupportDir = try fileManager.url(for: .applicationSupportDirectory, in: .userDomainMask, appropriateFor: nil, create: true)
            return appSupportDir.appendingPathComponent(modelFileName)
        } catch {
            print("ApplicationSupportDirectory取得失敗: \(error)")
            return nil
        }
    }
    
    // MARK: - モデル準備
    func prepareModel(completion: @escaping (Error?) -> Void) {
        guard let localURL = localURL else {
            completion(NSError(domain: "LLMManager", code: 0, userInfo: [NSLocalizedDescriptionKey: "モデル保存先不明"]))
            return
        }
        
        if FileManager.default.fileExists(atPath: localURL.path) {
            loadCoreMLModel(from: localURL, completion: completion)
            return
        }
        
        downloadAndCompileModel(to: localURL, completion: completion)
    }
    
    private func downloadAndCompileModel(to localURL: URL, completion: @escaping (Error?) -> Void) {
        guard let url = URL(string: remoteModelURLString) else {
            completion(NSError(domain: "LLMManager", code: 1, userInfo: [NSLocalizedDescriptionKey: "URL不正"]))
            return
        }
        
        let task = URLSession.shared.downloadTask(with: url) { tempURL, _, error in
            if let error = error {
                completion(error)
                return
            }
            guard let tempURL = tempURL else {
                completion(NSError(domain: "LLMManager", code: 2, userInfo: [NSLocalizedDescriptionKey: "ダウンロード失敗"]))
                return
            }
            
            do {
                if FileManager.default.fileExists(atPath: localURL.path) {
                    try FileManager.default.removeItem(at: localURL)
                }
                try FileManager.default.moveItem(at: tempURL, to: localURL)
                let compiledURL = try MLModel.compileModel(at: localURL)
                self.loadCoreMLModel(from: compiledURL, completion: completion)
            } catch {
                completion(error)
            }
        }
        task.resume()
    }
    
    private func loadCoreMLModel(from url: URL, completion: @escaping (Error?) -> Void) {
        do {
            let config = MLModelConfiguration()
            
            // 端末性能に応じた設定
            if ProcessInfo.processInfo.physicalMemory < 3_000_000_000 { // 3GB未満
                config.computeUnits = .cpuOnly
                config.allowLowPrecisionAccumulationOnGPU = false
            } else {
                config.computeUnits = .all
                config.allowLowPrecisionAccumulationOnGPU = true
            }
            
            mlModel = try MLModel(contentsOf: url, configuration: config)
            completion(nil)
        } catch {
            completion(error)
        }
    }
    
    // MARK: - 推論
    func predict(text: String, chunkSize: Int = 128, completion: @escaping (String) -> Void) {
        prepareModel { error in
            if let error = error {
                print("モデル準備失敗: \(error)")
                completion("")
                return
            }
            
            guard let mlModel = self.mlModel else {
                completion("")
                return
            }
            
            DispatchQueue.global(qos: .userInitiated).async {
                var result = ""
                let chunks = text.chunked(by: chunkSize) // チャンク分割
                
                for chunk in chunks {
                    do {
                        let input = try MLDictionaryFeatureProvider(dictionary: ["prompt": chunk])
                        let output = try mlModel.prediction(from: input)
                        let textOutput = output.featureValue(for: "text")?.stringValue ?? chunk
                        result += textOutput
                    } catch {
                        print("推論失敗: \(error)")
                        result += chunk
                    }
                }
                
                DispatchQueue.main.async {
                    completion(result)
                }
            }
        }
    }
    
    // MARK: - モデル解放
    func unloadModel() {
        mlModel = nil
    }
}

// MARK: - 文字列チャンク化拡張
extension String {
    func chunked(by size: Int) -> [String] {
        var chunks: [String] = []
        var startIndex = self.startIndex
        while startIndex < self.endIndex {
            let endIndex = self.index(startIndex, offsetBy: size, limitedBy: self.endIndex) ?? self.endIndex
            chunks.append(String(self[startIndex..<endIndex]))
            startIndex = endIndex
        }
        return chunks
    }
}
```
