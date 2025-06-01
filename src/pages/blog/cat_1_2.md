---
layout: ../../layouts/BlogLayout.astro
title: (1,2節) 圏の構造と公理
description: 対象と射、およびそれらの振る舞いを規定する公理体系
tags: ["圏論"]
time: 15
featured: true
timestamp: 2025-07-14T14:10:03+00:00
filename: cat_1_2
---

## 圏（Category）の定義

圏 $\mathcal{C}$ は、次のデータからなる：

- **対象集合** $\mathrm{Ob}(\mathcal{C})$
- **射集合** $\mathrm{Hom}_{\mathcal{C}}(A, B)$（$A, B \in \mathrm{Ob}(\mathcal{C})$）
- **合成**：$g \circ f$（$f : A \to B$, $g : B \to C$）
- **恒等射**：$\mathrm{id}_A \in \mathrm{Hom}_{\mathcal{C}}(A, A)$

```haskell
class Category cat where
  id  :: cat a a
  (.) :: cat b c -> cat a b -> cat a c
```

---


### 公理体系

- **結合律**：
  $$
  h \circ (g \circ f) = (h \circ g) \circ f
  $$
- **恒等律**：
  $$
  \mathrm{id}_B \circ f = f = f \circ \mathrm{id}_A
  $$

```haskell
-- h . (g . f) == (h . g) . f
-- id . f == f == f . id
```

---

g :: String -> [Char]
g = reverse
h :: Int -> [Char]
h = g . f
f : X → Y に対して：
id_X     id_Y

## 図式記法と合成の視覚化

```plaintext
A ──f──▶ B ──g──▶ C

↓ g∘f

A ─────▶ C
```

---


## 例

- **集合の圏 $\mathbf{Set}$**  対象：集合、射：写像

- **群の圏 $\mathbf{Grp}$**  対象：群、射：群準同型

- **型と関数の圏（Haskell）**

```haskell
instance Category (->) where
  id x = x
  (g . f) x = g (f x)
```

---


---

このように、圏論は「対象・射・合成・恒等射・公理」の最小限の枠組みで、あらゆる数学的構造を統一的に記述する。
