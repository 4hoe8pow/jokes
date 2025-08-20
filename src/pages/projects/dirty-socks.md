---
layout: ../../layouts/ProjectLayout.astro
title: 🧦dirty socks
description: テキスト主体靴下ゲーム
tags: ["Swift"]
timestamp: 2025-08-20T10:03:00+00:00
featured: true
filename: dirty-socks
---

# DIRTY SOCKS 仕様まとめ（SwiftUI / Composable Architecture）

## アプリ概要
- **アプリ名**: DIRTY SOCKS  
  iOS / SwiftUI / SwiftData ベース
- **開発方針**: SwiftUI単独、UIKit/Unity/UE5 不使用  
  古典的テキストRPG風ログゲーム。やりこみ要素重視。

## アーキテクチャ
- **Composable Architecture (MVVM禁止)**  
  - View は軽量
  - `@StateObject` や `@ObservedObject` で GameState / PlayerState / EnemyState を購読
  - Action は State 側で処理、テスト容易

## セッション・ゲーム進行
| 項目             | 内容                                                                                                | 補足                                                                                                   |
| -------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **セッション**      | 1プレイ1セッション                                                                                        | 試合終了 or タスクキルで全ログ・GameState を初期化し HomeView に戻る                                                          |
| **画面構成**       | - HomeView: 現場選択、パーティ編成、難易度設定<br>- EscapeInGameView: プレイ中の操作・ログ表示                                 | HomeView → EscapeInGameView に `NavigationStack` や `fullScreenCover` で遷移。選択情報を GameState に反映 |
| **画面遷移**       | HomeView → EscapeInGameView                                                                       | セッション開始時に HomeView でパーティ・現場を選択しゲーム開始。終了後は HomeView に戻る                                           |
| **ログ管理**       | LogEntry(Model) + SwiftData                                                                       | Timestamp・Message保持。In-Memoryでセッション完結型。バックグラウンド復帰時も保持可能                                            |
| **ログ表示領域**     | ScrollView + LazyVStack、縦62%固定                                                                    | プレイ中ログはスクロール追従。操作パネルは恒久表示。ログ蓄積はボタン操作中も継続                                                            |
| **タイマー**       | - ログ追加タイマー（4秒間隔）<br>- エンカウンタータイマー（6秒間隔）                                                           | Combine または GameState 内で管理。View破棄時に自動停止可能                                                              |
| **ゲーム状態**      | GameEvent: idle / enemyEncounter                                                                  | View 依存なしで State で一元管理。操作待ち・敵出現状態を明確化                                                                  |
| **操作アクション**    | PlayerAction: attack / evade / escape                                                             | ボタンは恒久表示。操作待ちはハイライト表示で視覚的に緊張感を演出                                                                    |
| **敵構造**        | 無数のキーパー(洗濯機、物干し竿、引き出し)                                                                            | ログで戦闘経過表現、敵画像は靴下で統一。GameState で生成・退却・衝突管理                                                            |
| **プレイヤーパーティ**  | 4キャラクター選択してパーティ行動                                                                                 | 各キャラクター固有スキルあり。HomeViewで選択し GameState に反映                                                             |
| **UI構造**       | - 背景: LinearGradient + GlassEffect<br>- ログ領域: ScrollView + LazyVStack<br>- 操作パネル: VStack + HStack | ボタン押下中もログ蓄積。操作待ちは視覚的ハイライト。ログスクロール追従                                                                 |
| **ローカライズ**     | String Catalog (FString)                                                                          | `String.LocalizationValue`で管理。GUIで多言語追加可能。NSLocalizedString非依存                                      |
| **データ永続化**     | SwiftData / In-Memory                                                                             | 1プレイ完結型。セッション終了で全削除。バックグラウンドでの一時保持は設計次第                                                             |
| **外部API / DB** | Firebase                                                                                          | View 直結で購読可能。リアルタイム反映。必要に応じ GameState に橋渡し                                                             |
| **ログ寿命**       | プレイ中のみ                                                                                            | バックグラウンド復帰時は保持。試合終了で全削除                                                                             |
| **課金 / マネタイズ** | なし（MVP段階）                                                                                         | 将来的に広告や課金要素追加可                                                                                      |
| **操作演出**       | リアルタイム制、操作待ちはボタンハイライト                                                                             | ログやタイマー表示と組み合わせてプレイヤーに緊張感を演出                                                                        |
| **テスト容易性**     | Composable Architecture + State分離                                                                 | Unitテストは GameState / PlayerState / EnemyState に集中。View は軽量。UIテストは HomeView → EscapeInGameView の流れで可能 |

## 推奨フォルダ構成（画面遷移版）

```sh
DIRTY SOCKS/
├─ DIRTY SOCKS/
│ ├─ App/
│ │ └─ DIRTY_SOCKSApp.swift ← Appエントリポイント、HomeViewを最初に表示
│ │
│ ├─ Views/
│ │ ├─ HomeView.swift ← 現場選択・パーティ編成画面
│ │ ├─ EscapeInGameView.swift ← 旧 ContentView、プレイ画面
│ │ ├─ LogView.swift ← ログ表示
│ │ ├─ ControlPanelView.swift ← 操作パネル
│ │ └─ EnemyView.swift ← 敵表示や出現アニメーション
│ │
│ ├─ State/
│ │ ├─ GameState.swift ← ゲーム全体状態
│ │ ├─ PlayerState.swift ← パーティ・操作中のキャラクター状態
│ │ └─ EnemyState.swift ← 敵生成・退却・ログ管理
│ │
│ ├─ Models/
│ │ └─ LogEntry.swift ← ログエントリ / SwiftDataモデル
│ │
│ ├─ Resources/
│ │ ├─ Assets.xcassets ← 靴下や背景など画像
│ │ ├─ DIRTY_SOCKS.xcstrings ← String Catalog
│ │ └─ Sounds/ ← 効果音・BGM
│ │
│ └─ Utilities/
│ ├─ Extensions.swift ← View / String / Timer など拡張
│ └─ Helpers.swift ← 汎用関数
│
├─ DIRTY SOCKSTests/
│ └─ GameStateTests.swift ← 状態管理の単体テスト
│
└─ DIRTY SOCKSUITests/
└─ DIRTY_SOCKSUITests.swift ← ボタン操作やログ表示などUIテスト
```

### 設計ポイント
- **State分離**: GameState / PlayerState / EnemyState は View から独立
- **Composable View**: EscapeInGameView は LogView / ControlPanelView / EnemyView を組み合わせ
- **画面遷移**: HomeView でパーティや現場を選択 → EscapeInGameView に渡す
- **テスト容易性**: View は軽量、State 集中で単体テストとUIテスト分離
- **リアルタイム演出**: タイマー + ログ + ボタンハイライトで緊張感を演出
