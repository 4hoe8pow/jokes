---
layout: ../../layouts/BlogLayout.astro
title: (1,3節) 射の構造と公理
description: 恒等射・射の合成・結合律により、圏の内部構造が定義される
tags: ["圏論"]
time: 15
featured: true
timestamp: 2025-07-22T14:10:03+00:00
filename: cat_1_3
---

## 射の構造と公理

圏論における「射（morphism）」は、対象間の構造的な“つながり”を担う中心的存在である。射の振る舞いと構成的ルールは、圏の本質を規定する。

### 射の定義

圏 $\mathcal{C}$ の任意の対象 $A, B$ に対し、$A$ から $B$ への射 $f : A \to B$ が存在する。射の集合は $\mathrm{Hom}_{\mathcal{C}}(A, B)$ で表される。

### 恒等射

各対象 $A$ には、必ず恒等射 $\mathrm{id}_A : A \to A$ が存在し、
$$
\mathrm{id}_A \circ f = f, \quad g \circ \mathrm{id}_A = g
$$
が任意の射 $f : B \to A$, $g : A \to C$ について成り立つ。

### 合成

射 $f : A \to B$, $g : B \to C$ の合成 $g \circ f : A \to C$ が定義される。
合成は「流れ」として直感的に捉えられる：

```plaintext
A ──f──▶ B ──g──▶ C

↓ g∘f

A ─────▶ C
```

### 結合法則（Associativity）

合成は常に結合的である：
$$
h \circ (g \circ f) = (h \circ g) \circ f
$$
（$f : A \to B$, $g : B \to C$, $h : C \to D$）

```haskell
-- Haskellの関数合成は結合的
h . (g . f) == (h . g) . f
```

### 射の世界の直感

圏論では、対象そのものよりも「射のネットワーク」が主役となる。射の合成・恒等性・結合律こそが、圏の構造を支配する。

> 射の世界に生きるとは、あらゆる構造を「変換の連鎖」として捉え、流れ・合成・恒等性の規則に従って思考することである。
