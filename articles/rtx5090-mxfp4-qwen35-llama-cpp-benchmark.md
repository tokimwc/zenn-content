---
title: "【検証】RTX 5090でQwen3.5 MXFP4量子化を動かす — Q4_K_Mとの性能比較とMMQクラッシュ解消の記録"
emoji: "🔬"
type: "tech"
topics: ["llama.cpp", "RTX5090", "GGUF", "LLM", "量子化"]
published: true
---

## はじめに

MXFP4（Microscaling Floating Point 4-bit）は、ブロック浮動小数点方式で重みを4bitに圧縮する量子化手法です。llama.cppでは `MXFP4_MOE` として MoE（Mixture of Experts）モデル向けにサポートされていますが、**NVIDIA Blackwell世代での動作報告はまだほとんどありません**。

筆者が以前 MXFP4_MOE を試した際には MMQ カーネルのクラッシュで使用を断念しましたが、llama.cpp のバージョンアップにより状況が変わりました。

本記事では、**Blackwell 環境で MXFP4_MOE が動作するようになった経緯と、Q4_K_M との定量的な性能比較**を報告します。

## 検証環境

| 項目        | 詳細                                                             |
| --------- | -------------------------------------------------------------- |
| GPU       | NVIDIA GeForce RTX 5090（32GB VRAM, Blackwell）                  |
| llama.cpp | b8196 (c99909dd0)                                              |
| ビルド       | `cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=120` |
| OS        | WSL2 Ubuntu 24.04                                              |
| モデル       | Qwen3.5-35B-A3B（MoE, 256 Experts, 8+1活性化）                      |

### 比較対象の量子化

| 量子化 | ファイルサイズ | 方式 |
|--------|-------------|------|
| MXFP4_MOE | 18.42 GiB | ブロック浮動小数点FP4（Expert重み） |
| Q4_K_M | 19.7 GiB | K-quant 4bit混合精度 |

## b8157 → b8196：MMQクラッシュ解消の経緯

### 何が起きていたか

llama.cpp b8157 時点では、**Blackwell (SM120) + qwen35moe アーキテクチャ**の組み合わせで MMQ（Matrix Multiply Quantized）カーネルがクラッシュしていました。

```
CUDA error: mmq_x_best=0 (no suitable kernel found)
```

この問題は MXFP4_MOE だけでなく Q4_K_M でも発生しており、当時は回避策として `GGML_CUDA_FORCE_CUBLAS=ON` を付けてビルドし、MMQ を完全にバイパスして cuBLAS にフォールバックさせる必要がありました。

関連 Issue:
- [ggml-org/llama.cpp#18250](https://github.com/ggml-org/llama.cpp/issues/18250)
- [ggml-org/llama.cpp#19683](https://github.com/ggml-org/llama.cpp/issues/19683)

### b8196 での修正

llama.cpp b8196 では Blackwell 向けの MMQ カーネルが修正され、以下の改善が確認できました。

- **FORCE_CUBLAS が不要に** — 通常ビルドで qwen35moe が安定動作
- **CMake が自動で SM120 → 120a に変換** — `CMAKE_CUDA_ARCHITECTURES=120` の指定だけで OK
- **MXFP4_MOE が正常にロード・推論可能に** — 以前はクラッシュしていた量子化形式が完全に動作

:::message
**ポイント**: Blackwell + MoE モデルで MXFP4 量子化を使いたい場合、llama.cpp は **b8196 以降**を使ってください。b8157 以前ではクラッシュします。
:::

## ベンチマーク結果

### llama-bench（pp512 / tg128）

まず `llama-bench` で純粋な推論性能を比較しました。

| 量子化 | サイズ | pp512 (t/s) | tg128 (t/s) |
|--------|--------|:----------:|:----------:|
| **MXFP4_MOE** | **18.42 GiB** | **1082 ± 21** | 144 ± 1.5 |
| Q4_K_M | 19.7 GiB | 937 | **156** |

- **Prompt 処理（pp）**: MXFP4_MOE が **+15%** 高速
- **テキスト生成（tg）**: Q4_K_M が **+8%** 高速

Prompt 処理で MXFP4_MOE が速いのは、モデルサイズが約 1.3GiB 小さい分、メモリ帯域のボトルネックが軽減されるためです。一方、テキスト生成で Q4_K_M が速いのは、FP4 → FP16 のデコード処理のオーバーヘッドが生成フェーズで顕著になるためと考えられます。

### サーバー E2E（テキスト応答）

実際にサーバーを起動し、OpenAI互換APIでテキスト応答のE2Eレイテンシを測定しました。

**サーバー設定**: `--ctx-size 8192 -np 2 --reasoning-budget 0 --flash-attn on --cache-type-k q8_0 --cache-type-v q8_0`

| 量子化 | VRAM使用 | VRAM空き | E2E レイテンシ |
|--------|---------|---------|:------------:|
| **MXFP4_MOE** | **23.7 GB** | **8.4 GB** | 631 - 646 ms |
| Q4_K_M | 24.7 GB | 6.2 GB | 309 - 531 ms |

MXFP4_MOE は **VRAM を約 1GB 節約**できます。この 1GB の差が他プロセス（TTS、画像処理等）との共存に影響するケースがあります。

**応答サンプル（MXFP4_MOE）**:

> わーい！今日も盛り上げるぞ！
> まずはゲーム実況でみんなを笑わせたいな～！
> その後はチャットで雑談もしたいな！
> どんな企画がいいかな？一緒に考えよう！

日本語の応答品質は Q4_K_M と遜色ありません。

### Vision（マルチモーダル）動作確認

Qwen3.5-35B-A3B は**テキスト + Vision の統合モデル**です。`--mmproj` で Vision プロジェクター（F16, 858MB）を追加することで、1プロセスでテキストと画像入力の両方を処理できます。

MXFP4_MOE でも Vision が正常に動作するか検証しました。

**テスト**: 640x480 の PNG 画像（テスト用に生成、下図）を入力し、画像認識させる

![Vision テスト用画像](/images/mxfp4-vision-test-640x480.png)
*テスト用に生成した 640x480 PNG。テキスト3行（白・緑・黄）+ 赤枠のシンプルな構成*

| 量子化 | Vision E2E | 画像認識精度 |
|--------|:----------:|------------|
| MXFP4_MOE | 1621 ms | テキスト・色・レイアウトを正確に認識 |

**応答サンプル**:

> これは、AI Tuber のテスト画面が映っています。深みのある青紫色の背景に、「AI Tuber Test」（白い文字）、「MXFP4_MOE Vision Test」（緑色の文字）、「2026-03-05」（黄色の文字）が表示されています。赤い枠で囲まれた、システムやモデルの能力をテストするための画面のようです。

**MXFP4_MOE + mmproj で Vision も問題なく動作**します。テキスト色やレイアウト構造まで正確に認識できており、品質の劣化は見られませんでした。

## 用途別の使い分け提案

検証結果をもとに、用途に応じた量子化の選択指針をまとめます。

| ユースケース | 推奨 | 理由 |
|-------------|------|------|
| リアルタイム応答（チャット、配信） | **Q4_K_M** | 生成速度 156 t/s で E2E レイテンシが最小 |
| 長文入力が多い（RAG、長いシステムプロンプト） | **MXFP4_MOE** | Prompt 処理が 15% 高速（1082 vs 937 t/s） |
| VRAM に余裕がない（他プロセスと共存） | **MXFP4_MOE** | 1GB 少ない VRAM で動作（23.7 vs 24.7 GB） |
| 日本語品質重視 | **どちらも同等** | 目立った品質差なし |

例えば、AITuber 用途（リアルタイムチャット + TTS 併用）では、**生成速度が E2E レイテンシに直結するため Q4_K_M を維持**しています。ただし、VRAM 節約が必要な場面（ゲーム併用配信等）では MXFP4_MOE への切り替えも選択肢に入ります。

## まとめ

| 項目 | 結果 |
|------|------|
| MXFP4_MOE の動作 | llama.cpp b8196 で **RTX 5090 上で完全に動作** |
| MMQ クラッシュ | b8196 で**解消**（FORCE_CUBLAS 不要） |
| 生成速度 | Q4_K_M が +8% 速い（156 vs 144 t/s） |
| Prompt 処理 | MXFP4_MOE が +15% 速い（1082 vs 937 t/s） |
| VRAM | MXFP4_MOE が 1GB 少ない（23.7 vs 24.7 GB） |
| Vision | MXFP4_MOE でも正常動作・品質劣化なし |

**Blackwell 世代で MXFP4 量子化を試したい方は、llama.cpp b8196 以降にアップデートしてください。** 特に MoE モデル（qwen35moe アーキテクチャ）では、以前のバージョンで発生していた MMQ クラッシュが完全に解消されています。

MXFP4 はまだ対応モデルが限られていますが、MoE モデルの Expert 重みとの相性が良く、今後のエコシステム拡大が期待されます。Blackwell 世代の GPU をお持ちの方は、ぜひ試してみてください。

## 参考

- [llama.cpp - GitHub](https://github.com/ggml-org/llama.cpp)
- [Qwen3.5 - Hugging Face](https://huggingface.co/Qwen)
- [MXFP4 量子化について - llama.cpp Wiki](https://github.com/ggml-org/llama.cpp/wiki)
- [Blackwell MMQ Issue #18250](https://github.com/ggml-org/llama.cpp/issues/18250)
- [Blackwell MMQ Issue #19683](https://github.com/ggml-org/llama.cpp/issues/19683)