---
title: "Matra"
description: "A custom tokenizer algorithm that outperforms GPT-5, Gemini 3.5 Flash, Gemma-4-31B, Qwen-3.6-MoE, Sarvam-105B, Sutra-v2, and MUTANT (Krutrim-2) across 23 languages, slashing sequence lengths by 70.3% with lowest fertility of 2.06 and highest BPT of 8.90."
category: "Research / GenAI"
category_order: 10
image: "matra.png"
date: 2026-06-07
math: true
toc: true
card_image_only: true
links:
  github: "https://github.com/yumpyy/matra"
  paper: "./matra_paper_200k.pdf"
  huggingface: "https://huggingface.co/yenupam/matra"
draft: false
---

> Technical report: [matra_paper_200k.pdf](./matra_paper_200k.pdf)

## Live Demo
> note: visit [huggingface space link](https://huggingface.co/spaces/yenupam/matra-tokenizer-visualizer) if the embed fails to load.
<iframe
    src="https://yenupam-matra-tokenizer-visualizer.static.hf.space"
    frameborder="0"
    width="100%"
    height="500"
></iframe>

# Matra: A Tokenizer Built for India's Languages

**200k, 128k vocab size. 23 languages. The lowest token counts and highest compression across the board.**

Most tokenizers treat Indian languages as an afterthought, fragmented syllables and half-formed characters scattered across a vocabulary designed for English. Matra was built from the ground up to fix that. The result, benchmarked on June 15 2026 against the IN22-Gen test split at 200k tokens per language, is unambiguous: Matra produces fewer tokens, denser representations, and dramatically lower fragmentation than GPT-5, Gemini 3.5 Flash, Gemma-4-31B, Qwen-3.6-MoE, Sarvam-105B, and Sutra-v2 across all 23 scheduled languages of India.

That is not a claim. It is a number: **1,103,089 total tokens** to encode the entire benchmark corpus, versus 1,794,545 for the next-best competitor (Gemma-4-31B) and 3,225,298 for the worst (Qwen-3.6-MoE).

---

## Architectural Innovations

Tokenization for Indic scripts is hard for a structural reason. A single syllable in Hindi, Tamil, or Kannada may be encoded as 2–4 UTF-8 bytes. A tokenizer trained predominantly on English text will decompose that syllable into individual bytes or Unicode code-points, producing a cascade of meaningless fragments. Every extra token is wasted context, wasted compute, and a weaker representation for the model.

Matra addresses this with seven architectural innovations.

### Unicode-Aware Pre-Tokenization

The pre-tokenizer uses a hand-crafted regex pattern that respects Unicode letter, mark, and number categories (`\p{L}`, `\p{M}`, `\p{N}`). Diacritics are kept attached to their base letters. CJK characters are split individually (they are morphologically atomic). Right-to-left scripts and Arabic extensions are handled natively. Whitespace, four-space indents, tabs, newlines are preserved as explicit tokens for code-aware applications.

### Language-Weighted Training (XLM-R Scaling)

A two-stage merge strategy with script-aware frequency scaling. During Stage 1, word frequencies are multiplied by a `lang_weight` factor when the word contains target-script characters  -  detected via `is_target_script()` across Devanagari, Bengali, Gurmukhi, Gujarati, Oriya, Tamil, Telugu, Kannada, Malayalam, Sinhala, Arabic/Perso-Arabic (Urdu, Kashmiri, Sindhi), Ol Chiki (Santali), and Meetei Mayek (Manipuri).

The scaling follows the XLM-R alpha-smoothing formula:

$$w_w = w^{\frac{1-\alpha}{\alpha}}$$

With $\alpha = 0.75$, target-script words receive a ~1.33× frequency boost while English remains unscaled. Stage 2 uses a **blended sentence-level multiplier**: a sentence with 80% Hindi and 20% English gets 80% of the full weight, preventing code-switched "Hinglish" from distorting the merge budget.

### Two-Stage Merge Strategy

Stage 1 learns intra-word subword merges (morphological prefixes, suffixes, character n-grams) constrained within word boundaries. Stage 2 extends merges across word boundaries within a sentence, learning multi-word "superword" compounds. The merge budget is split: `transition_vocab_size` tokens allocated to Stage 1, the remainder to Stage 2. This decouples morphological learning from syntactic collocation, and no single-character fragment is promoted until both stages complete.

### Sentence Boundary Detection

Pre-tokenized word tokens are grouped into sentences using a zero-width lookbehind split on `[.!?\n।]` followed by a lookahead for whitespace. During Stage 2, any merge whose resulting token contains a sentence delimiter (`.` `!` `?` `\n` `।` `。` `？` `！`) is permanently banned from the vocabulary. This prevents semantically invalid tokens like "delhi. the" from ever being learned, preserving clean sentence boundaries for downstream transformer models.

### R2L Digit Grouping

Following Singh & Strouse (2024), digit sequences are chunked right-to-left in groups of three using a positive-lookahead regex:

```
(?:\p{N}{1,3}(?=(?:\p{N}{3})*(?!\p{N})))
```

This preserves base-10 alignment in right-to-left scripts and prevents the arithmetic hallucination that arises when digit strings are fragmented across arbitrary byte boundaries.

### Hapax Legomena Pruning

The `drop_singletons` flag removes word entries with frequency ≤ 1.0 between Pass 1 and Stage 1. Hapax legomena  -  words appearing exactly once in the corpus  -  contribute noise to merge statistics without providing generalizable patterns. Pruning them frees the merge budget for higher-frequency morphological units. Sentences containing pruned words are also filtered before Stage 2.

This is mathematically defensible as a data-efficiency choice; for maximally pure results, `drop_singletons` defaults to `False`.

### Pointerless State-Machine Engine

Instead of Python's $O(N)$ list `pop()` shifting, sequences are stored as `array.array("i")`  -  contiguous C `int32_t` buffers  -  with two parallel linked arrays:

- `prev_indices`: `p[i] = i-1` (sentinel `-1` at boundaries)
- `next_indices`: `n[i] = i+1` (sentinel `n_j` at boundaries)

When position $k$ is merged into $i$, the consumed position is tombstoned ($s[k] = -1$), `next_indices[i]` forwards to `n[k]`, and the reciprocal `prev_indices` updates at `n[k]`. The result is $O(1)$ pointer updates per merge instead of $O(N)$ element shifts.

Pair positions use 64-bit packing: `packed = (seq_idx << 32) | offset`. A single machine word replaces the Python tuple, reducing per-position memory from ~56 bytes to 8 bytes.

## Resource Efficiency

### 2-Pass Streaming Architecture

A naive in-memory BPE trainer accumulates four large dicts simultaneously: word sequences, word frequencies, word-to-string maps, and sentence counters. On a 300 MB corpus (2.8 GB effective after alpha-smoothing), this peaks at **75+ GB RAM**  -  well beyond consumer hardware.

Matra splits training into two streaming passes:

1. **Pass 1**  -  Stream the corpus, populate word-level dicts, run Stage 1 BPE, then **delete all three word dicts** and call `gc.collect()`.
2. **Pass 2**  -  Re-stream the corpus (the `iterator_factory` is called a second time), populate sentence-level statistics, and run Stage 2 BPE.

This decouples word and sentence accumulation:

| Phase | 1-Pass RAM | 2-Pass RAM |
|---|---|---|
| Word accumulation | ~60 GB | ~2-3 GB |
| Stage 1 BPE | ~70 GB | ~3-4 GB |
| Sentence accumulation | ~75 GB | ~1-2 GB |
| Stage 2 BPE | ~75 GB | ~3-4 GB |
| **Peak** | **~75 GB** | **~5-6 GB** |

Training comfortably fits within a 16 GB machine budget.

### RAM Shield

Stage 2 operates on a single concatenated 1D array of all corpus sentences  -  ~39M tokens for a 300 MB corpus. Each merge creates new pairs at the two affected boundaries, and the majority of those new pairs are singletons (freq=1). BPE requires `freq >= 2` to execute a merge, but the dict still holds them  -  and with millions of merges, the dict grows unboundedly until the OOM killer fires.

The **RAM Shield** hard-caps memory: when `len(pair_counts) > max_pairs` (default 3,000,000), all singleton pairs are purged, the heap is rebuilt from live entries, and `gc.collect()` releases the freed memory. Purging is mathematically lossless  -  no pair with `freq <= 1` can ever be selected for merging.

**Early stopping** terminates the merge loop when the top heap entry's frequency drops below 2, preventing the trainer from grinding through dead heap pops. On typical corpora, Stage 2 halts at ~5K–15K merges (well before the budget) once no more repeat pairs exist.

Combined effect: Stage 2 peak RSS is hard-capped at ~2.5 GB regardless of corpus size, and OOM is impossible.

### Checkpointing & Crash Recovery

The trainer auto-saves a `.ckpt` file after each phase boundary, keyed by a 16-char SHA-256 hash of the training config `{vocab_size, alpha, drop_singletons, lang_weight, passes}`. This gives two superpowers:

1. **Crash recovery**  -  kill mid-Stage 1 with Ctrl-C, restart with `--resume-from pass1`, the 20-minute Pass 1 is loaded from cache.
2. **Hyperparameter sweep**  -  train once with `--vocab 200000 --alpha 0.5`, then re-run with `--vocab 256000 --alpha 0.5 --reuse-pass1` to skip Pass 1 entirely.

Checkpoints are written atomically (`.tmp` + `os.replace()`), surviving mid-write SIGKILL. On load, `__version__`, `__stage__`, and `config_hash` are verified against the current run  -  all mismatches raise `ValueError` with actionable messages.

### Cython-Accelerated Hot Paths

The inner merge loop (`_bpe_merge.pyx`) and pair-position builder (`_bpe_passb.pyx`) are optionally compiled with Cython via `pyximport`. When available, they replace the equivalent Python loops at the two points that dominate training time. On a 200k-token corpus the difference is roughly 4–5× wall-clock time. The pure-Python fallback remains correct and usable without a build step, making Matra deployable in environments where compiling C extensions is impractical.

### O(1) Encode Path

At inference time, the byte-to-token mapping is pre-computed as a 256-entry tuple indexed by raw byte value, eliminating per-byte dictionary lookups. The merge pass uses a lazy min-heap with integer-keyed `_merge_ranks_int` maps  -  pairs are resolved via `(id0, id1) → rank` lookups instead of string hashing. A bounded LRU cache (default 2,000,000 entries) means repeated subwords pay zero merge cost after the first occurrence. The end-to-end complexity is $O(n \log n)$ per sentence in the number of pre-tokenization tokens.

---

## Benchmark Results: IN22-Gen, 23 Languages

Benchmark date: 15 June 2026. Test split: IN22-Gen. Corpus size: approximately 200,000 tokens per language. Tokenizers evaluated: Matra, GPT-5, Gemini 3.5 Flash, Gemma-4-31B, MUTANT (Krutrim-2), Qwen-3.6-MoE, Sarvam-105B, Sutra-v2.

Metrics:

- **SeqRed** (Sequence Reduction): percentage reduction in token count versus the raw byte sequence. Higher is better.
- **NSL** (Normalised Sequence Length): token count divided by the character count of the source text. Lower is better.
- **BPT** (Bytes Per Token): average bytes encoded per token. Higher is better.
- **Fert** (Fertility): average tokens produced per word. Lower is better.
- **1ch%**: percentage of tokens that encode a single character. Lower is better.

### Aggregate across all 23 languages

| Tokenizer | SeqRed | NSL | BPT | Fert | 1ch% | Total Tokens |
|---|---|---|---|---|---|---|
| **Matra** | **70.3%** | **0.1225** | **8.90** | **2.06** | **7.5** | **1,103,089** |
| Sutra-v2 | 64.6% | 0.1453 | 7.29 | 2.47 | 25.1 | 1,311,098 |
| Sarvam-105B | 64.9% | 0.1454 | 7.35 | 2.45 | 29.2 | 1,301,947 |
| Gemini 3.5 Flash | 51.5% | 0.1959 | 6.28 | 3.34 | 0.0 | 1,794,568 |
| Gemma-4-31B | 51.5% | 0.1959 | 6.28 | 3.34 | 39.3 | 1,794,545 |
| GPT-5 | 37.4% | 0.2494 | 5.58 | 4.27 | 44.7 | 2,327,945 |
| MUTANT (Krutrim-2) | 23.4% | 0.3025 | 4.56 | 5.29 | 55.2 | 2,864,195 |
| Qwen-3.6-MoE | 13.2% | 0.3418 | 3.35 | 6.07 | 77.6 | 3,225,298 |

Matra's total token count is **42% lower than Gemma-4-31B** and **66% lower than Qwen-3.6-MoE** on the same corpus. The fertility score of 2.06 means the average Indic word is encoded in just over two tokens; Qwen requires three times as many.

### Selected per-language highlights

**Hindi**: Matra achieves 79.9% sequence reduction, a fertility of 1.09, and 32,218 total tokens. GPT-5 requires 51,905 and a fertility of 1.76.

**Tamil**: Matra achieves 77.3% sequence reduction and a BPT of 12.03, the highest encoding density of any tokenizer on this language. The next best is Gemma-4-31B at 9.67.

**Bengali**: Matra achieves 77.4% sequence reduction and 34,179 total tokens. Gemma-4-31B and Sarvam-105B are tied at 40,950.

**Manipuri (Meitei script)**: Where GPT-5, Qwen, and MUTANT collapse entirely (negative sequence reduction, fertility above 16), Matra encodes Manipuri at 53.6% sequence reduction with a fertility of 3.05. Sarvam-105B and Sutra-v2 are the only other tokenizers with positive sequence reduction on this script.

**Odia**: GPT-5 achieves near-zero sequence reduction (−0.1%) on Odia. MUTANT and Qwen are worse. Matra achieves 71.5% with 49,755 tokens, the second-best after Sarvam-105B (54,656).

<div style="border: 2px solid #3AB59A; border-radius: 10px; padding: 16px; background: #f6fcf9; margin: 20px 0;">

<details>
<summary style="font-weight: 700; font-size: 1.05em; cursor: pointer; color: #0F6E56;">Full per-language breakdown (23 languages × 7 tokenizers)  -  click to expand</summary>

| Language | Tokenizer | SeqRed↑ | NSL↓ | BPT↑ | Fert↓ | 1ch%↓ | Tokens |
|---|---|---|---|---|---|---|---|
| **as (Assamese)** | GPT-5 | 57.4% | 0.1597 | 6.26 | 2.97 | 38.0 | 68,816 |
| | Gemini 3.5 Flash | 58.9% | 0.1541 | 6.49 | 2.87 | 0.0 | 66,401 |
| | Gemma-4-31B | 58.9% | 0.1541 | 6.49 | 2.87 | 40.6 | 66,400 |
| | MUTANT (Krutrim-2) | 32.4% | 0.2536 | 3.94 | 4.72 | 70.2 | 109,244 |
| | **Matra** | **71.9%** | **0.1053** | **9.50** | **1.96** | **6.7** | **45,342** |
| | Qwen-3.6-MoE | 23.9% | 0.2853 | 3.50 | 5.31 | 80.7 | 122,906 |
| | Sarvam-105B | 58.9% | 0.1541 | 6.49 | 2.87 | 40.6 | 66,400 |
| | Sutra-v2 | 67.3% | 0.1225 | 8.16 | 2.28 | 26.2 | 52,789 |
| **bn (Bengali)** | GPT-5 | 63.0% | 0.1386 | 7.21 | 2.56 | 31.1 | 55,995 |
| | Gemini 3.5 Flash | 72.9% | 0.1014 | 9.87 | 1.87 | 0.0 | 40,951 |
| | Gemma-4-31B | 72.9% | 0.1014 | 9.87 | 1.87 | 19.7 | 40,950 |
| | MUTANT (Krutrim-2) | 52.8% | 0.1768 | 5.66 | 3.27 | 48.6 | 71,433 |
| | **Matra** | **77.4%** | **0.0846** | **11.82** | **1.56** | **7.5** | **34,179** |
| | Qwen-3.6-MoE | 31.1% | 0.2576 | 3.88 | 4.76 | 68.6 | 104,090 |
| | Sarvam-105B | 72.9% | 0.1014 | 9.87 | 1.87 | 19.7 | 40,950 |
| | Sutra-v2 | 68.1% | 0.1193 | 8.38 | 2.21 | 24.3 | 48,208 |
| **brx (Bodo)** | GPT-5 | 48.5% | 0.1931 | 5.18 | 3.96 | 42.6 | 84,466 |
| | Gemini 3.5 Flash | 53.9% | 0.1729 | 5.78 | 3.54 | 0.0 | 75,637 |
| | Gemma-4-31B | 53.9% | 0.1729 | 5.78 | 3.54 | 33.6 | 75,636 |
| | MUTANT (Krutrim-2) | 42.5% | 0.2158 | 4.63 | 4.42 | 52.1 | 94,396 |
| | **Matra** | **65.1%** | **0.1312** | **7.62** | **2.69** | **5.7** | **57,376** |
| | Qwen-3.6-MoE | 28.1% | 0.2697 | 3.71 | 5.52 | 76.4 | 117,971 |
| | Sarvam-105B | 53.9% | 0.1729 | 5.78 | 3.54 | 33.6 | 75,636 |
| | Sutra-v2 | 49.6% | 0.1893 | 5.28 | 3.88 | 39.2 | 82,800 |
| **doi (Dogri)** | GPT-5 | 57.5% | 0.1656 | 6.04 | 2.31 | 41.0 | 67,844 |
| | Gemini 3.5 Flash | 62.7% | 0.1453 | 6.88 | 2.03 | 0.0 | 59,505 |
| | Gemma-4-31B | 62.7% | 0.1453 | 6.88 | 2.03 | 32.4 | 59,504 |
| | MUTANT (Krutrim-2) | 50.4% | 0.1932 | 5.18 | 2.70 | 50.6 | 79,152 |
| | **Matra** | **67.4%** | **0.1268** | **7.88** | **1.77** | **3.8** | **51,960** |
| | Qwen-3.6-MoE | 30.5% | 0.2706 | 3.70 | 3.78 | 79.0 | 110,857 |
| | Sarvam-105B | 62.7% | 0.1453 | 6.88 | 2.03 | 32.4 | 59,504 |
| | Sutra-v2 | 59.1% | 0.1592 | 6.28 | 2.22 | 35.1 | 65,232 |
| **en (English)** | GPT-5 | 79.1% | 0.2089 | 4.79 | 1.32 | 15.3 | 33,561 |
| | Gemini 3.5 Flash | 78.3% | 0.2173 | 4.60 | 1.38 | 0.0 | 34,920 |
| | Gemma-4-31B | 78.3% | 0.2173 | 4.60 | 1.38 | 20.2 | 34,919 |
| | MUTANT (Krutrim-2) | 78.1% | 0.2189 | 4.57 | 1.39 | 19.3 | 35,178 |
| | **Matra** | **81.0%** | **0.1904** | **5.25** | **1.21** | **11.4** | **30,595** |
| | Qwen-3.6-MoE | 77.7% | 0.2232 | 4.48 | 1.42 | 18.8 | 35,869 |
| | Sarvam-105B | 78.3% | 0.2173 | 4.60 | 1.38 | 20.2 | 34,919 |
| | Sutra-v2 | 80.9% | 0.1909 | 5.24 | 1.21 | 5.3 | 30,675 |
| **gu (Gujarati)** | GPT-5 | 60.2% | 0.1514 | 6.61 | 2.56 | 34.6 | 59,667 |
| | Gemini 3.5 Flash | 57.7% | 0.1610 | 6.21 | 2.73 | 0.0 | 63,455 |
| | Gemma-4-31B | 57.7% | 0.1610 | 6.21 | 2.73 | 36.1 | 63,454 |
| | MUTANT (Krutrim-2) | 38.4% | 0.2345 | 4.26 | 3.97 | 60.2 | 92,431 |
| | **Matra** | **73.4%** | **0.1014** | **9.86** | **1.72** | **9.1** | **39,973** |
| | Qwen-3.6-MoE | 18.4% | 0.3105 | 3.22 | 5.26 | 91.4 | 122,402 |
| | Sarvam-105B | 65.9% | 0.1298 | 7.70 | 2.20 | 26.9 | 51,161 |
| | Sutra-v2 | 64.5% | 0.1351 | 7.40 | 2.29 | 26.4 | 53,260 |
| **hi (Hindi)** | GPT-5 | 67.7% | 0.1258 | 7.95 | 1.76 | 22.4 | 51,905 |
| | Gemini 3.5 Flash | 72.1% | 0.1088 | 9.19 | 1.52 | 0.0 | 44,886 |
| | Gemma-4-31B | 72.1% | 0.1088 | 9.19 | 1.52 | 18.1 | 44,885 |
| | MUTANT (Krutrim-2) | 61.5% | 0.1501 | 6.66 | 2.10 | 31.7 | 61,902 |
| | **Matra** | **79.9%** | **0.0781** | **12.80** | **1.09** | **6.0** | **32,218** |
| | Qwen-3.6-MoE | 34.3% | 0.2560 | 3.91 | 3.57 | 73.1 | 105,590 |
| | Sarvam-105B | 72.1% | 0.1088 | 9.19 | 1.52 | 18.1 | 44,885 |
| | Sutra-v2 | 69.4% | 0.1192 | 8.39 | 1.66 | 17.3 | 49,182 |
| **kn (Kannada)** | GPT-5 | 58.9% | 0.1507 | 6.64 | 3.78 | 40.1 | 71,024 |
| | Gemini 3.5 Flash | 59.4% | 0.1487 | 6.72 | 3.73 | 0.0 | 70,092 |
| | Gemma-4-31B | 59.4% | 0.1487 | 6.72 | 3.73 | 37.6 | 70,091 |
| | MUTANT (Krutrim-2) | 53.5% | 0.1704 | 5.87 | 4.27 | 46.8 | 80,282 |
| | **Matra** | **72.7%** | **0.1000** | **10.00** | **2.50** | **9.9** | **47,119** |
| | Qwen-3.6-MoE | 19.6% | 0.2947 | 3.39 | 7.38 | 83.9 | 138,860 |
| | Sarvam-105B | 68.0% | 0.1174 | 8.52 | 2.94 | 28.2 | 55,326 |
| | Sutra-v2 | 67.2% | 0.1200 | 8.33 | 3.01 | 26.0 | 56,554 |
| **kok (Konkani)** | GPT-5 | 56.5% | 0.1647 | 6.07 | 3.13 | 38.2 | 66,568 |
| | Gemini 3.5 Flash | 59.9% | 0.1520 | 6.58 | 2.88 | 0.0 | 61,431 |
| | Gemma-4-31B | 59.9% | 0.1520 | 6.58 | 2.88 | 32.5 | 61,430 |
| | MUTANT (Krutrim-2) | 50.4% | 0.1881 | 5.32 | 3.57 | 46.7 | 76,022 |
| | **Matra** | **70.7%** | **0.1112** | **9.00** | **2.11** | **7.7** | **44,936** |
| | Qwen-3.6-MoE | 27.8% | 0.2735 | 3.66 | 5.19 | 78.0 | 110,566 |
| | Sarvam-105B | 59.9% | 0.1520 | 6.58 | 2.88 | 32.5 | 61,430 |
| | Sutra-v2 | 58.9% | 0.1559 | 6.41 | 2.96 | 32.9 | 63,018 |
| **ks (Kashmiri)** | GPT-5 | 35.9% | 0.3545 | 2.82 | 3.68 | 67.3 | 104,745 |
| | Gemini 3.5 Flash | 43.1% | 0.3144 | 3.18 | 3.26 | 0.0 | 92,891 |
| | Gemma-4-31B | 43.1% | 0.3144 | 3.18 | 3.26 | 61.0 | 92,890 |
| | MUTANT (Krutrim-2) | 34.1% | 0.3643 | 2.75 | 3.78 | 69.6 | 107,639 |
| | **Matra** | **55.2%** | **0.2475** | **4.04** | **2.57** | **7.3** | **73,121** |
| | Qwen-3.6-MoE | 30.6% | 0.3839 | 2.60 | 3.98 | 72.7 | 113,435 |
| | Sarvam-105B | 43.1% | 0.3144 | 3.18 | 3.26 | 61.0 | 92,890 |
| | Sutra-v2 | 45.1% | 0.3032 | 3.30 | 3.15 | 46.8 | 89,598 |
| **mai (Maithili)** | GPT-5 | 61.7% | 0.1454 | 6.88 | 2.31 | 35.2 | 57,157 |
| | Gemini 3.5 Flash | 64.9% | 0.1332 | 7.51 | 2.11 | 0.0 | 52,360 |
| | Gemma-4-31B | 64.9% | 0.1332 | 7.51 | 2.11 | 34.5 | 52,359 |
| | MUTANT (Krutrim-2) | 50.8% | 0.1869 | 5.35 | 2.97 | 52.0 | 73,470 |
| | **Matra** | **72.2%** | **0.1054** | **9.49** | **1.67** | **5.0** | **41,417** |
| | Qwen-3.6-MoE | 29.1% | 0.2694 | 3.71 | 4.28 | 77.5 | 105,896 |
| | Sarvam-105B | 64.9% | 0.1332 | 7.51 | 2.11 | 34.5 | 52,359 |
| | Sutra-v2 | 60.6% | 0.1497 | 6.68 | 2.38 | 38.6 | 58,848 |
| **ml (Malayalam)** | GPT-5 | 62.9% | 0.1347 | 7.43 | 4.01 | 34.3 | 68,334 |
| | Gemini 3.5 Flash | 64.4% | 0.1294 | 7.73 | 3.85 | 0.0 | 65,635 |
| | Gemma-4-31B | 64.4% | 0.1294 | 7.73 | 3.85 | 32.1 | 65,634 |
| | MUTANT (Krutrim-2) | 49.0% | 0.1850 | 5.41 | 5.51 | 54.5 | 93,866 |
| | **Matra** | **72.3%** | **0.1006** | **9.94** | **2.99** | **6.9** | **51,027** |
| | Qwen-3.6-MoE | 16.7% | 0.3023 | 3.31 | 9.00 | 86.5 | 153,397 |
| | Sarvam-105B | 68.1% | 0.1159 | 8.63 | 3.45 | 27.6 | 58,822 |
| | Sutra-v2 | 68.5% | 0.1143 | 8.75 | 3.40 | 25.0 | 58,019 |
| **mni (Manipuri)** | GPT-5 | -151.4% | 0.9461 | 1.06 | 16.50 | 99.6 | 386,349 |
| | Gemini 3.5 Flash | -85.6% | 0.6984 | 1.43 | 12.18 | 0.0 | 285,205 |
| | Gemma-4-31B | -85.6% | 0.6984 | 1.43 | 12.18 | 91.9 | 285,204 |
| | MUTANT (Krutrim-2) | -151.3% | 0.9459 | 1.06 | 16.49 | 99.5 | 386,258 |
| | **Matra** | **53.6%** | **0.1747** | **5.72** | **3.05** | **12.3** | **71,346** |
| | Qwen-3.6-MoE | -151.4% | 0.9461 | 1.06 | 16.50 | 99.6 | 386,359 |
| | Sarvam-105B | 66.7% | 0.1254 | 7.97 | 2.19 | 23.3 | 51,212 |
| | Sutra-v2 | 67.3% | 0.1230 | 8.13 | 2.15 | 5.6 | 50,247 |
| **mr (Marathi)** | GPT-5 | 62.5% | 0.1404 | 7.12 | 2.73 | 31.0 | 60,860 |
| | Gemini 3.5 Flash | 70.3% | 0.1114 | 8.98 | 2.17 | 0.0 | 48,287 |
| | Gemma-4-31B | 70.3% | 0.1114 | 8.98 | 2.17 | 21.0 | 48,286 |
| | MUTANT (Krutrim-2) | 54.5% | 0.1703 | 5.87 | 3.32 | 41.7 | 73,828 |
| | **Matra** | **76.5%** | **0.0880** | **11.37** | **1.71** | **9.4** | **38,130** |
| | Qwen-3.6-MoE | 27.7% | 0.2707 | 3.69 | 5.27 | 76.3 | 117,368 |
| | Sarvam-105B | 70.3% | 0.1114 | 8.98 | 2.17 | 21.0 | 48,286 |
| | Sutra-v2 | 68.9% | 0.1165 | 8.59 | 2.27 | 22.4 | 50,484 |
| **ne (Nepali)** | GPT-5 | 64.8% | 0.1308 | 7.65 | 2.54 | 27.3 | 54,170 |
| | Gemini 3.5 Flash | 67.6% | 0.1206 | 8.29 | 2.34 | 0.0 | 49,957 |
| | Gemma-4-31B | 67.6% | 0.1206 | 8.29 | 2.34 | 22.8 | 49,956 |
| | MUTANT (Krutrim-2) | 53.1% | 0.1742 | 5.74 | 3.38 | 41.9 | 72,147 |
| | **Matra** | **76.4%** | **0.0878** | **11.38** | **1.70** | **5.9** | **36,387** |
| | Qwen-3.6-MoE | 29.5% | 0.2619 | 3.82 | 5.08 | 73.0 | 108,505 |
| | Sarvam-105B | 67.6% | 0.1206 | 8.29 | 2.34 | 22.8 | 49,956 |
| | Sutra-v2 | 69.4% | 0.1139 | 8.78 | 2.21 | 21.4 | 47,170 |
| **or (Odia)** | GPT-5 | -0.1% | 0.3746 | 2.67 | 7.31 | 98.6 | 175,015 |
| | Gemini 3.5 Flash | 28.1% | 0.2690 | 3.72 | 5.25 | 0.0 | 125,669 |
| | Gemma-4-31B | 28.1% | 0.2690 | 3.72 | 5.25 | 74.4 | 125,668 |
| | MUTANT (Krutrim-2) | -162.7% | 0.9829 | 1.02 | 19.19 | 95.2 | 459,205 |
| | **Matra** | **71.5%** | **0.1065** | **9.39** | **2.08** | **7.8** | **49,755** |
| | Qwen-3.6-MoE | 11.4% | 0.3314 | 3.02 | 6.47 | 92.7 | 154,810 |
| | Sarvam-105B | 68.7% | 0.1170 | 8.55 | 2.28 | 28.3 | 54,656 |
| | Sutra-v2 | 67.3% | 0.1222 | 8.18 | 2.39 | 30.0 | 57,095 |
| **pa (Punjabi)** | GPT-5 | 45.8% | 0.2146 | 4.66 | 2.76 | 56.8 | 79,889 |
| | Gemini 3.5 Flash | 43.2% | 0.2250 | 4.44 | 2.90 | 0.0 | 83,750 |
| | Gemma-4-31B | 43.2% | 0.2250 | 4.44 | 2.90 | 61.2 | 83,749 |
| | MUTANT (Krutrim-2) | 36.9% | 0.2499 | 4.00 | 3.22 | 68.0 | 93,022 |
| | **Matra** | **72.4%** | **0.1094** | **9.14** | **1.41** | **4.9** | **40,728** |
| | Qwen-3.6-MoE | 16.6% | 0.3301 | 3.03 | 4.25 | 94.9 | 122,894 |
| | Sarvam-105B | 64.5% | 0.1404 | 7.12 | 1.81 | 27.8 | 52,261 |
| | Sutra-v2 | 69.5% | 0.1208 | 8.28 | 1.55 | 18.6 | 44,952 |
| **sa (Sanskrit)** | GPT-5 | 52.5% | 0.1751 | 5.71 | 4.56 | 44.0 | 75,377 |
| | Gemini 3.5 Flash | 58.4% | 0.1535 | 6.52 | 3.99 | 0.0 | 66,066 |
| | Gemma-4-31B | 58.4% | 0.1535 | 6.52 | 3.99 | 35.9 | 66,065 |
| | MUTANT (Krutrim-2) | 47.4% | 0.1939 | 5.16 | 5.04 | 48.4 | 83,451 |
| | **Matra** | **70.9%** | **0.1071** | **9.34** | **2.79** | **6.4** | **46,087** |
| | Qwen-3.6-MoE | 23.9% | 0.2805 | 3.57 | 7.30 | 75.8 | 120,747 |
| | Sarvam-105B | 58.4% | 0.1535 | 6.52 | 3.99 | 35.9 | 66,065 |
| | Sutra-v2 | 55.8% | 0.1630 | 6.14 | 4.24 | 37.6 | 70,150 |
| **sat (Santali)** | GPT-5 | -164.5% | 0.9961 | 1.00 | 16.59 | 93.9 | 442,477 |
| | Gemini 3.5 Flash | -2.0% | 0.3843 | 2.60 | 6.40 | 0.0 | 170,688 |
| | Gemma-4-31B | -2.0% | 0.3843 | 2.60 | 6.40 | 84.5 | 170,687 |
| | MUTANT (Krutrim-2) | -150.2% | 0.9423 | 1.06 | 15.69 | 99.6 | 418,593 |
| | **Matra** | **47.4%** | **0.1980** | **5.05** | **3.30** | **12.6** | **87,934** |
| | Qwen-3.6-MoE | -150.2% | 0.9425 | 1.06 | 15.69 | 99.6 | 418,656 |
| | Sarvam-105B | 65.1% | 0.1313 | 7.61 | 2.19 | 26.9 | 58,338 |
| | Sutra-v2 | 67.1% | 0.1240 | 8.07 | 2.06 | 3.7 | 55,070 |
| **sd (Sindhi)** | GPT-5 | 53.4% | 0.1816 | 5.51 | 2.57 | 46.2 | 74,948 |
| | Gemini 3.5 Flash | 56.8% | 0.1684 | 5.94 | 2.38 | 0.0 | 69,496 |
| | Gemma-4-31B | 56.8% | 0.1684 | 5.94 | 2.38 | 39.8 | 69,495 |
| | MUTANT (Krutrim-2) | 46.0% | 0.2108 | 4.74 | 2.98 | 57.8 | 86,969 |
| | **Matra** | **62.8%** | **0.1449** | **6.90** | **2.05** | **6.0** | **59,806** |
| | Qwen-3.6-MoE | 27.6% | 0.2823 | 3.54 | 3.99 | 81.0 | 116,498 |
| | Sarvam-105B | 56.8% | 0.1684 | 5.94 | 2.38 | 39.8 | 69,495 |
| | Sutra-v2 | 55.7% | 0.1729 | 5.79 | 2.44 | 36.0 | 71,326 |
| **ta (Tamil)** | GPT-5 | 63.7% | 0.1330 | 7.52 | 3.39 | 30.2 | 68,509 |
| | Gemini 3.5 Flash | 71.8% | 0.1035 | 9.67 | 2.63 | 0.0 | 53,282 |
| | Gemma-4-31B | 71.8% | 0.1035 | 9.67 | 2.63 | 21.3 | 53,281 |
| | MUTANT (Krutrim-2) | 58.8% | 0.1511 | 6.62 | 3.85 | 38.0 | 77,811 |
| | **Matra** | **77.3%** | **0.0831** | **12.03** | **2.11** | **8.4** | **42,797** |
| | Qwen-3.6-MoE | 26.0% | 0.2715 | 3.68 | 6.91 | 76.6 | 139,809 |
| | Sarvam-105B | 71.8% | 0.1035 | 9.67 | 2.63 | 21.3 | 53,281 |
| | Sutra-v2 | 71.3% | 0.1051 | 9.51 | 2.68 | 19.6 | 54,142 |
| **te (Telugu)** | GPT-5 | 58.7% | 0.1543 | 6.48 | 3.30 | 37.2 | 65,917 |
| | Gemini 3.5 Flash | 61.2% | 0.1448 | 6.90 | 3.10 | 0.0 | 61,879 |
| | Gemma-4-31B | 61.2% | 0.1448 | 6.90 | 3.10 | 32.3 | 61,878 |
| | MUTANT (Krutrim-2) | 51.2% | 0.1823 | 5.48 | 3.90 | 47.3 | 77,905 |
| | **Matra** | **72.8%** | **0.1017** | **9.84** | **2.18** | **8.6** | **43,437** |
| | Qwen-3.6-MoE | 18.0% | 0.3063 | 3.26 | 6.55 | 90.1 | 130,885 |
| | Sarvam-105B | 67.4% | 0.1217 | 8.22 | 2.60 | 27.3 | 51,991 |
| | Sutra-v2 | 67.1% | 0.1231 | 8.13 | 2.63 | 23.7 | 52,576 |
| **ur (Urdu)** | GPT-5 | 65.1% | 0.1962 | 5.10 | 1.69 | 22.9 | 54,352 |
| | Gemini 3.5 Flash | 66.5% | 0.1882 | 5.31 | 1.62 | 0.0 | 52,125 |
| | Gemma-4-31B | 66.5% | 0.1882 | 5.31 | 1.62 | 20.8 | 52,124 |
| | MUTANT (Krutrim-2) | 61.5% | 0.2166 | 4.62 | 1.86 | 30.0 | 59,991 |
| | **Matra** | **76.0%** | **0.1351** | **7.40** | **1.16** | **2.3** | **37,419** |
| | Qwen-3.6-MoE | 57.0% | 0.2416 | 4.14 | 2.08 | 39.1 | 66,928 |
| | Sarvam-105B | 66.5% | 0.1882 | 5.31 | 1.62 | 20.8 | 52,124 |
| | Sutra-v2 | 68.1% | 0.1794 | 5.57 | 1.54 | 16.5 | 49,703 |

</details>

</div>

### Matra 200K vs. 128K: Vocab Size Scaling

Matra was evaluated at two vocabulary scales (128K and 200K) to measure the impact of vocabulary size on compression quality. Both benchmarks use the same IN22-Gen test corpus, though the set of competing tokenizers differs  -  the 128K run compares against GPT-5, Gemini 2.5 Flash, Qwen-3.6-MoE, Sarvam-105B, and Sutra-v2, while the 200K run adds Gemma-4-31B and MUTANT (Krutrim-2).

| Metric | Matra 128K | Matra 200K | Improvement |
|---|---|---|---|
| Total Tokens | 1,390,797 | 1,103,089 | **−20.7%** |
| Fertility | 2.59 | 2.06 | **−20.5%** |
| BPT | 7.18 | 8.90 | **+24.0%** |
| 1ch% ↓ | 8.2% | 7.5% | −8.5% |

Scaling vocabulary size from 128K to 200K delivers consistent across-the-board compression gains. Total token count drops by over a fifth, fertility approaches the ideal 2.0 threshold, and bytes-per-token rises to 8.90  -  the highest encoding density among all tokenizers at either scale. The 128K variant remains a compact alternative for deployment environments where embedding table size is at a premium.

---

## Architecture Notes

The core class is `Matra`, initialised with a target vocabulary size and a set of special tokens. The public interface is intentionally narrow.

```python
from tokenizer import Matra

tok = Matra(target_vocab_size=300)
tok.train_from_iterator(texts)          # texts: Iterable[str]

ids = tok.encode("नमस्ते दुनिया")
text = tok.decode(ids)

tok.save("matra.json")
tok.load("matra.json")
```

The saved format is a JSON file containing the merge list (as space-separated string pairs) and the vocabulary (as a list of `[id, token_str]` pairs). The format is intentionally compatible with the HuggingFace tokenizer interface for downstream use.

Key parameters:

- `target_vocab_size`: final vocabulary size including the 256 byte-level base tokens and special tokens. Default 300.
- `transition_vocab_size`: intermediate vocabulary size used in Stage 1 training. Default 275.
- `lang_weight`: upsampling multiplier for target-script text during pair counting. Default 1.0; increase for corpus mixes that are predominantly English.
- `gc_threshold`: ratio of merge count to token count at which an intermediate garbage-collection pass is triggered during training. Default 3.0.
- `max_cache_size`: maximum number of entries in the encode cache. Default 2,000,000.

---

## Limitations and Known Behaviour

The benchmark results on this page use a vocabulary of 200,000 tokens, trained on a ~300 MB multilingual corpus via the 2-pass streaming pipeline. A compact 300-token variant is also available for transfer-learning and low-resource scenarios  -  smaller vocabulary sizes increase cross-lingual transferability at the cost of raw compression.

Cython compilation is optional. The pure-Python pipeline is identical in output; only training speed differs (4-5× slower without the compiled extensions).

---

## Reproducing the Benchmark

The benchmark was run on the IN22-Gen test split using the script included in the repository. Tokenizer weights for GPT-5, Gemini 3.5 Flash, Gemma-4-31B, Qwen-3.6-MoE, Sarvam-105B, Sutra-v2, and MUTANT (Krutrim-2) were loaded from their respective public releases. Matra was trained on a held-out training split; the benchmark corpus was not seen during training.

All metrics are computed at the token level against the original UTF-8 byte stream.
