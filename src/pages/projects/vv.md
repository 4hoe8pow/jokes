---
layout: ../../layouts/ProjectLayout.astro
title: 🦀 vv
description: TUIゲーミングポータル
tags: ["TUI"]
timestamp: 2025-09-26T14:41:00+00:00
featured: true
filename: vv
---

## 1. 概要
vvポータルは、ターミナル上で動作する**ゲーム・コミュニティ統合型ポータル**です。  
ユーザーはCohort（小規模サークル）に所属し、チャットやゲームを通じて交流できます。  
ゲームは公式リポジトリ経由で追加され、クライアントは自動更新されます。

---

## 2. システム構成

### 2.1 クライアント
- ターミナルUI（[ratatui](https://ratatui.rs/)）で軽量動作
- **メニュー構成**
  - **Home**：プロフィール + 所属Cohort一覧
  - **Games**：ゲーム選択・ASCIIサムネ + リーダーボード
  - **Settings**：認証状態確認、ログアウト
  - **Exit**：アプリ終了
- **Cohort内操作**
  - メンバー一覧、チャンネル閲覧
  - チャンネル内での発言（不可変ログ）
  - フォロー/フォロワー確認
  - ロール管理（owner、category1〜4、category0）
- **ゲーム管理**
  - ゲーム選択・起動
  - 更新チェック → `cargo update` で更新
  - ゲーム仕様はクライアント非依存

---

### 2.2 バックエンド（Firebase）
- **認証**
  - Firebase Auth（email/password, OAuth 等）
  - CLI・TUI連携用トークン管理
- **データ管理（Firestore）**
  - `users`：uuid, displayName, createdAt, Lv, followers/following
  - `cohorts`：ownerId, members(map), roles, チャンネル管理
  - `channels`：channelId, cohortId, messages（不可変）
  - `games`：GitHubリポジトリ連携情報
  - `leaderboards`：gameId, scores, updatedAt
- **ルール**
  - displayName/uuidは変更不可
  - メッセージの編集・削除不可
  - Cohort内ロール人数制限と権限付与ルール

---

### 2.3 ゲーム配布（GitHub）
- 公開OSS方式
- プルリクエストで追加、レビュー・承認後に main ブランチへマージ
- ratatuiクライアントは最新ブランチを pull して自動更新

---

## 3. ユーザー・アカウント設計

| 項目 | 内容 |
|------|------|
| uuid | 内部ID、非表示、永久固定 |
| displayName | 英数字 + `_ + #` のみ、3〜32文字、永久固定、唯一 |
| createdAt | アカウント作成日 |
| Lv | ゲームポイント合計、数値保持 |

- displayNameは作成時に段階的に警告（「間違えても愛してください」）
- アカウント削除可能だが、displayNameの再利用は不可

---

## 4. Cohort設計
- **構造**
  - 所属ユーザー最大100人
  - 各ユーザーにロールあり（owner、category0~4）
  - category1~4は人数制限（13, 8, 5, 3）、category0は無制限
- **チャンネル**
  - 無制限作成可能
  - メッセージは不可変ログ（編集/削除不可）
- **owner移譲**
  - ownerは他人に権限移譲でのみ退任可能

---

## 5. Games画面設計（TUI）
- **1スクロール = 1ゲームタイトル表示**
- **左ペイン**：ゲームASCIIサムネ
- **右ペイン**：リーダーボード（Top5表示、displayName+Lv）
- **操作**
  - ↑/↓：ゲーム選択
  - Enter：ゲーム起動
  - q：戻る

### レイアウト例

```txt
┌──────────────────────────────┐
│ Game: Foo                    │
├──────────────────────────────┤
│ ┌───────┐ Leaderboard        │
│ │ ##### │ 1. alice Lv42      │
│ │ # #   │ 2. bob Lv37        │
│ │ ##### │ 3. eve Lv35        │
│ └───────┘ 4. charlie Lv30    │
│           5. david Lv28      │
│ [↑/↓] Scroll [Enter] Play [q]│
└──────────────────────────────┘
```

## 6. 特徴・メリット
- **レトロ感と操作性**
  - TUI + ASCIIアート + 1タイトルスクロールで直感的
- **セキュリティ・透明性**
  - 名前/発言の不変性
  - Cohort権限管理・owner移譲
- **OSS開発文化**
  - GitHubプルリクでゲーム追加
  - 自動更新でクライアント軽量化
- **ゲームとコミュニティの統合**
  - Lv・リーダーボードで活動可視化
  - チャンネル発言でCohort内交流

---

## 7. 運営フロー
1. ゲーム追加：開発者 → プルリク → 承認 → mainブランチ
2. クライアント更新：ratatuiが `cargo update` で取得
3. ユーザー活動：ゲームプレイでLv加算、チャンネル発言
4. Cohort管理：ロール付与、owner移譲、チャンネル管理
