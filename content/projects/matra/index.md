---
title: "Matra"
description: "A custom tokenizer algorithm that outperforms GPT-5 and Gemini 3.5 across 22 Indic languages, slashing sequence lengths by 68.7%."
category: "Research / GenAI"
category_order: 10
image: "matra.png"
date: 2026-06-01
math: true
links:
  github: "https://github.com/yumpyy/matra"
  paper: "./matra_paper.pdf"
  huggingface: "https://huggingface.co/yenupam/matra"
draft: false
---

**Technical report.** [Download PDF](./matra_paper.pdf)

## Introducing Matra: An Efficient Script-Aware Tokenizer for Indic LLMs

Today, I am releasing Matra, an open-source tokenizer compiler that sets a new standard for Indic language processing. Standard tokenizers trained primarily on ASCII text lack the vocabulary to represent Devanagari conjuncts or Dravidian agglutination. This creates a proliferation of single-character tokens—a phenomenon called "tokenizer panic"—which inflates sequence lengths and degrades downstream task accuracy. Matra solves this structural mismatch with a script-aware architecture that rescues low-resource scripts from byte-level obliteration.

## Quick picks on Matra

- **Unmatched Performance:** Matra outperforms GPT-5, Gemini 3.5 Flash, Qwen-3.6-MoE, Sarvam-105B, and Sutra-v2 on every primary intrinsic metric.

- **Massive Efficiency:** The tokenizer achieves an aggregate sequence-length reduction of 68.7% and drives the single-character token rate down to 6.9%.

- **Hardware Accessibility:** Built with a pointerless state-machine engine, Matra trains on a 25.2 million word corpus in pure Python in just 3.2 hours on a single CPU core.

- **Low Memory Footprint:** Training requires a peak RAM of only 3.8 GB, keeping it well within consumer laptop limits and completely avoiding external Rust compiler dependencies.

---

## Benchmarks

Matra was evaluated using the standard IN22-Gen held-out benchmark, covering 22 scheduled Indian languages plus English. To ensure true generalization rather than in-distribution memorization, these 1,024 sentences per language were completely unseen during training.

### Aggregate Performance Across 22 Indic Languages + English

The following table compares Matra against leading tokenizers.

| Tokenizer | SeqRed | NSL | BPT | Fert | 1ch% |
| --- | --- | --- | --- | --- | --- |
| **GPT-5** | 46.6% | 0.2154 | 5.79 | 3.71 | 42.5 |
| **Gemini 3.5 Flash** | 53.9% | 0.1873 | 6.45 | 3.20 | 0.0 |
| **Qwen-3.6-MoE** | 34.3% | 0.2632 | 4.35 | 4.60 | 58.9 |
| **Sarvam-105B** | 64.9% | 0.1460 | 7.34 | 2.46 | 29.3 |
| **Sutra-v2** | 64.5% | 0.1463 | 7.26 | 2.49 | 26.1 |
| **Matra-Custom (128K)** | **68.7%** | **0.1316** | **8.21** | **2.17** | **6.9** |

Matra achieves the best aggregate fertility (2.17) and the highest Bytes-Per-Token (8.21), confirming that its tokens encode genuine syllabic and lexical units.

### Rescuing Low-Resource Scripts

The advantages of Matra are exceptionally pronounced on low-resource and structurally complex scripts:

- **Manipuri (Meetei Mayek):** Under GPT-5, this script suffers a 99.6% single-character shredding rate with a fertility of 16.50. Matra effectively rescues the script, dropping fertility to 2.30 and the shredding rate to just 7.3%.

- **Odia and Assamese:** Matra achieves an Odia fertility of 2.04 (compared to GPT-5's 7.31) and an Assamese fertility of 1.94 (compared to GPT-5's 2.97).

- **Cross-Lingual Parity:** Under GPT-5, Indic languages face a hidden "cognitive tax," requiring 4.5x more tokens per word than English. Matra reduces this disparity to a near-uniform parity of 1.4x.

---

## Matra Architecture

Standard tokenizers utilize greedy Byte-Pair Encoding (BPE) that maximizes global frequency, inherently starving low-resource scripts in multilingual corpora. Matra abandons this approach in favor of a specialized, four-pillar architecture.

### 1. Matra-Preserving Pre-tokenization

Standard pre-tokenizers ignore the Unicode Mark class, tearing floating diacritics away from their base consonants. Matra explicitly includes the `\p{M}` (Mark) Unicode class in its regex pattern. This keeps vowel signs, nuktas, and anusvaras fused to their bases, entirely preventing the "floating matra" problem.

### 2. Script-Aware Temperature Scaling

Matra features a two-stage superword curriculum with surgical merge reweighting. During Stage 1 (intra-word morphology), Matra detects target Unicode blocks and applies a temperature scaling formula:

$$w\_{w}=w^{\frac{1-\alpha}{\alpha}}$$

With $\alpha=0.75$, words containing target Indic scripts receive a frequency multiplier ($w\_{w} \approx 1.33$), while English scripts remain unscaled. This ensures fair vocabulary allocation even in code-switched "Hinglish" sentences.

### 3. Immutable Sentence Barriers

While previous models remove whitespace constraints entirely to build "superwords," this causes leaky merges across punctuation. Matra implements hard sentence delimiters during Stage 2, guaranteeing that semantically invalid tokens like "delhi. the" are permanently banned from the vocabulary.

### 4. Pointerless State-Machine Engine

Instead of using standard Python strings that incur massive $O(N)$ list shifting penalties, Matra's engine is built on a pointerless doubly-linked list using C-style integer arrays (`prev_indices` and `next_indices`). Coupled with a lazy max-heap and streaming deduplication, this allows the pure-Python pipeline to bypass out-of-memory crashes.

---

## Build with Matra

Matra proves that highly efficient, structurally coherent tokenizer training is achievable without external compiler toolchains. Its pure-Python implementation makes it highly deployable in resource-constrained environments where compiling Rust is impractical.

For inference, Matra utilizes an identical pointerless array setup and an LRU sentence cache. The resulting $O(N \log N)$ encode loop effortlessly processes millions of words per minute.

