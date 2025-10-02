---
layout: ../../layouts/BlogLayout.astro
title: クリーンアーキテクチャへの理解
description: レストランで例える
tags: ["クリーンアーキテクチャ"]
time: 2
featured: ture
timestamp: 2025-10-02T09:46:03+00:00
filename: clean
---

| #  | Layer                  | Component               | Abstract / Concrete          | Restaurant Analogy                          | Explanation |
|----|-----------------------|------------------------|-----------------------------|--------------------------------------------|-------------|
| 1  | Domain (Enterprise Rules) | Entity                 | Concrete                    | メニュー（例：カレーライス #123）       | ビジネス上の核となるオブジェクト。IDで同一性を識別し、属性は変更可能。顧客注文や業務ロジックの中心となる。 |
| 2  |                        | Value Object           | Concrete                    | 分量、トッピング、価格                     | 不変で値の等価性によって同一性を判断。料理の構成要素や属性として扱われる。 |
| 3  |                        | Domain Service         | Concrete                    | 料理長による裁定（在庫や予約状況に基づくコース構成） | 単独Entityに属さないドメインロジックを担当。複数Entity間のルールや判断を実装する。 |
| 4  |                        | Repository Interface   | Abstract                    | 倉庫への発注窓口                           | コックが食材を取得するための抽象窓口。実際のデータ取得手段には依存しない。 |
| 5  | Application (Use Case) | Request DTO            | Concrete (data structure)   | 注文伝票                                   | 外部注文情報をアプリケーション層で処理可能な形式に変換したデータ構造。 |
| 6  |                        | Input Port             | Abstract                    | 注文伝票の置き場                            | コックへの業務開始命令。Use Caseの開始ポイント。 |
| 7  |                        | Interactor             | Concrete (Input Port Impl.) | コック                                     | Input Portを実装し、伝票に基づいて調理。ビジネスロジックを実行する主体。 |
| 8  |                        | Response DTO           | Concrete (data structure)   | 調理済みの料理                             | 料理完成後の成果物を表現。内部手順は隠蔽され、外部提供情報のみを含む。 |
| 9  |                        | Output Port            | Abstract                    | デシャップ（配膳カウンター）             | 完成した料理とともに伝票をホールへ返却。Interactorと外界の橋渡しを行う。 |
| 10 | Adapter (UI / Interface) | Controller             | Concrete                    | ホールスタッフ（注文受付担当）            | お客様からの注文を受け、Request DTOに変換してInput Portに渡す。 |
| 11 |                        | Presenter              | Concrete (Output Port Impl.) | ホールスタッフ（料理提供担当）           | Output Portを実装し、完成料理をお客様に提供可能な形に整える。盛り付けや装飾も含む。 |
| 12 | Infrastructure        | View                   | Concrete                    | お客様のテーブル                           | 実際に料理が提供される場所。フレームワーク依存の表示処理を含む。 |
| 13 |                        | DAO                    | Concrete（Repository Impl.） | 仕入担当                         | 食材の仕入れや在庫管理を担当。Repository Interfaceを具体的に実装し、データベースや外部サービスと連携。 |

## 1. Domain Layer（組織ルール層）

### Entity
- **定義**：同一性（Identity）を持つ本質的対象。属性の変更可能性は本質的な問題ではなく、IDに基づく同一性の保持が核心。
- **ポイント**：Entityは「変化する属性の中で不変の本質を保持する存在」と考えられる。メニュー番号や顧客IDはその象徴。

### Value Object
- **定義**：属性の集合としてのみ意味を持つオブジェクト。不変で等価性によって同一性を判定。
- **ポイント**：「値は属性の集合体であり、自己の同一性はその内容に内在する」。IDは不要。価格や分量、座標など、振る舞いよりも状態の整合性が重要な要素に適用。

### Domain Service
- **定義**：単一Entityに属さない、ビジネスロジックを表現するオブジェクト。
- **ポイント**：「Entity間の関係や協調を扱う抽象的存在」として、ドメインのルールを形而上に実装。

### Repository Interface
- **定義**：永続化を抽象化したインターフェース。データ取得手段に依存せず、Entity操作の契約を提供。
- **ポイント**：「ドメインはデータアクセスの存在を知らない」。永続化の具体実装を意図的に隠蔽。

---

## 2. Application Layer（ユースケース層）

### Request DTO
- **役割**：外部からの要求をアプリケーション層で扱えるデータ構造に変換。

### Input Port / Interactor
- **Input Port（抽象）**：ユースケース実行の契約。Interactorに渡す「命令の型」。
- **Interactor（具体）**：ユースケースの実装。ビジネスルールを呼び出し、ドメインに指示を与える。

### Response DTO / Output Port
- **Response DTO**：処理結果を外部に返すための構造化された情報。
- **Output Port**：Interactorが結果を渡す契約。具体実装（Presenter）が整形して外界に提供。

---

## 3. Adapter Layer

### Controller
- **定義**：外部からの入力を受け取り、Request DTOに変換し、Input Portに渡す。

### Presenter / View
- **Presenter**：Output Portの具体実装。内部結果を表示可能な形に整形。
- **View**：ユーザーとの接点。フレームワーク依存の表示や操作を実行。

---

## 4. Infrastructure Layer（永続化・外部依存）

### DAO / Repository Impl.
- **定義**：Repository Interfaceを具体的に実装。データベースや外部サービスとのやり取りを担当。
- **ポイント**：「ドメインに汚染されることなく、永続化や外部連携の具体を担う」。

---

## 5. 総括的理解
- **ポートとアダプタ**：境界と契約を明示し、ドメインの純粋性を保つための手段。
- **Entity vs Value Object**：変化の中の不変性（Identity）と、値としての不変性（Value）を明確に分離。
- **ドメイン中心設計の核心**：ビジネスルールを中心に据え、外界や永続化への依存を極力隠蔽する構造。
- **抽象と具象の階層**：抽象は契約、具象は実行。境界を超える情報の変換がユースケース層の役割。

> **キーワード**：境界（Boundary）、契約（Contract）、純粋性（Purity）、抽象化（Abstraction）、実行（Execution）、分離（Separation of Concerns）
