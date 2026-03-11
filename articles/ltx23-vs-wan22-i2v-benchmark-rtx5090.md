---
title: "LTX-2.3 22B vs Wan 2.2 14B — I2V速度5.7倍差の裏側とAVパイプラインの罠（RTX 5090検証）"
emoji: "🎬"
type: "tech"
topics: ["ComfyUI", "RTX5090", "AI動画生成", "LTXVideo", "Blackwell"]
published: true
---

## はじめに

2026年3月現在、ローカル環境でのImage-to-Video（I2V）生成は選択肢が急増していますが、ComfyUI上でGGUF量子化して32GB VRAMに載るモデルとなると、実質的に **LTX-2.3 22B** と **Wan 2.2 14B** の二択に絞られます。

今回、**RTX 5090（Blackwell / 32GB VRAM）** でこの2モデルのI2V性能を同一条件（解像度・フレーム数・入力画像・量子化レベル）で比較しました。結果は **LTX-2.3が5.7倍高速**（22.1秒 vs 125秒）という大差。

しかし、この速度差の裏には3つの落とし穴がありました。

- **動かない** — LTX-2.3 22Bは新アーキテクチャ（AVTransformer3DModel）のため、従来のパイプラインでは`split_with_sizes`エラーで生成失敗
- **暴れる** — 蒸留モデルの速度は圧倒的だが、カメラが勝手にズームし、手が破綻する
- **効かない** — GGUF量子化環境ではsageattn3による高速化が無効化される

本記事では、これらの罠にハマって解決するまでの過程と、そこから導いた**用途別モデル使い分け戦略**をワークフロー構築ガイドとしてまとめます。

:::message
**対象読者**:
- ComfyUIで最新のI2Vモデルを導入しようとしている方
- RTX 5090/5080等のBlackwell世代GPUユーザー
- LTX-2.3の導入でエラー（`split_with_sizes`等）に遭遇している方
:::

## 検証環境

| 項目          | 詳細                                  |     |
| ----------- | ----------------------------------- | --- |
| **GPU**     | NVIDIA GeForce RTX 5090 (32GB VRAM) |     |
| **OS**      | Windows 11 (26200.7840)             |     |
| **ComfyUI** | v0.16.4                             |     |
| **Python**  | 3.13.12                             |     |
| **PyTorch** | 2.9.1+cu130                         |     |
| **CUDA**    | 13.0                                |     |

## 罠その1：「LTX-2.3」と思っていたモデルが別物だった

### モデルファイル名だけでは判別できない

LTX-Videoには複数のバージョンが存在し、ファイル命名が紛らわしいです。筆者は当初 `ltx-2-19b-distilled-fp8.safetensors` を「LTX-2.3」だと思い込んで検証を進めていました。

**しかしこれは LTX-2.0 19B でした。**

| ファイル名 | 実際のバージョン | パラメータ数 |
|-----------|-----------------|-------------|
| `ltx-2-19b-distilled-fp8.safetensors` | **LTX-2.0** | 19B |
| `ltx-2-19b-distilled.safetensors` | **LTX-2.0** | 19B |
| `LTX-2.3-distilled-Q4_K_M.gguf` | **LTX-2.3** | 22B |
| `ltx-2.3-22b-distilled_transformer_only_fp8_input_scaled.safetensors` | **LTX-2.3** | 22B |

見分けるポイントは以下の通りです。

- **ファイルサイズ**: LTX-2.0 19B fp8は約23.5GB、LTX-2.3 22B GGUF Q4は約17.8GB
- **ファイル名に `2.3` が明示されている**かどうか
- **モデルアーキテクチャ**: LTX-2.3は `AVTransformer3DModel`（後述の罠その2に直結）

なお、LTX-2.3ではパラメータ数が19B→22Bに増加しています。fp8のまま（23.5GB）では32GB VRAMに載せると余裕がなくなるため、**GGUF Q4_K_M量子化（17.8GB）が実質必須**です。

ちなみに、誤認モデル（LTX-2.0 19B fp8）での実測値は暖機状態で**114秒**。Wan 2.2 14Bの125秒とほぼ同等で、LTX-2.3 22B蒸留（22秒）とは全く別物の性能でした。モデル取り違えに気付いたのは、この「速すぎる」テスト結果がきっかけです。

:::message alert
**教訓**: HuggingFaceからダウンロードしたモデルは、ファイル名だけでなくモデルカードのアーキテクチャ情報を必ず確認しましょう。「ltx-2」という接頭辞はバージョン2.0と2.3の両方に使われています。
:::

## 罠その2：LTX-2.3は従来のComfyUIパイプラインで動かない

### AVTransformer3DModel とは

LTX-2.3 22Bは**Audio+Video統合アーキテクチャ**（AVTransformer3DModel）を採用しています。これは従来のLTX-2.0が映像のみを扱うTransformerだったのに対し、音声と映像を同時に生成する設計です。

そのため、従来のComfyUIの基本ノード構成：

```
CheckpointLoaderSimple → KSampler → VAEDecode
```

この構成では `split_with_sizes` エラーが発生し、**生成が失敗します。**

### 必要な専用ノード

LTX-2.3 22Bを動かすには、[ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) カスタムノードが提供する**AV専用ノード群**が必要です。

筆者は公式サンプルワークフロー（`LTX-2.3_T2V_I2V_Single_Stage_Distilled_Full.json`）の蒸留パスをベースに、GGUF量子化対応 + I2V専用化の変更を加えた21ノード構成のワークフローを構築しました。主な変更点と構成の要点は以下の通りです。

**テキストエンコーダ（Gemma 3 12B）:**

- `LTXAVTextEncoderLoader` — Gemma CLIPモデル + 投影レイヤー（proj-only）を読み込み
- 投影レイヤーはフルチェックポイント（28GB）から抽出して6GBに軽量化可能

**Audio-Video潜在空間の統合:**

- `LTXVEmptyLatentAudio` — 空のオーディオ潜在空間を生成
- `LTXVConcatAVLatent` — 映像潜在空間とオーディオ潜在空間を結合

**サンプリング:**

- `CFGGuider`（cfg=1.0）+ `ManualSigmas` + `SamplerCustomAdvanced`
- 蒸留モデル専用の8段階非均等シグマスケジュール

**デコード:**

- `LTXVSeparateAVLatent` — AV結合潜在空間を映像・音声に分離
- `VAEDecode` — 映像のみデコード（音声は使わない場合は破棄）

### 軽量化テクニック

フルチェックポイント（`ltx-2.3-22b-dev-fp8.safetensors` = 28GB）にはCLIP投影レイヤーやAudio VAEが同梱されていますが、すべてのパーツを別々にロードするため、**必要な部分だけを抽出する**ことでロード時間を短縮できます。

```python
# 投影レイヤーの抽出例（6GB）
import safetensors.torch
full = safetensors.torch.load_file("ltx-2.3-22b-dev-fp8.safetensors")
proj_keys = {k: v for k, v in full.items()
             if not k.startswith(("model.", "first_stage_model.", "conditioner."))}
safetensors.torch.save_file(proj_keys, "ltx-2.3-22b-proj-only.safetensors")
```

これにより、テキストエンコーダロード時に28GBファイル全体を読む必要がなくなり、起動が高速化します。

## ベンチマーク結果：LTX-2.3が5.7倍高速

### テスト条件

両モデルとも **GGUF Q4_K_M 量子化**で統一し、同一の入力画像・解像度・フレーム数で計測しました。

| 項目 | LTX-2.3 22B | Wan 2.2 14B |
|------|-------------|-------------|
| モデルファイル | LTX-2.3-distilled-Q4_K_M.gguf (17.8GB) | HighNoise + LowNoise Q4_K_M (各9GB) |
| 出力解像度 | 832×480 | 832×480 |
| 出力フレーム数 | 81 (3.4秒 @24fps) | 81 (3.4秒 @24fps) |
| ステップ数 | 8（ManualSigmas蒸留スケジュール） | 6（3+3 two-stage方式） |
| サンプラー | euler (CFGGuider, cfg=1.0) | dpm++_sde |
| CLIP | Gemma 3 12B fp4 (8.8GB) | Wan内蔵 |
| パイプライン | AV Pipeline (21ノード) | WanVideoWrapper (16ノード) |

### 速度計測

| 項目 | LTX-2.3 22B | Wan 2.2 14B |
|------|-------------|-------------|
| 暖機状態 | **22.1秒** | **125秒** ※1 |
| コールドスタート | 48.5秒 | 143.9秒 ※2 |
| 速度比（暖機） | **5.7倍高速** ⚡ | 1.0x（基準） |

> ※1 Python 3.13 + sageattn3環境での計測。sageattn3なし（Python 3.12）では143.9秒
> ※2 Python 3.12環境での計測（初回ロード込み）

暖機状態（同じモデルで2回目以降の生成）では、**LTX-2.3が22.1秒、Wan 2.2が125秒**と約5.7倍の速度差が出ました。なお、Wan 2.2の125秒はsageattn3による最適化込みの値です（後述の罠その3で詳説）。

### なぜこれほど差がつくのか

速度差の主因は**アーキテクチャ効率の違い**です。

1. **蒸留モデルの効率**: LTX-2.3 distilledは8ステップの非均等シグマスケジュール（`1.0, 0.99375, 0.9875, ... 0.421875, 0.0`）で、少ないステップ数で高品質な収束を実現
2. **CFGGuider + SamplerCustomAdvanced**: KSamplerベースより低オーバーヘッド
3. **単一モデルローダー**: Wan 2.2は HighNoise + LowNoise の2モデルを切り替える two-stage 方式で、切り替えコストが発生

## 罠その3：GGUF量子化ではsageattnが効かない

### 実測データ

RTX 5090（Blackwell SM120）向けの `sageattn3` をインストールして効果を計測しました。計測はすべて暖機状態（2回目以降の生成）で実施しています。

| モデル | 環境 | 生成時間 | sageattn3効果 |
|--------|------|---------|--------------|
| **LTX-2.3 22B (GGUF Q4)** | Python 3.12（sageattn2） | 22.1秒 | — |
| **LTX-2.3 22B (GGUF Q4)** | Python 3.13（sageattn3） | 22.1秒 | **なし（0%）** |
| **Wan 2.2 14B (GGUF Q4)** | Python 3.12（sageattn2） | 143.9秒 | — |
| **Wan 2.2 14B (GGUF Q4)** | Python 3.13（sageattn3） | 125秒 | **13%改善** ✅ |

LTX-2.3ではPython 3.13 + sageattn3環境に移行しても生成時間は完全に同一でした。一方、Wan 2.2では約13%（143.9→125秒）の高速化が確認されました。

### なぜ差が出るのか

この違いはモデル側のattention実装に起因します。

- **Wan 2.2**: `WanVideoWrapper` が独自のattention実装を持ち、`sageattn3_blackwell` を内部で自動インポートする仕組みがある（`wanvideo/modules/attention.py`）。ノード追加なしで自動的に恩恵を受ける
- **LTX-2.3 (GGUF)**: `UnetLoaderGGUF` 経由のロードでは、GGUF dequantization（4bit→bf16変換）が推論ボトルネックとなり、attention計算はボトルネックではない

つまり、LTX-2.3 GGUFの場合、**推論時間の大部分がdequantizationに費やされている**ため、attention層を高速化しても全体への影響がゼロということです。

:::message
**結論**: sageattn3の効果はモデルのattention実装に依存します。Wan 2.2のようにカスタムノードがsageattnを自動インポートする場合は効果ありですが、GGUF dequantizationがボトルネックのLTX-2.3では効果がありません。なお、RAM飽和時（92%超）はWan 2.2でも逆に50%低下したため、大型モデル連続実行時はComfyUI再起動が重要です。
:::

## 品質比較：速度と品質のトレードオフ

**LTX-2.3 22B 蒸留（22秒）**
<img src="images/ltx23_22b_gguf_av.gif" alt="LTX-2.3 22B 蒸留 I2V結果" width="260" height="150">

**Wan 2.2 14B（125秒）**
<img src="images/wan22_14b_i2v.gif" alt="Wan 2.2 14B I2V結果" width="260" height="250">

> 同一入力画像・同一プロンプトで生成。LTX-2.3はカメラがズームアウトし、動きが大きい。Wan 2.2はカメラ固定で安定した動き。

### 蒸留パイプラインの品質

速度では圧倒的なLTX-2.3ですが、映像品質には課題があります。同じ入力画像・同じプロンプトで目視比較した結果です。

| 評価項目   | LTX-2.3 22B 蒸留 (22s)   | Wan 2.2 14B (125s) |
| ------ | ---------------------- | ------------------ |
| 動きの自然さ | ズームアウト + 髪・マントの揺れ      | マント・髪が風でなびく自然な動き   |
| カメラ安定性 | ❌ 不安定（勝手にズームアウト/スクロール） | ✅ カメラ固定で安定         |
| 手の描写   | ❌ 手が見えると破綻する           | ⚠️ ブレはあるが破綻は少ない    |
| 全体印象   | 速いが暴れやすい               | 安定しているがブレ気味        |

### Fullパイプライン（MultimodalGuider）の試行

品質改善のため、`MultimodalGuider` + `ClownSampler_Beta` + `LTXVScheduler`（15ステップ）を使ったFullパイプラインも試しました。

**結果: 540秒（9分）— 蒸留の24.5倍遅い、Wan 2.2の4.3倍遅い**

しかもFullパイプラインで生成した動画は、背景が夕焼けに変化し、カメラが大きく移動するなど、**蒸留モデル用のパラメータと非蒸留(dev)用のパイプラインの相性が悪く**、意図しない結果になりました。

:::message alert
**注意**: LTX-2.3の `MultimodalGuider` パイプラインは非蒸留（dev）モデル用に設計されています。蒸留モデルで使うと制御が効かず、速度も品質も悪化します。現状、蒸留モデルでは `CFGGuider` + `ManualSigmas` の組み合わせ一択です。
:::

## 用途別モデル使い分け戦略

ベンチマークと品質評価の結果、**用途に応じた使い分け**が最適解という結論に至りました。

| 用途 | 推奨モデル | 生成時間 | 根拠 |
|------|-----------|---------|------|
| **リップシンク（口パク）** | LTX-2.3 蒸留 | **22秒** | 速度が命。顔アップなら手の破綻は映らない |
| **アニメPV・キャラ動画** | Wan 2.2 14B | **125秒** | カメラ安定、手の破綻が少ない |
| **バッチ生成（大量）** | LTX-2.3 蒸留 | **22秒** | 5.7倍差はバッチ処理で決定的 |

### 不採用の選択肢

| パイプライン | 時間 | 不採用理由 |
|-------------|------|-----------|
| LTX-2.3 Full (MultimodalGuider) | 540秒 | 蒸留モデルと相性が悪く、Wan 2.2より4.3倍遅い |
| LTX-2.0 19B fp8 | 114秒 | Wan 2.2と同等速度で品質も劣る（罠その1で述べたモデル誤認時の計測値） |

## ComfyUI v0.16.4 環境構築のポイント

最後に、今回の検証環境の構築で得られた知見をまとめます。

### モデル配置

```
H:\ComfyUI-v16\
├── models/
│   ├── diffusion_models/
│   │   ├── LTX2/
│   │   │   ├── LTX-2.3-distilled-Q4_K_M.gguf        # 17.8GB
│   │   │   └── ltx-2.3-22b-distilled_transformer_only_fp8...  # 23.5GB
│   │   └── Wan22/Base/
│   │       ├── Wan2.2-I2V-A14B-HighNoise-Q4_K_M.gguf # 9GB
│   │       └── Wan2.2-I2V-A14B-LowNoise-Q4_K_M.gguf  # 9GB
│   └── checkpoints/
│       └── LTX2/
│           ├── ltx-2.3-22b-proj-only.safetensors      # 6GB (抽出済み)
│           └── ltx-2.3-22b-audio-vae-only.safetensors  # 102MB (抽出済み)
├── custom_nodes/
│   ├── ComfyUI-LTXVideo/     # AV Pipeline ノード群
│   ├── ComfyUI-WanVideoWrapper/
│   ├── ComfyUI-GGUF/
│   └── ... (40ノード)
```

### 必須カスタムノード

| ノード | リポジトリ | 用途 |
|--------|-----------|------|
| `ComfyUI-LTXVideo` | [Lightricks/ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) | LTX-2.3 AVパイプライン全般 |
| `ComfyUI-GGUF` | [city96/ComfyUI-GGUF](https://github.com/city96/ComfyUI-GGUF) | GGUF量子化モデルの読み込み（UnetLoaderGGUF） |
| `ComfyUI-WanVideoWrapper` | [kijai/ComfyUI-WanVideoWrapper](https://github.com/kijai/ComfyUI-WanVideoWrapper) | Wan 2.2 I2V |
| `ComfyUI-VideoHelperSuite` | [Kosinkadink/ComfyUI-VideoHelperSuite](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite) | 動画出力（VHS_VideoCombine） |
| `ComfyUI-KJNodes` | [kijai/ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) | GGUFLoaderKJ等のユーティリティ |

## まとめ

RTX 5090環境でのLTX-2.3 22B vs Wan 2.2 14B I2V比較から得られた知見をまとめます。

1. **速度**: LTX-2.3蒸留は暖機状態で22.1秒、Wan 2.2の5.7倍高速
2. **品質**: Wan 2.2の方がカメラ安定性・手の描写で優位
3. **パイプライン**: LTX-2.3 22Bは専用AVパイプライン（21ノード）が必須。従来のKSamplerでは動かない
4. **sageattn**: GGUF量子化モデルではdequantizationがボトルネックとなり、sageattn3の効果がモデル依存
5. **Fullパイプライン**: 蒸留モデルとMultimodalGuiderの組み合わせは非実用的
6. **結論**: 速度重視（リップシンク、バッチ生成）→ LTX-2.3、品質重視（PV、キャラ動画）→ Wan 2.2

:::message
**今後の展望**: LTX-2.3の非蒸留(dev) GGUF版が公開されれば、Fullパイプラインの品質改善が期待できます。また、プロンプトエンジニアリング（`fixed camera, no zoom, minimal motion`等）による蒸留パイプラインの安定化も検証の余地があります。
:::

## 参考リンク

- [LTX-Video GitHub](https://github.com/Lightricks/LTX-Video)
- [LTX-2.3 22B HuggingFace モデルカード](https://huggingface.co/Lightricks/LTX-Video-2.3)
- [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo) — 公式ワークフロー同梱
- [city96/ComfyUI-GGUF](https://github.com/city96/ComfyUI-GGUF) — GGUF量子化モデルローダー
- [kijai/ComfyUI-WanVideoWrapper](https://github.com/kijai/ComfyUI-WanVideoWrapper)
- [kijai/ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes)
- [SageAttention (sageattn3)](https://github.com/thu-ml/SageAttention)

---

:::message
**検証環境・データの詳細**: 本記事のベンチマークデータは筆者の環境固有の結果であり、ドライバーバージョン・VRAM使用状況・モデルの組み合わせによって異なる場合があります。再現する場合は、暖機状態（2回目以降の生成）で計測することを推奨します。
:::