---
layout: ../../layouts/BlogLayout.astro
title: 第1章 圏の構造と公理
description: 対象と射、およびそれらの振る舞いを規定する公理体系
tags: ["圏論"]
time: 15
featured: true
timestamp: 2025-07-14T14:10:03+00:00
filename: cat_1_2
---

# 第1章 圏の構造と公理

## 1.1 圏（Category）の定義

**圏**（category）とは、対象と射（morphism）の集まりであり、以下の公理に従って構造化されたものをいう。

### 定義（圏）

圏 $\mathcal{C}$ とは、以下のデータから構成される：

- **対象のクラス** $\mathrm{Ob}(\mathcal{C})$（object）
- **射のクラス** $\mathrm{Hom}_{\mathcal{C}}(X, Y)$（morphisms）各対象 $X, Y \in \mathrm{Ob}(\mathcal{C})$ に対して
- 各対象 $X$ に対して恒等射 $\mathrm{id}_X : X \to X$
- 各射 $f : X \to Y$, $g : Y \to Z$ に対する**合成**写像：
  $$
  g \circ f : X \xrightarrow{f} Y \xrightarrow{g} Z \quad \Rightarrow \quad X \xrightarrow{g \circ f} Z
  $$

これらは以下の**二つの公理**を満たす：

---

## 1.2 圏の公理

### 【1】結合律（Associativity）

任意の射 $f : X \to Y$, $g : Y \to Z$, $h : Z \to W$ に対して、
$$
h \circ (g \circ f) = (h \circ g) \circ f
$$

### 【2】単位律（Identity laws）

任意の射 $f : X \to Y$ に対して、
$$
\mathrm{id}_Y \circ f = f = f \circ \mathrm{id}_X
$$

---

## 1.3 図式による視覚化

合成と恒等射の挙動は**可換図式（commutative diagram）**で表される。

#### 合成の図式例：

```plaintext
X ──f──▶ Y ──g──▶ Z

↓ g∘f

X ─────▶ Z
```

#### 恒等射の図式：

```plaintext
X ──id_X──▶ X

f : X → Y に対して：

X ──f──▶ Y
│        ▲
id_X     id_Y
│        │
X        Y
```

---

## 1.4 圏の例

### 例1：集合の圏 $\mathbf{Set}$

- **対象**：集合
- **射**：写像（関数）$f : X \to Y$
- **恒等射**：恒等関数 $\mathrm{id}_X(x) = x$
- **合成**：関数の合成

### 例2：群の圏 $\mathbf{Grp}$

群の圏 $\mathbf{Grp}$ は、圏論的公理体系に則り、次のように定義される：

- **対象（object）**：群 $G$。すなわち、集合 $G$ と二項演算 $\cdot : G \times G \to G$、単位元 $e \in G$、逆元 $g^{-1}$ の存在により、群公理を満たすもの。
- **射（morphism）**：群準同型 $f : G \to H$。すなわち、任意の $g_1, g_2 \in G$ に対し $f(g_1 g_2) = f(g_1) f(g_2)$ を満たす写像。
- **恒等射**：各群 $G$ に対し、恒等写像 $\mathrm{id}_G : G \to G$、$\mathrm{id}_G(g) = g$。
- **合成**：群準同型 $f : G \to H$, $g : H \to K$ に対し、合成 $g \circ f : G \to K$ を $g(f(g'))$（$g' \in G$）で定める。
- **公理**：合成と恒等射は、圏の結合律・単位律を満たす。

このようにして、$\mathbf{Grp}$ は圏の公理体系の下で厳密に構成される。

---

## まとめ

| 構成要素       | 意味                                     |
| -------------- | ---------------------------------------- |
| 対象（object） | 構造の単位。ノードとして図式化される     |
| 射（morphism） | 対象間の構造保持写像。関数のようなもの   |
| 恒等射         | 各対象に自然に備わる自明な射             |
| 合成           | 射同士の連結。演算としての構成力学を担う |
| 公理           | 構造を整合的にするための制約条件         |

---

> 圏論は、「構造そのもの」ではなく「構造の間の変換」に注目する数学です。これにより、個別理論（集合・群・位相空間など）を統一的に眺めることが可能になります。

---

## 圏の構造的公理体系

圏 $\mathcal{C}$ は、以下のデータと公理により定義される：

- **対象（object）**：$\mathrm{Ob}(\mathcal{C})$ は圏 $\mathcal{C}$ の対象全体の類。
- **射（morphism）**：任意の $X, Y \in \mathrm{Ob}(\mathcal{C})$ に対し、$X$ から $Y$ への射の集合 $\mathrm{Hom}_{\mathcal{C}}(X, Y)$ が与えられる。各射 $f \in \mathrm{Hom}_{\mathcal{C}}(X, Y)$ は $f : X \to Y$ と表記される。
- **合成写像**：任意の $X, Y, Z \in \mathrm{Ob}(\mathcal{C})$ に対し、
  $$
  \circ : \mathrm{Hom}_{\mathcal{C}}(Y, Z) \times \mathrm{Hom}_{\mathcal{C}}(X, Y) \to \mathrm{Hom}_{\mathcal{C}}(X, Z)
  $$
  が定義され、$g \circ f$ は $f : X \to Y$, $g : Y \to Z$ に対して $X \xrightarrow{f} Y \xrightarrow{g} Z$ の合成 $X \xrightarrow{g \circ f} Z$ を与える。
- **恒等射**：各対象 $X \in \mathrm{Ob}(\mathcal{C})$ に対し、恒等射 $\mathrm{id}_X \in \mathrm{Hom}_{\mathcal{C}}(X, X)$ が存在する。

これらは、次の二つの公理を満たす：

### (A) 結合律（Associativity）

任意の $f : X \to Y$, $g : Y \to Z$, $h : Z \to W$ に対して、
$$
h \circ (g \circ f) = (h \circ g) \circ f
$$
が成立する。

### (B) 単位律（Identity Laws）

任意の $f : X \to Y$ に対して、
$$
\mathrm{id}_Y \circ f = f = f \circ \mathrm{id}_X
$$
が成立する。

---

圏論は、個々の構造そのものではなく、構造間の射およびその合成の体系的性質に主眼を置く。公理体系は、射の合成と恒等射の振る舞いを厳密に規定し、あらゆる数学的構造の統一的記述を可能とする。

---

## 本質

圏論の本質は、個々の対象や射の具体的な内容ではなく、それらの間に張り巡らされた射のネットワークと、その合成・恒等性の厳密な制御にある。圏の公理体系は、あらゆる数学的構造を「射の合成と恒等射の振る舞い」という一点に還元し、個別理論の壁を超えた普遍的な枠組みを与える。圏論的視点は、構造の本質を「変換の体系」として捉え直すことで、数学の統一的理解と新たな展開を可能にする。
