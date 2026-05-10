# Coherence and Asymmetry — EEG Analysis Reference

**English** | [日本語](#日本語)

---

## Overview

This repository contains annotated documentation for the technical paper *Coherence and Asymmetry* (Elektrika, Inc. / Somatics, LLC), which defines the mathematical basis for two EEG analysis indices used in Cymatron-family neurofeedback devices.

The paper itself is concise — one page of formulas and procedure notes. The documentation here unpacks the underlying mathematics, explains why each step matters, and provides a detailed account of implementation pitfalls that are not obvious from the paper alone.

## Contents

| File | Description |
|---|---|
| `Coherence_and_Asymmetry_explanation_en.md` | Annotated explanation — English |
| `Coherence_and_Asymmetry_explanation_jp.md` | Annotated explanation — Japanese |

## What the Paper Covers

The paper defines two per-frequency indices derived from two EEG channels:

**Coherence** (γ²) — measures the strength of phase synchronization between channels.

$$\gamma^2(f) = \frac{|\overline{S_{xy}}(f)|^2}{\overline{S_{xx}}(f) \cdot \overline{S_{yy}}(f)}$$

Range: 0 (no correlation) to 1 (fully synchronized)

**Asymmetry** (A) — measures the power imbalance between channels.

$$A(f) = 2 \cdot \frac{|S_{xx}(f) - S_{yy}(f)|}{S_{xx}(f) + S_{yy}(f)}$$

Range: 0 (symmetric) to 2 (entirely one-sided), expressed as 0–200%

## What the Documentation Adds

The paper assumes familiarity with spectral analysis and leaves several critical implementation details implicit. The documentation here makes them explicit:

- Why computing coherence from a single FFT segment always returns 1.0 — and how to avoid it
- The correct averaging order for the Welch method (real/imaginary parts separately, magnitude squared last)
- Why band aggregation must happen after per-bin coherence is computed, not before
- Six concrete implementation bugs, with symptoms, causes, and fixes
- Mapping of all indices to the columns of the Cymatron Genie *Bands* display
- A result interpretation guide with typical values and patterns of concern

## Reference

> *Coherence and Asymmetry*, Elektrika, Inc. / Somatics, LLC

## License

The documentation in this repository is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to share and adapt the material for any purpose, provided appropriate credit is given.

---

## 日本語

このリポジトリは、Cymatron 系ニューロフィードバック機器で使用される 2 つの EEG 解析指標の数学的根拠を定義した技術論文 *Coherence and Asymmetry*（Elektrika, Inc. / Somatics, LLC）の解説ドキュメントをまとめたものです。

原文は数式と手順の注記をまとめた 1 ページの短い論文です。このドキュメントでは、その背景にある数学を丁寧に展開し、各ステップの意味と、論文だけでは見えにくい実装上の落とし穴を詳しく解説しています。

## 収録ファイル

| ファイル | 内容 |
|---|---|
| `Coherence_and_Asymmetry_explanation_en.md` | 解説ドキュメント — 英語版 |
| `Coherence_and_Asymmetry_explanation_jp.md` | 解説ドキュメント — 日本語版 |

## 論文の定義する指標

2 チャンネルの EEG 信号から、周波数ごとに以下の 2 つの指標を算出します。

**Coherence**（コヒーレンス / γ²）— チャンネル間の位相同期の強さ。

$$\gamma^2(f) = \frac{|\overline{S_{xy}}(f)|^2}{\overline{S_{xx}}(f) \cdot \overline{S_{yy}}(f)}$$

範囲: 0（無相関）〜 1（完全同期）

**Asymmetry**（非対称性 / A）— チャンネル間のパワーの偏り。

$$A(f) = 2 \cdot \frac{|S_{xx}(f) - S_{yy}(f)|}{S_{xx}(f) + S_{yy}(f)}$$

範囲: 0（対称）〜 2（完全に片側）、0〜200% で表記

## ドキュメントの内容

論文はスペクトル解析の事前知識を前提としており、実装に必要な細部は省略されています。解説ドキュメントではそれらを補足しています。

- 1 セグメントの FFT からコヒーレンスを計算すると必ず 1.0 になる理由、およびその回避方法
- Welch 法における正しい平均化の順序（実部・虚部を別々に平均し、最後に絶対値の二乗を取る）
- 帯域集計を bin 単位のコヒーレンス算出より後に行わなければならない理由
- 症状・原因・修正を示した実装上の落とし穴 6 例
- Cymatron Genie の *Bands* 表示の各列と各指標の対応
- 典型値と注意すべきパターンを含む結果の解釈ガイド

## 参考文献

> *Coherence and Asymmetry*, Elektrika, Inc. / Somatics, LLC

## ライセンス

本リポジトリの解説ドキュメントは
[クリエイティブ・コモンズ 表示 4.0 国際（CC BY 4.0）](https://creativecommons.org/licenses/by/4.0/deed.ja)
のもとで公開しています。

出典を明記すれば、目的を問わず自由に共有・改変・再配布できます。
