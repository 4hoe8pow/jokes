---
layout: ../../layouts/ProjectLayout.astro
title: 🫎 鹿く & ShikakuMLS 企画要旨
description: エンドツーエンド暗号対応コミュニケーションプラットフォームと端末完結MLSライブラリ
tags: ["Google Cloud", "CryptoKit", "E2EE", "MLS"]
timestamp: 2025-09-18T11:46:00+00:00
featured: true
filename: shikaku
---

## 1. プロジェクト構成

エンドツーエンド暗号化（E2EE）に対応したコミュニケーションプラットフォーム「鹿く」と、その基盤となる端末完結型MLS（Messaging Layer Security）ライブラリ「ShikakuMLS」のアーキテクチャおよび設計思想を詳述する。

| プロジェクト | 役割 |
|--------------|------|
| **鹿く** | UI/UX、Push/Pullフロー制御、SwiftDataによる暗号文・メタデータ管理、表示時復号、AI審査ロジック |
| **ShikakuMLS (OSS)** | 端末完結E2EEコアライブラリ：1:1 Double Ratchetプロトコル、グループMLSプロトコル実装、鍵管理、Forward Secrecy保証、メッセージの暗号化・復号処理 |

---

## 2. 鹿く本体設計コンセプト

「鹿く」は、**思考の圧縮と表現責任**を核とした新しい対話文化の創出を目指す。

-   **Pull主体UX**: 受信者が自身のペースで情報を取得するモデルを採用し、情報過多による疲弊を軽減。
-   **AI裁定によるPush承認制**: 送信メッセージはAIによる事前審査を経てPush承認される。これにより誤字・脱字、論理逸脱、意図せぬハラスメント等を排除し、メッセージ品質を担保。
-   **責任あるコミュニケーション文化**: Push権とPushライセンス制度を導入。連続Pushの制限やAI審査の承認プロセスを通じて、発言に対するユーザーの内面的な責任感を醸成。
-   **チケット単位議論管理**: 課題の進捗をチケット単位で管理。情報の構造化を促進し、議論の可視性と追跡性を向上。

---

### 2.1 UX設計

-   **チケット単位**:
    -   **通常チケット**: 特定の担当者を指定し、課題管理やプロジェクト進捗管理に活用。
    -   **Daylogチケット**: 日次で自動生成されるフリーディスカッション用チケット。日次リフレッシュにより、その日のトピックに集中できる設計。
-   **Push権 / Pushライセンス**:
    -   メッセージPushにはPush権が必要。連続Pushは禁止され、権利は参加者間で順次委譲される。
    -   AI審査の承認を得ることでPushライセンスが付与され、メッセージ送信が可能になる。
    -   Pushされたメッセージは編集不可とし、発言に対する厳格な責任を内面化させる。
-   **Pull主体情報取得**:
    -   ユーザーは自分のタイミングで新しい情報を取得する。通知はPush権が到来したことのリマインドに留め、即時応答を強制しない。
-   **メッセージ表示**:
    -   SwiftDataには暗号文と検索用のメタデータのみを保存。
    -   メッセージ表示時にShikakuMLSを用いて端末内で復号し、平文は永続的に保存しない設計とする。

---

### 2.2 データ保存設計

| 保存対象           | 内容 | 形式 | TTL/期限 | 備考 |
|-------------------|------|------|-----------|------|
| 暗号化メッセージ   | AES-GCM暗号文 | バイナリ | 30〜90日 | 平文は保存せず表示時復号。ビジネスプランでは永続化オプション提供 |
| メタデータ         | UID、チケットID、送信日時、署名ハッシュ等 | JSON | 永続(WORM) | 検索・フィルタリング用途。改ざん不可、証跡性保証 |
| AI審査ログ         | 承認結果、逸脱スコア、指摘箇所、推敲履歴 | JSON | 最大90日 | 推敲履歴はユーザーの学習・改善に活用。ビジネスプランでは永続化オプション提供 |
| Push権履歴         | Push権の順番回転情報、リマインドログ | JSON | 約90日 | 公平性UX維持、運用履歴の透明性確保 |
| タイムスタンプ証跡 | RFC3161準拠TS、Ed25519署名 | JSON/署名 | 永続(WORM) | メタデータと共にメッセージの非否認性を保証。法的効力あり |

**SwiftDataへの保存方針の補足**:
SwiftDataには、暗号化されたメッセージ本文(`EncryptedMessage`構造体)に加え、検索・フィルタリング・表示に必要な非機密性のメタデータ（例: `senderUserID`, `ticketID`, `timestamp`, `messageHashForIntegrity`）を別途フィールドとして保存する。これにより、平文を永続化することなく、ユーザー体験を損なわない効率的な情報アクセスを実現する。

---

## 3. 暗号化戦略

鹿くのE2EEは、通信形式に応じて最適なプロトコルをShikakuMLSライブラリを通じて適用する。

| 通信形式 | プロトコル                 | 鍵交換 | 対称暗号 | 署名・非否認性 |
|-----------|----------------------------|--------|----------|----------------|
| 1:1通信   | Double Ratchet             | X25519 | AES-GCM | Ed25519          |
| グループ (3〜21人) | MLS (TreeKEM + Sender Key) | X25519 | AES-GCM | Ed25519          |

-   鹿く本体は、ShikakuMLSのPublic APIを呼び出すことで暗号化・復号処理を実行。
-   メッセージ平文は表示時のみ端末メモリ上で復号され、永続的なストレージには保存されない。
-   MLSプロトコルにより、Forward Secrecy（過去のセッション鍵が漏洩しても未来のメッセージは解読されない）と、グループメンバー変更後の過去メッセージ復号不可（Post-Compromise Securityの一部）を保証する。

---

## 4. ShikakuMLSライブラリ設計

ShikakuMLSは、Swift言語とApple CryptoKitのみで実装される端末完結型E2EEライブラリであり、OSSとして公開される。

-   **目的**: モバイルアプリケーションにE2EE機能を手軽かつセキュアに組み込むためのSwiftライブラリ。
-   **対応プロトコル**:
    -   1:1通信向けの[Double Ratchet Protocol](https://signal.org/docs/specifications/doubleratchet/)
    -   グループ通信向けの[Messaging Layer Security (MLS) Protocol](https://messaginglayersecurity.com/) (RFC9420準拠を目標)
-   **主なPublic API**:
    -   `PublicKey` / `PrivateKey` ラッパー型: 暗号プリミティブの鍵を抽象化し、型安全な操作を保証。
    -   `encryptMessage(message: Data, recipients: [ShikakuMLS.PublicKey], senderPrivateKey: ShikakuMLS.PrivateKey, context: Data?) -> EncryptedMessage`
        -   `context`にはチケットIDなど、メッセージと紐付けたい非機密情報を付加し、Replay Attack対策や文脈整合性チェックに利用。
    -   `decryptMessage(encrypted: EncryptedMessage, recipientPrivateKey: ShikakuMLS.PrivateKey, expectedSenderPublicKey: ShikakuMLS.PublicKey) -> Data?`
    -   `createGroup(initialMembers: [ShikakuMLS.PublicKey], creatorPrivateKey: ShikakuMLS.PrivateKey) -> (GroupState, GroupCommit)`
    -   `addMember(groupState: GroupState, newMember: ShikakuMLS.PublicKey, committerPrivateKey: ShikakuMLS.PrivateKey) -> (GroupState, GroupCommit, WelcomePackage)`
        -   `GroupCommit`は新しいグループ状態への変更をコミットする情報。
        -   `WelcomePackage`は新規メンバーがグループに参加するために必要な鍵情報を含む。
    -   `removeMember(groupState: GroupState, memberToRemove: ShikakuMLS.PublicKey, committerPrivateKey: ShikakuMLS.PrivateKey) -> (GroupState, GroupCommit)`
    -   `processCommit(groupState: GroupState, commit: GroupCommit, welcomePackage: WelcomePackage?) -> GroupState`
        -   CommitメッセージとWelcomePackageを受け取り、グループ状態を更新する。
-   **特徴**:
    -   Swift + CryptoKitのみで実装され、外部ライブラリへの依存を最小限に抑える。
    -   メッセージ平文は永続化しない設計思想をライブラリレベルで保証。
    -   厳格なUnit Test、Forward Secrecyテスト、過去メッセージ復号不可テスト、グループ加入/脱退後の鍵更新テストを独立して実施。
    -   **OSS公開予定**: Apache 2.0ライセンス。

**鍵管理・更新方針の補足**:
ShikakuMLS内部では、Diffie-Hellman鍵交換によって生成されるセッション鍵をワンタイム利用し、定期的な鍵更新（例: メッセージ送受信ごと、一定時間ごと）によってForward Secrecyを保証する。MLSにおいてはTreeKEMメカニズムがグループ鍵の効率的かつセキュアな更新を担う。ライブラリのAPIを通じて、アプリケーション側で鍵の生成、保存、失効ポリシーを柔軟に設定できるように設計する。

---

## 5. 技術スタック

| 層 | 技術 | 備考 |
|-----|-----|------|
| クライアント | Swift 6, SwiftUI, CryptoKit, SwiftData | 最新のAppleエコシステム技術を採用 |
| サーバ | Google Cloud (Cloud Functions, Firestore, KMS/HSM) | スケーラビリティとセキュリティを重視 |
| 暗号ライブラリ | ShikakuMLS (Swift Package, CryptoKit) | 自社開発OSS、端末完結E2EE |
| AI審査 | Gemini API | 高度な自然言語処理とAI裁定を実現 |

---

## 6. サービス収益モデル

カジュアルユーザーからエンタープライズ顧客までをターゲットとした柔軟な収益モデルを展開する。

| 機能 | カジュアルプラン | ビジネスプラン |
|------|----------------|----------------|
| AI裁定機能 | ○ | ○ |
| Push権/Pushライセンス制度 | ○ | ○ |
| 高度チケットシステム | ○ | ○ |
| グループ参加人数上限 | 20人 | 無制限 |
| 暗号メッセージ保存期間 (短期) | 30〜90日 | 無制限 (長期保存オプション) |
| AI審査ログ永続保存 | ✕ | ○ |
| WORMメタデータ永続化 | ✕ | ○ |
| タイムスタンプ証跡生成・提供 | ✕ | ○ (法務・監査対応) |
| HSM鍵管理 (M-of-N秘密分散, Proxy Re-encryption, Threshold Decryption) | ✕ | ○ |
| 監査証跡・法的対応レポート | ✕ | ○ |
| 専用API連携・Webhook | ✕ | ○ |
| 文字アイコンアセット販売 | ○ | ○ |

---

### 6.1 マーケティングポイント

-   **端末完結E2EE**: メッセージ平文は端末に残さず表示時復号。データプライバシーとセキュリティを最大限に保護。
-   **Double Ratchet + MLS**: 1:1通信とグループ通信で業界標準の最先端プロトコルを適用し、強固なForward Secrecyを保証。
-   **AI審査 + Push権制度**: 品質高く、責任感のあるコミュニケーションを促進するユニークなUX。
-   **OSSライブラリ提供**: ShikakuMLSを独立したOSSとして公開することで、E2EE実装の信頼性と透明性を確保し、技術コミュニティへの貢献を目指す。
-   **法務・監査対応**: ビジネスプランでは、法的効力のあるタイムスタンプ証跡や監査ログを提供し、エンタープライズの要件に対応。

---

## 7. 鹿く本体処理フロー

1.  **ユーザー登録 / フレンド追加**: ユーザーは自身の公開鍵を生成し、サーバーに登録。フレンド追加時に公開鍵を交換。
2.  **チケット作成**: ユーザーはチケットを作成し、参加メンバーを指定。
3.  **メッセージ作成**:
    -   ユーザーがメッセージを入力。
    -   AI審査システムがメッセージ内容を解析し、品質を評価。
    -   AI審査を通過したメッセージはShikakuMLSに渡され、受信者（またはグループ）の公開鍵を用いて暗号化。
4.  **メッセージPush**:
    -   暗号化されたメッセージ (`EncryptedMessage`) と関連メタデータがCloud Functions経由でメッセージキューに登録。
    -   受信者へ通知（リマインド）。
5.  **受信者Pull**:
    -   受信者がアプリを起動または更新すると、新しい暗号文メッセージとメタデータをプル。
    -   SwiftDataに暗号文と検索用メタデータを保存。
6.  **表示時復号**:
    -   メッセージは表示時に端末内で復号され、平文は一時的にメモリ上で処理され永続化されない。
7.  **Push権ライセンスの回転・記録**:
    -   メッセージPush後、Push権の順番が更新され、その履歴が記録される。
8.  **タイムスタンプ証跡生成・保存**:
    -   サーバー側でメッセージメタデータにRFC3161準拠のタイムスタンプとEd25519署名を付与し、非否認性を保証して永続保存する。

---

## 8. 今後の拡張

-   **多言語対応**: グローバル展開に向けたUI/UXおよびAI審査の多言語化。
-   **組織内ダッシュボード**: 企業・組織向けに、コミュニケーションの品質、チケット消化状況、AI審査傾向などを可視化する管理ダッシュボード。
-   **文章力向上コンテンツ**: 『理科系の作文技術』などの論理的思考・文章作成術をベースとした学習コンテンツを提供し、AI審査と連携してユーザーの表現能力向上を支援。
-   **ShikakuMLSの高度機能**: MLSにおける群脱退時の鍵更新自動検証や、Post-Compromise Securityのより厳密な保証機能などを実装。
