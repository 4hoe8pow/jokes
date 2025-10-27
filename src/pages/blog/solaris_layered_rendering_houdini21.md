---
layout: ../../layouts/BlogLayout.astro
title: Solaris Layered Rendering in Houdini 21.0
description: A comprehensive guide to leveraging Solaris in Houdini 21.0 for modern Layered Rendering pipelines—integrating USD contexts, Render Products, and COP2 compositing.
tags: ["Houdini", "Solaris", "USD", "Rendering", "Layered Rendering", "Compositing"]
time: 25
featured: true
timestamp: 2025-10-27T09:33:03+00:00
filename: solaris_layered_rendering_houdini21
---

## 概要

Houdini 21.0 における **Solaris (LOPs / USD)** を用いた **Layered Rendering（レイヤーベースレンダリング）** の実装設計を示す。
キャラクター・背景・エフェクトを独立したレイヤとしてレンダリングし、COP2 で非破壊的に合成可能なパイプラインを構築する。

---

## 1. 定義と位置付け

| 概念 | 定義 | Houdini Solaris 内での対応箇所 |
|------|------|--------------------------------|
| **AOV (Arbitrary Output Variable)** | 単一ショット内の光学要素（Diffuse, Specular, ZDepth 等）を別パス出力 | Render Vars / Render Settings |
| **Render Layer (Layered Rendering)** | シーン構造単位（キャラ・背景・FX 等）を独立してレンダーし、後段で合成 | LOP ネットワーク / USD Layer Stack |
| **Render Pass** | 上記両者の包括概念。粒度によってどちらにも該当 | ROP / Render Product レベル |

---

## 2. Houdini 21.0 での関連強化点

| 機能 | 内容 | Layered Rendering への利点 |
|------|------|---------------------------|
| **Configure Layer LOP のカラー管理メタデータ出力** | Stage に `colorManagementSystem` / `colorConfiguration` を直接埋め込み可能 | レイヤごとに異なる色空間設定を保持しつつ統合可能 |
| **Render-specific タブの自動再構築** | レンダラー変更や設定リロードを容易化 | レイヤ追加・再構成時の整合性維持 |
| **Path Expression Collection サポート** | `/World/Char/*` 等の USD Path 式で動的コレクション生成 | キャラ・背景・FX の一括指定が容易 |
| **Edit Layer Begin/End ノード** | 特定 USD レイヤへの限定編集を可能化 | 非破壊的オーバーライド編集 |
| **複数レンダーデリゲート同時アクティブ化** | Karma / Redshift 等を並行利用可 | レイヤごとに最適レンダラーを割り当て可能 |

---

## 3. シーン構成指針（USD レイヤ設計）

```text
/stage
 ├─ /char_layer        キャラクター専用サブレイヤ
 │    ├─ Geometry
 │    ├─ Materials
 │    └─ Lights (Key / Rim)
 ├─ /bg_layer          背景セット・環境光
 │    ├─ Geometry
 │    └─ Lights (Env / Indirect)
 ├─ /fx_layer          エフェクト系（Volume / Particle / Fluid）
 │    ├─ Geometry
 │    └─ Lights (Linked)
 ├─ /lighting_layer    共通ライト（必要に応じて独立）
 ├─ /renders
 │    ├─ RenderSettings_CHAR → `char.exr`
 │    ├─ RenderSettings_BG   → `bg.exr`
 │    └─ RenderSettings_FX   → `fx.exr`
 └─ /compose (COP2)
      ├─ FileCOP(char.exr)
      ├─ FileCOP(bg.exr)
      ├─ FileCOP(fx.exr)
      └─ Composite → Output
```

## 4. レンダー出力設計／AOV＋レイヤ分離  
### 4.1 レンダーパス構成原則  
Houdini 21.0のSolarisでは、Render Productノードで**レイヤ単位出力**と**AOV出力**を明確に分離できる。  
この段階での設計指針は以下の通り。

| 設計項目 | 意図／説明 |
|-----------|------------|
| **レイヤ分割** | キャラクター（CHAR）・背景（BG）・エフェクト（FX）など、論理上独立した要素単位でRender Productを作成。|
| **AOV分割** | 各レイヤ内で、beauty／diffuse／specular／sss／emission／shadow／zdepthなどを別AOVとしてRender Varで出力。|
| **命名規則** | `<shot>_<layer>_<aov>.<frame>.exr`。例：`S010_CHAR_diffuse.####.exr`|
| **カラースペース** | 21.0では`Configure Layer LOP`でOCIOメタデータを直接埋め込み可能。ACEScgを推奨。|
| **ファイルフォーマット** | すべてOpenEXR (multi-channel / tiled / mipmap enabled)。SolarisのEXR拡張で高品質・低メモリ運用可能。|

実際のRender Product構成例：

```text
/stage/renders
├─ RenderSettings_CHAR → EXR出力: /out/CHAR/<shot><aov>.####.exr
│ ├─ RenderVars: beauty, diffuse, specular, zdepth, shadow
├─ RenderSettings_BG → EXR出力: /out/BG/<shot><aov>.####.exr
│ ├─ RenderVars: beauty, indirect, albedo, zdepth
└─ RenderSettings_FX → EXR出力: /out/FX/<shot>_<aov>.####.exr
├─ RenderVars: beauty, emission, velocity, motionVector
```


### 4.2 AOV構築の実装手順（Solaris/Karma）
1. **Render Varノード作成**  
   - LOPネットワークで`Render Var`ノードを追加。  
   - `Source Type`に`LPE（Light Path Expression）`を選択。  
   - 例：`C<TD>+L`（Diffuse）、`C<TS>+L`（Specular）、`C<Z>`（ZDepth）等を定義。  
   - Houdini 21.0ではUIでLPE補完が実装され、構文エラー時にLive Renderで警告が出るよう改良されている。  

2. **Render Productノードへバインド**  
   - `Render Product`の`Render Var`パラメータで上記Varを参照。  
   - 各AOVを明示的にアタッチしておくと、EXRのチャンネル構造が整理され、COP2読み込みが容易。  

3. **カラー管理メタデータの埋込**  
   - `Configure Layer LOP`をRender設定群の直前に追加。  
   - `colorManagementSystem`：`OCIO`  
   - `colorConfiguration`：プロジェクトの`config.ocio`パス  
   - これにより、COP2内で自動的に色空間を解釈可能。  

4. **マルチレンダーターゲットの同時出力（21.0新仕様）**  
   - Houdini 21.0では1つのRender Settingsから複数Render Productを同時出力可能。  
   - 例：同じカメラで「CHAR_beauty」と「CHAR_zdepth」を同フレームで出力。  
   - 設定：Render Settings → “Products”リストに複数Render Productを追加。  

---

## 5. COP2合成構成（Layered Comp）  
21.0以降のCOP2は、Solaris由来のEXRメタデータ（AOV名、ACESタグ）を自動認識可能。  
これを前提に、**Layered Rendering→COP2合成**構成を設計する。

### 5.1 COP2 Network構成例

```text
/img/comp_main
├─ File COP: /out/BG/<shot>_bg_beauty.####.exr
├─ File COP: /out/CHAR/<shot>_char_beauty.####.exr
├─ File COP: /out/FX/<shot>_fx_beauty.####.exr
├─ ZDepth Merge: zdepth_char / zdepth_bg を基に前後関係を算出
├─ Composite COP: BG → CHAR → FX (Merge Over Mode = Pre-Multiplied)
├─ Grade COP: レイヤ毎のグレーディング調整
├─ Depth of Field COP: zdepthを参照して被写界深度処理
├─ Vector Blur COP: motionVectorを参照し動体ブラー
└─ Output COP: /out/COMP/<shot>_final.####.exr
```

### 5.2 合成上の注意点
- **Pre-Multiplied Alphaの管理**：  
  21.0のKarmaでは、半透明素材出力時に自動Premultiplyが行われる。  
  COP2で再Premultiplyすると二重適用になるため、`Unpremultiply → Composite → Premultiply`の順で統一する。
- **ZDepthのスケーリング**：  
  Solaris出力のZはメートル単位。必要に応じて`Fit Range COP`で正規化。  
- **マスク連携**：  
  AOV内に`cryptomatte`を含めておくと、COP2内でIDベースマスク抽出が可能。  
  Houdini 21.0ではCryptomatte抽出精度が改善され、Subpixel ID処理がサポートされた。  

---

## 6. 実運用上のポイントとパフォーマンス最適化  
### 6.1 レイヤ間整合性の保持
- **シャドウ転写**：  
  背景へのキャラ影を正しく投影するには、背景側マテリアルを`Shadow Matte`に設定し、  
  キャラライトをリンク。これによりCHARレイヤの影をBGレイヤEXRに保持可能。  
- **GI共有**：  
  各レイヤを完全分離するとGIの整合性が崩れるため、GI用ライトキャッシュを別USDに保存し、全レイヤで参照。  

### 6.2 パフォーマンスとキャッシュ運用
- Solaris 21.0では、`USD Asset Resolver`がマルチスレッド化され、  
  サブレイヤのロード速度が最大40％向上（SideFX Release Notesより）。  
- すべてのレイヤを`usdcache`ディレクトリに書き出し、`Sublayer LOP`で読み込む構造にすることで  
  ROP出力時の初期化コストを大幅に削減。  
- Denoiser（Karma XPU Denoiser）はAOVごとに適用可能になったため、  
  beautyだけでなくshadow/zdepthを同時にクリーン化できる。  

### 6.3 トラブルシューティング
| 症状 | 原因 | 対応 |
|------|------|------|
| Render Product出力が空 | LPE式誤り、またはCollection範囲外 | Path Expressionを再確認 |
| COP2で色ズレ | カラースペースメタデータ不一致 | `Configure Layer LOP`でOCIO統一 |
| 合成時にギザギザ | Premultiply二重処理 | Unpremultiply→Composite→Premultiplyへ修正 |
| ライブレンダーで更新されない | Render Gallery連携未設定 | `Live Render LOP`をRender Galleryに接続 |

---

## 7. 総括  
Houdini 21.0におけるSolarisのLayered Renderingは、  
単なるレイヤ出力分割ではなく「USDレイヤ構造＋AOV階層管理＋OCIO統合＋複数レンダーデリゲート」の統合運用体系へと進化した。  
これにより、ショートフィルム規模でも  
- 差し替え容易な再レンダリング  
- GPU/CPU混在の効率的分業  
- 一貫したカラーマネジメント下でのCOP2コンポジット  
を同一シーン構造で実現できる。  
