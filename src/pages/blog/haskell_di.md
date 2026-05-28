---
layout: ../../layouts/BlogLayout.astro
title: Haskellの依存性注入、現実的な落としどころ
description: 型クラスDIが破綻する理由と、ReaderT＋明示的レコードによる「崩れない」設計の実践
tags: ["Haskell", "CleanArchitecture", "DI", "ReaderT", "関数型設計"]
time: 30
featured: false
timestamp: 2026-05-28T12:50:03+00:00
filename: haskell_di
---

## 1. そもそも、なぜOO流の「型クラスDI」は破綻するのか

オブジェクト指向の感覚でHaskellのクリーンアーキテクチャを設計しようとすると、多くの人が同じ道をたどります。まずインターフェースを型クラスで抽象化し、広域なmtlパターンで依存を解決しようとする。しかしそのアプローチは、規模が育つにつれて以下の三つの落とし穴に順番に落ちていきます。

**手動DIの肥大化。** DIコンテナのない世界では、エントリーポイントでの依存オブジェクトの組み立てが雪だるま式に膨れ上がります。最初は数行だったコードが、いつの間にか触れたくない巨大な儀式になっている――経験者には心当たりがあるはずです。

**型クラスのボイラープレート地獄。** モックを差し替えたいだけなのに、`TestM` 型と大量のインスタンス定義を量産することになります。テストを書くたびに型パズルと格闘することになり、開発の勢いが失われていきます。

**制約の追跡不能化。** `MonadReader` や `MonadError` で過剰に抽象化すると、コンパイルエラーのメッセージが意味不明になり、「この関数、実際にどこで動いているんだっけ？」という問いへの答えがコードから消えていきます。

ひとつ誤解しないでほしいのは、**型クラスそのものが悪いわけではない**ということです。問題は「アプリケーション全体にまたがる、暗黙的で重厚な設計」にあります。単一の関心事に絞った局所的なCapabilityとして型クラスを使うアプローチは、今でも有効です。ただ、認知負荷を下げ、最も破綻しにくい設計を選ぶという観点から、この記事では**明示的なレコード**を中心に据えて解説します。

---

## 2. 結論：目指すべき「現実的な落としどころ」

関数型における依存性の逆転の本質は、重厚なDIの仕組みを導入することではありません。**関数合成を整理すること**です。

実務でスケールする設計は、次の三原則に集約されます。

1. **環境（Env）は「1つの構造化レコード」にする** ―― ただし、内部は関心事ごとに分割して神オブジェクト化を防ぐ
2. **モナドスタックは固定する** ―― `ReaderT` + `ExceptT` の具体的なスタックを決め打ちにする
3. **エラーは抽象ヘルパーに頼らず、明示的にマッピングする** ―― 「どこで何に変換しているか」を読んでわかるコードにする

---

## 3. アンチパターン：「一見きれいなコード」はなぜ崩壊するのか

具体例として、定食屋の注文ユースケースを考えます。「注文を受け取り（`OrderRequest`）、食材の在庫を確認・消費し、伝票を発行して、売上イベントを記録する」という、ありがちな業務フローです。

### ❌ 過度な抽象化の例

```haskell
-- ❌ ダメな例：型クラスによる過度な抽象化と、将来への備えがないユースケース
executeOrderAntiPattern :: (MonadStock m, MonadBilling m, MonadEvent m, MonadError OrderError m) 
                       => OrderRequest -> m OrderResponse
executeOrderAntiPattern req = do
  maybeStock <- checkStock (req.menuId)  -- どこで実装されているか追えない暗黙的なDI
  case maybeStock of
    Nothing -> throwError InsideDomainError
    Just stock -> do
      -- ⚠️ 危険：この時点では「純粋な計算」として設計されている
      let (updatedStock, event) = consumeStock stock 
      
      _ <- saveStock updatedStock
      _ <- emitEvent event
      pure $ toResponse updatedStock
```

このコードは、読んだ瞬間はすっきりして見えます。しかし仕様変更という現実に当てると、一瞬で崩れます。

### 🚨 現場で実際に起こる崩壊のシナリオ

「軽減税率の導入に伴い、`consumeStock` が失敗するケース（上限チェック等）を扱う必要が出てきた」という、ごく普通の仕様変更を想像してください。

ドメインロジックの戻り値の型が `(Stock, Event)` から `Either DomainError (Stock, Event)` に変わった瞬間、**この関数を呼び出している全てのユースケース――数十から数百ファイル――の型とdo構文の結合が連鎖的に崩壊します。** 修正範囲が雪だるまのように膨れ、「ちょっとした業務ルールの追加」が全体リファクタリングに化けるのです。

---

## 4. ⭕ 改善例：「崩れない」設計の考え方

では、どうすれば壊れにくくなるのか。答えは大きく三つです。過度な抽象化をやめ、環境を構造化されたレコードに集約し、整形レイヤーを設けてユースケースをノイズから守る。

```haskell
{-# LANGUAGE OverloadedRecordDot #-}

module UseCase.Order where

import Control.Monad.Reader (ask, lift)
import Control.Monad.Except (liftEither)
import Data.Bifunctor (first)

-- ====================================================================
-- 1. 環境（Env）の定義：1つにまとめるが、内部は関心事で構造化する
-- ====================================================================
-- 「1つのEnvで引き回す利便性」と「関心事ごとの分割」を両立するために、
-- RepoやService単位で入れ子にする。
data StockRepo m = StockRepo { check :: MenuId -> m (Either DBError Stock)
                             , save  :: Stock -> m (Either DBError ()) }
data EventBus  m = EventBus  { emit  :: Event -> m () }

data TeishokuEnv m = TeishokuEnv
  { envStockRepo :: StockRepo m
  , envEventBus  :: EventBus m
  }

-- モナドスタックを「具体的に」固定することが、追跡可能性の鍵
type TeishokuAppM m = ReaderT (TeishokuEnv m) (ExceptT DomainError m)

-- ====================================================================
-- 2. 整形レイヤー：「型変換の泥臭さ」をここに閉じ込める
-- ====================================================================
-- このレイヤーの責務は明確に三つ。
--   (a) IO や Either をアプリケーションモナド(AppM) へ引き上げる
--   (b) インフラのエラー(DBError) をドメインのエラー(DomainError) へ翻訳する
--   (c) lift の階層といった「モナドの都合」をユースケースから隠す
--
-- ここを整形レイヤーとして切り出すことで、ユースケース本体は
-- ビジネスロジックの「ストーリー」だけを語れるようになる。

loadStockM :: Monad m => TeishokuEnv m -> MenuId -> TeishokuAppM m Stock
loadStockM env mid = do
  -- lift (ExceptT層を突破) -> lift (ReaderT層を突破) の順でモナドを引き上げる
  res <- lift (lift (env.envStockRepo.check mid))
  liftEither $ first StockDomainError res  -- インフラエラー → ドメインエラーへ翻訳

saveStockM :: Monad m => TeishokuEnv m -> Stock -> TeishokuAppM m ()
saveStockM env stock = do
  res <- lift (lift (env.envStockRepo.save stock))
  liftEither $ first StockDomainError res

-- ====================================================================
-- 3. ユースケース：ビジネスのストーリーだけが残る
-- ====================================================================
executeOrder :: Monad m => OrderRequest -> TeishokuAppM m OrderResponse
executeOrder req = do
  env <- ask  -- 依存関係を環境から取り出す

  -- 入力のバリデーション
  menuId <- liftEither $ first InvalidInputError (mkMenuId req.rawMenuId)

  -- データのロード（整形レイヤーがノイズを吸収してくれている）
  stock <- loadStockM env menuId

  -- ★ 設計の核心：失敗する可能性が1%でもある操作は、最初からEitherで受ける
  -- こうしておけば、将来ドメイン側にエラーケースが増えても
  -- このユースケースの型は1文字も変わらない。
  (updatedStock, event) <- liftEither $ consumeStock stock

  -- 保存と副作用
  saveStockM env updatedStock
  lift (lift (env.envEventBus.emit event))  -- lift の階層を意識して明示

  pure $ toOrderResponse updatedStock
```

---

## 5. この設計が守ってくれるもの

### ① 「1つのEnv」と「関心事の分割」の両立

`env.envStockRepo.check` のように `OverloadedRecordDot` を使うことで、Envは1つのまま引き回しつつ、内部の肥大化を防げます。50個の関数が一枚岩のレコードに押し込まれる「神オブジェクト化」を、構造の入れ子だけで綺麗に回避できます。

### ② 「Either先回り」の防衛思想

「今は正常系だけでいい」は短期的な最適化であり、長期的には罠です。実務では、ほぼ確実に後からチェックロジックが追加されます。*失敗する可能性が1%でもあるドメイン操作は、最初からEitherで設計する*――この一貫した思想が、ドメイン層の変更がユースケース層に波及する連鎖崩壊を防ぎます。

### ③ 整形レイヤーが作る「境界線」

`loadStockM` のようなヘルパーは、単なるショートカットではありません。「インフラの都合（`DBError`）」を「ドメインの言葉（`DomainError`）」に翻訳する防波堤です。ユースケースはその翻訳を知らなくてよく、ビジネスロジックの流れだけに集中できます。

### ④ 学習コストの局所化

`lift (lift (...))` のようなモナドスタックの操作は、Haskell初学者が最も混乱するポイントです。それを整形レイヤーに閉じ込め、「なぜこのliftが必要か」をそこにコメントで説明することで、チーム全体の理解コストを大幅に下げられます。難しい部分を隠すのではなく、*難しい部分を一箇所に集める*という発想です。

### ⑤ テストは「レコードにモックを詰めるだけ」

型クラスの新しいインスタンスも、専用のテスト用モナドも必要ありません。`TeishokuEnv` のフィールドにモック関数を手で詰めて渡すだけ。型パズルなしに、安全で軽量なテストが書けます。

---

## まとめ：「効果的な具象の境界線」を引く

この設計の本質は、抽象化を否定することではありません。**どこまでを具象にし、どこから先を抽象にするか、その境界線を意図的に引くこと**です。

- 型クラスは「アプリケーション全体を覆う暗黙のDI」としてではなく、「局所的なCapability」として限定的に使う
- Envは「1つにまとめて引き回す」が、内部は「関心事ごとに構造化」して膨張を防ぐ
- ドメインの結合は「最初からEither」で設計し、将来の変更コストを先払いしておく
