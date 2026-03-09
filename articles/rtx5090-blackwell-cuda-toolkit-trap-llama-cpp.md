---
title: "【実測】Blackwell × llama.cpp — CUDA Toolkit選択で性能が5倍変わる罠"
emoji: "💣"
type: "tech"
topics: ["llama.cpp", "RTX5090", "CUDA", "LLM", "Blackwell"]
published: true
---

## はじめに

RTX 5090（Blackwell / SM120）で llama.cpp を使っていて、**思ったほど速くない**と感じていませんか？

筆者は前回の記事で Qwen3.5-35B-A3B のベンチマークを公開しましたが、**実はその数値は本来の性能の5分の1しか出ていませんでした。**

![CUDA Toolkit選択によるパフォーマンスの差](/images/cuda-toolkit-performance-gap.png)

原因は2つ：

1. **CUDA Toolkit 13.1 でビルドするとクラッシュまたは大幅劣化する**
2. **`FORCE_CUBLAS=ON` が CMake キャッシュに残っていると、MMQ カーネルが無効化されて遅くなる**

本記事では、同一モデル・同一環境でビルド設定だけを変えた比較データを示し、**Blackwell で llama.cpp の性能を最大化する正しいビルド手順**を解説します。

:::message alert
**前回記事の訂正**: [【検証】RTX 5090でQwen3.5 MXFP4量子化を動かす](https://zenn.dev/toki_mwc/articles/rtx5090-mxfp4-qwen35-llama-cpp-benchmark) で報告した数値（Q4_K_M: pp=937, tg=156）は、`FORCE_CUBLAS=ON`（cuBLAS フォールバック）の状態で計測していました。MMQ を有効にした正しい設定では **pp=5611, tg=211** と大幅に異なります。
:::

## 検証環境

| 項目 | 詳細 |
|------|------|
| GPU | NVIDIA GeForce RTX 5090（32GB VRAM, Blackwell / SM120） |
| llama.cpp | b8240 (d088d5b74) |
| OS | WSL2 Ubuntu 24.04 |
| モデル | Qwen3.5-35B-A3B（MoE, 256 Experts, 8+1活性化） |
| CUDA Toolkit | 12.8.61 / 13.1.115（両方検証） |
| ドライバ | 581.80（CUDA Version: 13.0） |

### 比較対象の量子化

| 量子化 | ファイルサイズ | 方式 |
|--------|-------------|------|
| MXFP4_MOE | 18.42 GiB | ブロック浮動小数点FP4（Expert重み） |
| Q4_K_M | 19.74 GiB | K-quant 4bit混合精度 |

## 結果：ビルド設定で性能が5倍変わる

### Q4_K_M

| ビルド設定 | pp512 (t/s) | tg128 (t/s) | 状態 |
|-----------|:-----------:|:-----------:|------|
| **CUDA 12.8 + MMQ** | **5611 ± 26** | **211 ± 8** | **最速** |
| CUDA 12.8 + cuBLAS | 989 ± 35 | 152 ± 8 | 動作するが遅い |
| CUDA 13.1 + cuBLAS | 772 ± 6 | 121 ± 2 | 動作するがさらに遅い |
| CUDA 13.1 + MMQ | — | — | **Segmentation fault** |

### MXFP4_MOE

| ビルド設定 | pp512 (t/s) | tg128 (t/s) | 状態 |
|-----------|:-----------:|:-----------:|------|
| **CUDA 12.8 + MMQ** | **6566 ± 70** | **201 ± 4** | **最速** |
| CUDA 12.8 + cuBLAS | 1050 ± 9 | 146 ± 4 | 動作するが遅い |
| CUDA 13.1 + cuBLAS | 721 ± 25 | 103 ± 4 | 動作するがさらに遅い |
| CUDA 13.1 + MMQ | — | — | **Segmentation fault** |

**CUDA 12.8 + MMQ** と **CUDA 12.8 + cuBLAS** を比べると：

- **Prompt処理（pp）: 5.7〜6.3倍の差**
- **テキスト生成（tg）: 1.4倍の差**

そして CUDA 13.1 では MMQ がクラッシュするため、cuBLAS にフォールバックせざるを得ず、さらに遅くなります。

## なぜこうなるのか

### 罠1: CUDA 13.1 は Blackwell の MMQ カーネルを壊す

llama.cpp の MMQ（Matrix Multiply Quantized）カーネルは、量子化されたまま行列演算を行う高速パスです。cuBLAS フォールバックは汎用的ですが、量子化のデコード → FP16変換 → cuBLAS演算というオーバーヘッドがあります。

CUDA 13.1 でビルドすると、**Blackwell の MMQ カーネルが Segmentation fault でクラッシュします。** NVIDIA の公式マイグレーションガイドでも、Blackwell（SM120）での llama.cpp は **CUDA 12.8 でのビルドを明示的に推奨**しています。

> *"Building with CUDA 12.8 for compute capability 120 and an upgraded cuBLAS avoids PTX JIT compilation for end users and provides Blackwell-optimized cuBLAS routines."*
> — [NVIDIA Software Migration Guide for Blackwell RTX GPUs](https://forums.developer.nvidia.com/t/software-migration-guide-for-nvidia-blackwell-rtx-gpus-a-guide-to-cuda-12-8-pytorch-tensorrt-and-llama-cpp/321330)

### 罠2: FORCE_CUBLAS=ON が CMake キャッシュに残る

Blackwell 初期（llama.cpp b8157 以前）では MMQ カーネルがクラッシュしていたため、`-DGGML_CUDA_FORCE_CUBLAS=ON` を付けてビルドする必要がありました。

問題は、**この設定が `CMakeCache.txt` に残り続ける**ことです。

```bash
# 古いビルドのキャッシュを確認
grep FORCE_CUBLAS build/CMakeCache.txt
# → GGML_CUDA_FORCE_CUBLAS:BOOL=ON  ← 残っている！
```

`cmake -B build` で再構成しても、キャッシュ済みの値は上書きされません。明示的に `OFF` を指定するか、`build/` ディレクトリを削除する必要があります。

**筆者自身、前回記事のベンチマーク時にこのキャッシュに気づかず、cuBLAS フォールバックの遅い数値を報告していました。**

## 正しいビルド手順

```bash
# 1. CUDA 12.8 を明示的に使用（13.1 と共存していても OK）
export PATH=/usr/local/cuda-12.8/bin:$PATH

# 2. build ディレクトリを完全に削除（キャッシュ残り防止）
rm -rf build

# 3. CMake — CUDA 12.8 を明示指定、FORCE_CUBLAS は OFF
cmake -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES=120 \
  -DGGML_CUDA_FORCE_CUBLAS=OFF \
  -DCUDAToolkit_ROOT=/usr/local/cuda-12.8

# 4. ビルド
cmake --build build --config Release -j $(nproc)
```

### ビルド後の確認

```bash
# FORCE_CUBLAS が OFF であることを確認
grep FORCE_CUBLAS build/CMakeCache.txt
# → GGML_CUDA_FORCE_CUBLAS:BOOL=OFF  ← これが正解

# CUDA Toolkit バージョンを確認
grep CUDAToolkit_VERSION build/CMakeCache.txt
# → 12.8.61 であること
```

## 前回記事のベンチマーク数値の訂正

前回記事（b8196 / CUDA 12.8 / `FORCE_CUBLAS=ON`）と、今回の正しい設定（b8240 / CUDA 12.8 / `FORCE_CUBLAS=OFF`）を比較します。

### Q4_K_M

| 設定 | pp512 (t/s) | tg128 (t/s) |
|------|:-----------:|:-----------:|
| 前回記事（cuBLAS フォールバック） | 937 | 156 |
| **今回（MMQ 有効）** | **5611** | **211** |
| **改善倍率** | **6.0x** | **1.4x** |

### MXFP4_MOE

| 設定 | pp512 (t/s) | tg128 (t/s) |
|------|:-----------:|:-----------:|
| 前回記事（cuBLAS フォールバック） | 1082 | 144 |
| **今回（MMQ 有効）** | **6566** | **201** |
| **改善倍率** | **6.1x** | **1.4x** |

### 用途別の使い分け（更新版）

前回記事の結論（用途別推奨）も更新します。MMQ 有効時のデータで再評価すると：

| ユースケース | 推奨 | 理由 |
|-------------|------|------|
| リアルタイム応答（チャット、配信） | **Q4_K_M** | tg=211 t/s で最速 |
| 長文入力が多い（RAG、長いプロンプト） | **MXFP4_MOE** | pp=6566 t/s（Q4_K_M比 +17%） |
| VRAM に余裕がない | **MXFP4_MOE** | 1GB 少ない VRAM で動作 |

**傾向は前回記事と同じ**ですが、絶対的な速度が大幅に向上しています。

## 実用上の影響

tg=211 t/s という数値は、たとえばリアルタイム会話パイプライン（STT → LLM → TTS）で使う場合、TTS 処理速度の**10倍以上**に相当します。ストリーミング応答時に LLM がボトルネックになることはまずありません。

cuBLAS フォールバック時の tg=152 t/s でも多くのユースケースでは十分ですが、pp（プロンプト処理）の **5〜6倍差** は RAG や長文システムプロンプトで体感的な差になります。

## チェックリスト

Blackwell で llama.cpp を使っている方は、以下を確認してください。

- [ ] CUDA Toolkit は **12.8** を使用しているか（`nvcc --version` で確認）
- [ ] `build/CMakeCache.txt` に `FORCE_CUBLAS:BOOL=ON` が**残っていないか**
- [ ] 過去に `FORCE_CUBLAS=ON` でビルドした場合、**`rm -rf build` してから**再ビルドしたか
- [ ] CUDA 13.1 がインストールされている場合、`-DCUDAToolkit_ROOT=/usr/local/cuda-12.8` で**明示指定**しているか

## まとめ

| 項目 | 結果 |
|------|------|
| CUDA 12.8 + MMQ | **最速**（pp=5611〜6566, tg=201〜211 t/s） |
| CUDA 12.8 + cuBLAS | 動作するが **5〜6倍遅い** |
| CUDA 13.1 + MMQ | **Segmentation fault**（使用不可） |
| CUDA 13.1 + cuBLAS | 動作するが CUDA 12.8 より**さらに遅い** |
| FORCE_CUBLAS の罠 | CMake キャッシュに残ると**気づかず5倍遅い状態**で運用 |

**2026年3月時点では、Blackwell で llama.cpp を使うなら CUDA 12.8 + MMQ が最適な組み合わせです。** CUDA 13.x での MMQ サポートは今後改善される可能性がありますが、現状ではクラッシュするため避けてください。過去に `FORCE_CUBLAS=ON` でビルドしたことがある場合は、必ず `build/` ディレクトリを削除してから再ビルドしてください。

## 参考

- [NVIDIA Software Migration Guide for Blackwell RTX GPUs](https://forums.developer.nvidia.com/t/software-migration-guide-for-nvidia-blackwell-rtx-gpus-a-guide-to-cuda-12-8-pytorch-tensorrt-and-llama-cpp/321330)
- [llama.cpp - GitHub](https://github.com/ggml-org/llama.cpp)
- [前回記事：RTX 5090でQwen3.5 MXFP4量子化を動かす](https://zenn.dev/toki_mwc/articles/rtx5090-mxfp4-qwen35-llama-cpp-benchmark)
- [Blackwell MMQ Issue #18250](https://github.com/ggml-org/llama.cpp/issues/18250)
- [CUDA 13.1 クラッシュ報告 Issue #19868](https://github.com/ggml-org/llama.cpp/issues/19868)
