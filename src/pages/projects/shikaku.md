---
layout: ../../layouts/ProjectLayout.astro
title: 🫎 鹿く & ShikakuMLS 企画要旨
description: エンドツーエンド暗号対応コミュニケーションプラットフォームと端末完結MLSライブラリ
tags: ["Google Cloud", "CryptoKit", "E2EE", "MLS"]
timestamp: 2025-09-18T11:46:00+00:00
featured: true
filename: shikaku
---

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

*   **Pull主体UX**: 受信者が自身のペースで情報を取得するモデルを採用することで、情報過多による疲弊を軽減します。
*   **AI裁定によるPush承認制**: 送信されるメッセージはAIによる事前裁定を受け、承認されたもののみが送信可能です。誤字脱字や論理的な逸脱が検知された場合、「鹿られ」として修正が求められます。
*   **責任あるコミュニケーション文化**: Push権とPushライセンス制度を導入。連続Pushの制限やAI裁定の承認プロセスを通じて、ユーザーの発言に対する内面的な責任感を醸成します。
*   **チケット単位議論管理**: 課題の進捗を「通常チケット」と「Daylogチケット」の2種類のチケットで管理します。これにより情報の構造化を促進し、議論の可視性と追跡性を向上させます。

### 2.2 UX設計詳細

*   **チケットの種類**:
    *   **通常チケット**: 特定の担当者を指定し、課題管理やプロジェクトの進捗管理に利用します。
    *   **Daylogチケット**: 日次で自動生成されるフリーディスカッション用チケットです。日次でリフレッシュされるため、その日のトピックに集中できます。
*   **Push権 / Pushライセンス**:
    *   メッセージのPush（送信）には「Push権」が必要です。連続Pushは禁止され、権利は参加者間で順次委譲されます。
    *   AI裁定の承認を得ることで「Pushライセンス」が付与され、メッセージ送信が可能になります。
    *   一度Pushされたメッセージは編集不可とし、発言に対する厳格な責任感を醸成します。
*   **Pull主体情報取得**:
    *   ユーザーは自分のタイミングで新しい情報を取得します。通知はPush権が到来したことのリマインドに留め、即時応答を強制しません。
*   **メッセージ表示**:
    *   SwiftDataには暗号文と検索用のメタデータのみを保存します。
    *   メッセージ表示時にShikakuMLSを用いて端末内で復号し、平文は永続的に保存しない設計です。

---

## 3. データ保存設計

「鹿く」では、プライバシーとセキュリティを最優先し、データの種類に応じて保存方針を定めています。

| 保存対象           | 内容                                       | 形式       | TTL/期限        | 備考                                                              |
| :----------------- | :----------------------------------------- | :--------- | :-------------- | :---------------------------------------------------------------- |
| 暗号化メッセージ   | AES-GCM暗号文                              | バイナリ   | 30〜90日        | 平文は保存せず表示時復号。ビジネスプランでは永続化オプション提供。 |
| メタデータ         | UID、チケットID、送信日時、署名ハッシュ等 | JSON       | 永続(WORM)      | 検索・フィルタリング用途。改ざん不可、証跡性保証。                |
| 鹿られログ         | 承認結果、逸脱スコア、指摘箇所、推敲履歴   | JSON       | 最大90日        | 推敲履歴はユーザーの学習・改善に活用。ビジネスプランでは永続化オプション提供。 |
| Push権履歴         | Push権の順番回転情報、リマインドログ       | JSON       | 約90日          | 公平性UX維持、運用履歴の透明性確保。                              |
| タイムスタンプ証跡 | RFC3161準拠TS、Ed25519署名                 | JSON/署名 | 永続(WORM)      | メタデータと共にメッセージの非否認性を保証。法的効力あり。        |

**SwiftDataへの保存方針**:
SwiftDataには、暗号化されたメッセージ本文 (`EncryptedMessage`構造体) に加え、検索・フィルタリング・表示に必要な非機密性のメタデータ（例: `senderUserID`, `ticketID`, `timestamp`, `messageHashForIntegrity`）を別途保存します。これにより、平文を永続化することなく、ユーザー体験を損なわない効率的な情報アクセスを実現します。

---

## 4. 暗号化戦略

「鹿く」のE2EEは、通信形式に応じてShikakuMLSライブラリを通じて最適なプロトコルを適用します。

| 通信形式           | プロトコル                 | 鍵交換 | 対称暗号 | 署名・非否認性 |
| :----------------- | :------------------------- | :----- | :------- | :------------- |
| **1:1通信**        | Double Ratchet             | X25519 | AES-GCM  | Ed25519        |
| **グループ (3〜21人)** | MLS (TreeKEM + Sender Key) | X25519 | AES-GCM  | Ed25519        |

*   「鹿く」本体は、ShikakuMLSの公開APIを呼び出すことで暗号化・復号処理を実行します。
*   メッセージ平文は表示時のみ端末メモリ上で復号され、永続的なストレージには保存されません。
*   MLSプロトコルにより、Forward Secrecy（過去のセッション鍵が漏洩しても未来のメッセージは解読されない）と、グループメンバー変更後の過去メッセージ復号不可（Post-Compromise Securityの一部）を保証します。

---

## 5. ShikakuMLSライブラリ設計

ShikakuMLSは、Swift言語とApple CryptoKitのみで実装される端末完結型E2EEライブラリであり、オープンソースソフトウェア（OSS）として公開されます。

*   **目的**: モバイルアプリケーションにE2EE機能を手軽かつセキュアに組み込むためのSwiftライブラリです。
*   **対応プロトコル**:
    *   1:1通信向けの[Double Ratchet Protocol](https://signal.org/docs/specifications/doubleratchet/)。
    *   グループ通信向けの[Messaging Layer Security (MLS) Protocol](https://messaginglayersecurity.com/) (RFC9420準拠を目標)。
*   **主な公開API**:
    *   `PublicKey` / `PrivateKey` ラッパー型: 暗号プリミティブの鍵を抽象化し、型安全な操作を保証。
    *   `encryptMessage`: メッセージの暗号化。Replay Attack対策や文脈整合性チェックのためにコンテキスト情報も付加可能。
    *   `decryptMessage`: 暗号化されたメッセージの復号。
    *   `createGroup`, `addMember`, `removeMember`, `processCommit`: グループ通信におけるMLSプロトコル（グループの作成、メンバーの追加・削除、グループ状態の更新）をサポート。
*   **特徴**:
    *   Swift + CryptoKitのみで実装され、外部ライブラリへの依存を最小限に抑えます。
    *   メッセージ平文は永続化しない設計思想をライブラリレベルで保証します。
    *   厳格な単体テスト、Forward Secrecyテスト、過去メッセージ復号不可テスト、グループ加入/脱退後の鍵更新テストを独立して実施します。
    *   **OSS公開予定**: Apache 2.0ライセンスで公開されます。

**鍵管理・更新方針**:
ShikakuMLS内部では、Diffie-Hellman鍵交換によって生成されるセッション鍵をワンタイム利用し、メッセージ送受信ごとや一定時間ごとといった定期的な鍵更新によってForward Secrecyを保証します。MLSにおいてはTreeKEMメカニズムがグループ鍵の効率的かつセキュアな更新を担います。

---

## 6. 技術スタック

「鹿く」および「ShikakuMLS」は、最新かつセキュアな技術スタックを採用しています。

| 層           | 技術                                 | 備考                                 |
| :----------- | :----------------------------------- | :----------------------------------- |
| **クライアント** | Swift 6, SwiftUI, CryptoKit, SwiftData | 最新のAppleエコシステム技術を採用。  |
| AI裁定 | Rakuten AI 2.0 mini (CoreML) | 端末内での推論により送信メッセージの裁定とPush承認を担当。外部API不要で低遅延・低コスト・プライバシー保護が可能。 |
| **暗号**     | ShikakuMLS (Swift Package, CryptoKit) | 自社開発OSSライブラリ、端末完結型E2EEを提供。 |
| **サーバー**   | Google Cloud (Cloud Functions, Firestore, KMS/HSM) | スケーラビリティとセキュリティを重視、メタデータ・証跡保存を管理。 |

---

## 7. サービス収益モデル

カジュアルユーザーからエンタープライズ顧客までをターゲットとした柔軟な収益モデルを展開します。

| 機能                       | カジュアルプラン | ビジネスプラン                         |
| :------------------------- | :--------------- | :------------------------------------- |
| 鹿り機能                   | ○                | ○                                      |
| Push権/Pushライセンス制度  | ○                | ○                                      |
| 高度チケットシステム       | ○                | ○                                      |
| グループ参加人数上限       | 20人             | 無制限                                 |
| 暗号メッセージ保存期間     | 30〜90日         | 無制限 (長期保存オプション)            |
| AI裁定ログ永続保存         | ✕                | ○                                      |
| WORMメタデータ永続化       | ✕                | ○                                      |
| タイムスタンプ証跡生成・提供 | ✕                | ○ (法務・監査対応)                     |
| HSM鍵管理                  | ✕                | ○ (M-of-N秘密分散, Proxy Re-encryption, Threshold Decryption) |
| 監査証跡・法的対応レポート | ✕                | ○                                      |
| 専用API連携・Webhook       | ✕                | ○                                      |
| 文字アイコンアセット販売   | ○                | ○                                      |

### 7.1 マーケティングポイント

*   **端末完結E2EE**: メッセージ平文は端末に残さず表示時復号。データプライバシーとセキュリティを最大限に保護します。
*   **Double Ratchet + MLS**: 1対1通信とグループ通信で業界標準の最先端プロトコルを適用し、強固なForward Secrecyを保証します。
*   **AI裁定 + Push権制度**: 品質が高く、責任感のあるコミュニケーションを促進するユニークなUXを提供します。
*   **OSSライブラリ提供**: ShikakuMLSを独立したOSSとして公開することで、E2EE実装の信頼性と透明性を確保し、技術コミュニティへの貢献を目指します。
*   **法務・監査対応**: ビジネスプランでは、法的効力のあるタイムスタンプ証跡や監査ログを提供し、エンタープライズの要件に対応します。

---

## 8. 鹿く本体処理フロー

以下に、メッセージの送受信と関連する処理フローを示します。

1.  **ユーザー登録 / フレンド追加**: ユーザーは自身の公開鍵を生成し、サーバーに登録。フレンド追加時に公開鍵を交換します。
2.  **チケット作成**: ユーザーはチケットを作成し、参加メンバーを指定します。
3.  **メッセージ作成**:
    *   ユーザーがメッセージを入力します。
    *   AI裁定システムがメッセージ内容を解析し、品質を評価します。
    *   AI裁定を通過したメッセージはShikakuMLSに渡され、受信者（またはグループ）の公開鍵を用いて暗号化されます。
4.  **メッセージPush**:
    *   暗号化されたメッセージ (`EncryptedMessage`) と関連メタデータがCloud Functions経由でメッセージキューに登録されます。
    *   受信者へ通知（リマインド）が送られます。
5.  **受信者Pull**:
    *   受信者がアプリを起動または更新すると、新しい暗号文メッセージとメタデータをプルします。
    *   SwiftDataに暗号文と検索用メタデータを保存します。
6.  **表示時復号**:
    *   メッセージは表示時に端末内で復号され、平文は一時的にメモリ上で処理され永続化されません。
7.  **Push権ライセンスの回転・記録**:
    *   メッセージPush後、Push権の順番が更新され、その履歴が記録されます。
8.  **タイムスタンプ証跡生成・保存**:
    *   サーバー側でメッセージメタデータにRFC3161準拠のタイムスタンプとEd25519署名を付与し、非否認性を保証して永続保存します。

---

## 9. 今後の拡張

*   **多言語対応**: グローバル展開に向けたUI/UXおよびAI裁定の多言語化を推進します。
*   **組織内ダッシュボード**: 企業・組織向けに、コミュニケーションの品質、チケット消化状況、AI裁定傾向などを可視化する管理ダッシュボードを提供します。
*   **文章力向上コンテンツ**: 『理科系の作文技術』などの論理的思考・文章作成術をベースとした学習コンテンツを提供し、AI裁定と連携してユーザーの表現能力向上を支援します。
*   **ShikakuMLSの高度機能**: MLSにおける群脱退時の鍵更新自動検証や、Post-Compromise Securityのより厳密な保証機能などを実装予定です。

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
