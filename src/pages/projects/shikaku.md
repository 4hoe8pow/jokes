---
layout: ../../layouts/ProjectLayout.astro
title: 🫎 鹿く & ShikakuMLS 企画要旨
description: エンドツーエンド暗号対応コミュニケーションプラットフォームと端末完結MLSライブラリ
tags: ["Google Cloud", "CryptoKit", "E2EE", "MLS"]
timestamp: 2025-09-18T11:46:00+00:00
featured: true
filename: shikaku
---

<img width="1024" height="1024" alt="Generated Image September 19, 2025" src="https://github.com/user-attachments/assets/b1143078-0a46-4936-b9a7-3a8499c10892" />

## 1. プロジェクト概要

「鹿く」と「ShikakuMLS」は、セキュアで責任ある新しいコミュニケーション文化を創造するための2つの主要プロジェクトです。

| プロジェクト名      | 役割・機能                                                                                                                                                                                                                                                                                                 |
| :------------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **鹿く**            | ユーザーインターフェース（UI/UX）、メッセージのPush/Pullフロー制御、SwiftDataによる暗号文・メタデータ管理、表示時のメッセージ復号、AIによるメッセージ裁定ロジックを担うコミュニケーションプラットフォーム。                                                                                                       |
| **ShikakuMLS (OSS)** | 端末内で完結するE2EEコアライブラリ。1対1のDouble Ratchetプロトコル、グループ向けMLSプロトコルの実装、鍵管理、Forward Secrecy（前方秘匿性）保証、メッセージの暗号化・復号処理を提供するオープンソースソフトウェアです。                                                                                            |

---

## 2. “鹿く”とは

「鹿く」は、**思考の圧縮と表現責任**を核とし、情報過多を避けつつ、責任感ある対話文化の醸成を目指すプロダクトです。

### 2.1 主要コンセプト

*   **Pull主体UX**: 受信者が自身のペースで情報を取得するモデルを採用し、情報過多による疲弊を軽減します。
*   **AI裁定によるPush承認制**: 送信されるメッセージはAIによる事前裁定を受け、承認されたもののみが送信可能です。「鹿られ」として修正が求められる場合もあります。
*   **責任あるコミュニケーション文化**: Push権とPushライセンス制度を導入。連続Pushの制限やAI裁定の承認プロセスを通じ、ユーザーの発言責任を強化します。
*   **チケット単位議論管理**: 2種類のチケット（Daylogチケット／トピックチケット）と3種類の主体（ユーザー／ユーザコホート／トピックコホート）に基づき、情報の構造化と議論の追跡性を向上します。

### 2.2 UX設計詳細

*   **チケットの種類**:
    *   **Daylogチケット**: 日次で締められる雑談・日次ログ用チケット。対象ユーザーまたはコホートに初投稿時に作成され、同じ日・同じ対象者には1件のみ生成。カレンダーUIで時系列管理。
    *   **トピックチケット**: 特定のプロジェクトやテーマに関連するチケット。作成者がクローズするまでオープンで、トピックコホートに紐付く。ガントチャート風リストUIで管理。
*   **主体分類**:
    *   **ユーザー**: 1対1の通信対象。
    *   **ユーザコホート**: 友人グループや常設チーム。日次Daylog通信も可。
    *   **トピックコホート**: 有期性のあるプロジェクトや限定テーマ用。トピックチケットを管理。
*   **Push権 / Pushライセンス**:
    *   メッセージのPushには「Push権」が必要。順番回転制で連続Push不可。
    *   AI裁定により「Pushライセンス」が付与され、承認済みメッセージを送信可能。
    *   Push済みメッセージは編集不可、責任感を強化。
*   **Pull主体情報取得**:
    *   ユーザーは自分のタイミングで情報取得。通知はリマインドに留め、即時応答は強制しない。
*   **メッセージ表示**:
    *   SwiftDataには暗号文とメタデータのみ保存。表示時にShikakuMLSで復号、平文は永続化しない。

---

## 3. データ保存設計

| 保存対象           | 内容                                       | 形式       | TTL/期限        | 備考                                                              |
| :----------------- | :----------------------------------------- | :--------- | :-------------- | :---------------------------------------------------------------- |
| 暗号化メッセージ   | AES-GCM暗号文                              | バイナリ   | 30〜90日        | 平文は保存せず表示時復号。ビジネスプランでは永続化オプション提供。 |
| メタデータ         | UID、チケットID、送信日時、署名ハッシュ等 | JSON       | 永続(WORM)      | 検索・フィルタリング用途。改ざん不可、証跡性保証。                |
| 鹿られログ         | 承認結果、逸脱スコア、指摘箇所、推敲履歴   | JSON       | 最大90日        | 推敲履歴は学習用。ビジネスプランでは永続化オプション提供。       |
| Push権履歴         | Push権順番回転、リマインドログ             | JSON       | 約90日          | 公平性UX維持、運用履歴の透明性確保。                              |
| タイムスタンプ証跡 | RFC3161準拠TS、Ed25519署名                 | JSON/署名 | 永続(WORM)      | メタデータと共に非否認性保証。法的効力あり。                     |

---

## 4. 暗号化戦略

| 通信形式           | プロトコル                 | 鍵交換 | 対称暗号 | 署名・非否認性 |
| :----------------- | :------------------------- | :----- | :------- | :------------- |
| **1:1通信**        | Double Ratchet             | X25519 | AES-GCM  | Ed25519        |
| **グループ (3〜21人)** | MLS (TreeKEM + Sender Key) | X25519 | AES-GCM  | Ed25519        |

* ShikakuMLSの公開APIを呼び出して暗号化／復号。
* 平文は表示時のみ端末メモリ上で復号。
* MLSによりForward Secrecy・Post-Compromise Security保証。

---

## 5. ShikakuMLSライブラリ設計

* Swift + CryptoKitで端末完結型E2EEライブラリを提供。
* **API例**:
    * `generateKeyPair(participantID:)`
    * `encryptMessage(plaintext:, recipientPublicKey:, contextInfo:)`
    * `decryptMessage(ciphertext:, recipientPrivateKey:, contextInfo:)`
    * `createCohort(memberPublicKeys:, initialState:)`
    * `addMember(cohortID:, newMemberPublicKey:, senderPrivateKey:)`
    * `getCohortState(cohortID:)`
* メッセージ平文は永続化せず、端末メモリ上で復号。
* Diffie-Hellmanセッション鍵やTreeKEMでForward Secrecyを保証。
* OSS公開予定（Apache 2.0）。

---

## 6. 技術スタック

| 層           | 技術                                 | 備考                                 |
| :----------- | :----------------------------------- | :----------------------------------- |
| クライアント | Swift 6, SwiftUI, CryptoKit, SwiftData | 最新Appleエコシステム技術             |
| AI裁定       | Rakuten AI 2.0 mini (CoreML)          | 端末内でメッセージ裁定とPush承認     |
| 暗号         | ShikakuMLS (Swift Package)            | 端末完結型E2EE                        |
| サーバー     | Google Cloud (Cloud Functions, Firestore, KMS/HSM) | メタデータ・証跡保存を管理           |

---

## 7. チケット・主体管理まとめ

| チケット種類       | 主体                     | 締め方 / 有期性                                | UI表示例                  |
| :----------------- | :----------------------- | :-------------------------------------------- | :------------------------ |
| **Daylogチケット**  | ユーザー / ユーザコホート | 日次で自動締め。対象者・日付で1件のみ作成  | カレンダーUIで時系列管理  |
| **トピックチケット** | トピックコホート         | 作成者がクローズするまでオープン。複数日跨ぎも可 | ガントチャート風リストUI |

* トピックチケットは一過性プロジェクトや限定テーマに最適。
* Daylogチケットは雑談や日次ログに最適。ユーザーの初投稿で自動生成される。

---

## 8. 鹿く本体処理フロー（チケット前提）

1. ユーザー登録 / フレンド追加  
2. チケット作成（Daylog or トピックチケット）  
3. メッセージ入力 → AI裁定 → ShikakuMLSで暗号
4. 暗号化メッセージとメタデータをサーバーにPush  
5. 受信者Pull → SwiftDataに暗号文・メタデータ保存  
6. 表示時復号（ShikakuMLSで復号）  
7. Push権回転・履歴記録
8. タイムスタンプ証跡生成・保存（非否認性保証）

---

## 9. 今後の拡張

* 多言語対応、組織向けダッシュボード、文章力向上コンテンツ  
* ShikakuMLSの高度機能（Post-Compromise Security、群脱退後の鍵更新自動検証）

## 10. 補遺

### 10.1 端末内AI裁定モデル実装例

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
