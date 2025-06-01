---
layout: ../../layouts/BlogLayout.astro
title: (2,1節) 関手の定義
description: 圏から圏への構造保存写像であり、構造間の変換を記述する
tags: ["圏論"]
time: 15
featured: true
timestamp: 2025-07-22T14:20:03+00:00
filename: cat_2_1
---

## 2.1 関手の定義（Functor）

圏論における関手は、圏から圏への“構造保存的な写像”です。関手は、圏の間に写像を張ることで、構造そのものを保ったまま変換を行います。

### 定義（形式的）

関手 $F : \mathcal{C} \to \mathcal{D}$ とは：

- 各対象 $A \in \mathrm{Ob}(\mathcal{C})$ に対し、対象 $F(A) \in \mathrm{Ob}(\mathcal{D})$ を対応させる。
- 各射 $f : A \to B \in \mathrm{Hom}_{\mathcal{C}}(A, B)$ に対し、射 $F(f) : F(A) \to F(B) \in \mathrm{Hom}_{\mathcal{D}}(F(A), F(B))$ を対応させる。

この対応は、次の2条件を満たす：

1. **恒等射の保存**：
   $$
   F(\mathrm{id}_A) = \mathrm{id}_{F(A)}
   $$
2. **合成の保存**：
   $$
   F(g \circ f) = F(g) \circ F(f)
   $$
   （$f : A \to B$, $g : B \to C$）

### 直感と例

関手は「圏の構造を壊さずに、別の圏へ写すルール」です。

例えば、集合の圏から群の圏への関手は、各集合を群に、各写像を群準同型に対応させます。

```haskell
-- Haskell型クラスによる関手のイメージ
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

> 関手は「圏の世界をまるごと別の圏へ移す変換装置」。恒等射・合成の保存が“構造保存”の本質です。
