---
layout: ../../layouts/BlogLayout.astro
title: A案 LookDevバイブル：Camera & Lens設計 v0.1
description: Houdini 21 / Solaris + Karma に基づく、ショートアニメのカメラ＆レンズ設計ガイド。FOV、DOF、心理効果を考慮したCut別運用方針を整理。
tags: ["LookDev", "カメラ設計", "Houdini", "Karma"]
time: 10
featured: false
timestamp: 2025-10-27T10:42:03+00:00
filename: lookdev_camera_lens
---

## 1️⃣ トーン & カラーパレット
### 基本方針
- **日常⇔異常の境界**を意識したトーン設計
- 現実感を維持しつつ、幻想的表現を加味

| 区分 | キーカラー | 補助カラー | 備考 |
|------|------------|------------|------|
| 店舗（日常） | #F1D7A5 | #3F3C32 | Warm Neutral、蛍光灯風、PBR Roughness 0.5〜0.7 |
| 地下特訓場 | #2A3C6A | #F8B232 | Cool Contrast、青＋オレンジ補色、非現実演出 |
| 壁画 | #9A3B00 | #EFD39B | Saturated Mono、赤土系＋光る文字、神話的演出 |

---

## 2️⃣ Shading
- MaterialXベース
- Karma XPU最適化
- BaseColorリニア値 0.18
- SSS: 人物キャラのみ、半径2〜3mm
- PBR Roughnessマップで「手作業感」を演出
- 壁画：Displacement 0.02〜0.05m（Raytrace推奨）
- ノイズテクスチャ最大512px、Close-up例外1Kまで

---

## 3️⃣ Lighting
- 3-point system + Fill Card 基本
- Solaris Light Mixer 統一管理
- Key Light：Area Light、ソフトシャドウ
- Fill Light：Bounce / Indirect Illumination
- Rim Light：強めのDirectional、逆光演出（社長など）
- EV ±1統一、緊張演出では ±2.5 まで可
- Light Linkerで階層衝突防止、カット単位固定

---

## 4️⃣ Copernicus
- Exposure / Contrast 調整で1カット内完結
- LUT変換後統一優先
- Bloom / Chromatic Aberration：Karma AOV後に追加
- Depth合成：Z-Depth AOV → ZDefocus
- HUD / ロゴ / 字幕：Copernicus USDレイヤー管理
- Zパスはリニア空間で保持、ACES変換前を保存

---

## 5️⃣ 表現方針
| 軸 | 設計方針 |
|-----|----------|
| テイスト | 実写×カートゥーン中間、Pixar風グレーディング |
| スケール感 | 現実的パースを保ちつつ誇張 |
| キャラ | セリフ芝居優先、目線カメラに合わせる |
| ライティング | 光源は物語的意味を持たせる（例：社長＝逆光） |
| レンダー | Karma XPU、リニアワークフロー、ACEScgカラースペース |
---

## 6️⃣ 運用注意
- Close-upや重要カットはテクスチャ解像度・Displacementを調整
- 光源の階層衝突防止、カット単位でSolaris Light Mixer固定
- HUD／テキストはUSDレイヤーで解像度固定
- Render PassごとにACEscgワークフロー統一
- EV調整の緩衝範囲を柔軟に運用し、演出的効果を優先

---

## 7️⃣ 推奨プロセス
1. SOPでジオメトリ／キャラクター構築
2. Solarisでレイアウト／カメラ／ライティング
3. KineFXでモーション／表情演技
4. Karma XPUでプレビュー・本番レンダー
5. COP2＋CopernicusでHUD／字幕／LUT処理
6. 最終コンポジット → ACEScg → 配信フォーマット

---

## 8️⃣ Camera & Lens Design
> Cutごとの演出意図と心理的トーンに沿ったFOV・DOF設計

| Cut ID | カメラ名 | 焦点距離 (mm) | 絞り値 (f) | 被写界深度 | カメラワーク | レンズ特性／狙い | 心理効果 | Houdini設定例 |
|--------|-----------|----------------|------------|------------|---------------|-----------------|-----------|---------------|
| CUT01–03 | Cam_MasterWide | 40 | f/8 | 深い | トラックアップ＋俯瞰 | 店舗全景、生活感 | 安心・群像感 | Solaris Camera＋Dollyパス |
| CUT04–05 | Cam_PresiClose | 50 | f/2.8 | 浅い | 固定＋微ブレ | 社長の狂気・ユーモア強調 | 緊張・異常感 | CameraShakeノード＋Noise CHOP |
| CUT07–09 | Cam_GroupMid | 45 | f/5.6 | 中 | パン／フォロー | チーム結成・群像感 | 日常→非日常心理切替 | Depth Map AOV 合成 |
| CUT10–11 | Cam_FallFX | 35 | f/2.8 | 浅め | 手持ち風 | 崩落・落下臨場感 | 緊張・主題強調 | Motion Path＋CameraFXブラー |
| CUT13–15 | Cam_PanReveal | 30 | f/2.8 | 浅い | パン＋ズームイン | ハヌマーン登場、非日常演出 | 儀式感・圧倒感 | USDカメラロック＋Zoom制御 |
| CUT20–21 | Cam_HaloBack | 70 | f/1.8 | 浅い | 固定 | 社長逆光シルエット、カリスマ演出 | 威厳・象徴性 | Solarisライトリンク＋Bokehエミュ |
| CUT25–29 | Cam_WallMural | 50 (Orth補正) | f/8 | ほぼなし | 直撮り | 壁画象徴性・平面性保持 | 神話・象徴表現 | Orthographic補正有 |
| CUT35–36 | Cam_ActionTrack | 35 | f/2 | 浅い | トラッキング | ハヌマーンレイド動作追従 | 主観・臨場感 | Camera Rig＋KineFXバインド |
| CUT39 | Cam_EndCredit | 28 | f/8 | 深い | トラック後退 | コメディ調エンドロール | 日常感・余韻 | Solaris Sequence Camera |

### 🔹 カテゴリ別 FOV / DOF ルール
- **日常カット**：FOV 40–45mm、深めDOF
- **地下／非日常カット**：FOV 28–35mm、浅いDOF
- **象徴／壁画カット**：FOV 20–50mm（Orth補正可）、DOFほぼなし
- **逆光／強調カット**：50–70mm、浅いDOF、キャラクター強調

### 🔹 運用ガイド
1. Handheld / Trackingカットは Shutter / Motion Blur を標準化
2. 社長逆光や Wall Mural はレンズ補正・DOFで象徴性を統一
3. EV ±1 統一、演出的露出差は ±2.5 まで許容
4. Close-up はノイズ・Displacement 高解像度で例外対応
5. カット単位プリセットを Solaris ノードに保存し再利用
6. 心理効果・演出意図はカメラ名や Cut メモに反映
