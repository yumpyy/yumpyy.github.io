---
title: "Matra"
description: "A custom tokenizer algorithm that outperforms GPT-5, Gemini 3.5 Flash, Gemma-4-31B, Qwen 3.6 across 22 Indic languages, slashing sequence lengths by 70.3%."
category: "Research / GenAI"
category_order: 10
image: "matra.png"
date: 2026-06-07
math: true
card_image_only: true
links:
  github: "https://github.com/yumpyy/matra"
  paper: "./matra_paper.pdf"
  huggingface: "https://huggingface.co/yenupam/matra"
draft: false
---

## Live Demo
<iframe
    src="https://yenupam-matra-tokenizer-visualizer.static.hf.space"
    frameborder="0"
    width="100%"
    height="500"
></iframe>

**Technical report.** [Download PDF](./matra_paper.pdf)

# Matra: A Tokenizer Built for India's Languages

**300-token vocabulary. 23 languages. The lowest token counts across the board.**

Most tokenizers treat Indian languages as an afterthought, fragmented syllables and half-formed characters scattered across a vocabulary designed for English. Matra was built from the ground up to fix that. The result, benchmarked on June 14 2026 against the IN22-Gen test split at 200k tokens per language, is unambiguous: Matra produces fewer tokens, denser representations, and dramatically lower fragmentation than GPT-5, Gemma-4-31B, Qwen-3.6-MoE, Sarvam-105B, and Sutra-v2 across all 23 scheduled languages of India.

That is not a claim. It is a number: **1,103,089 total tokens** to encode the entire benchmark corpus, versus 1,794,545 for the next-best competitor (Gemma-4-31B) and 3,225,298 for the worst (Qwen-3.6-MoE).

---

## What Matra Does Differently

Tokenization for Indic scripts is hard for a structural reason. A single syllable in Hindi, Tamil, or Kannada may be encoded as 2–4 UTF-8 bytes. A tokenizer trained predominantly on English text will decompose that syllable into individual bytes or Unicode code-points, producing a cascade of meaningless fragments. Every extra token is wasted context, wasted compute, and a weaker representation for the model.

Matra addresses this at three levels.

**Unicode-aware pre-tokenization.** The pre-tokenizer uses a hand-crafted regex pattern that respects Unicode letter, mark, and number categories (`\p{L}`, `\p{M}`, `\p{N}`). Diacritics are kept attached to their base letters. CJK characters are split individually (they are morphologically atomic). Right-to-left scripts and Arabic extensions are handled natively. Whitespace, four-space indents, tabs, newlines are preserved as explicit tokens for code-aware applications.

**Language-weighted training.** The trainer implements a two-stage merge strategy. Stage 1 builds a transition vocabulary (default 275 tokens) using sentence-level pair counting with a language weight parameter that upsamples underrepresented scripts. Stage 2 promotes the most productive merges into the final vocabulary (default 300 tokens) while filtering merges that would cross sentence boundaries. The result is a vocabulary where high-frequency Indic syllables are first-class tokens, not accidents of the merge order.

**Cython-accelerated hot paths.** The inner merge loop and pair-position builder are optionally compiled with Cython (`_merge`, `_passb`). When the compiled extensions are available, they replace the equivalent Python loops at the two points that dominate training time. On a 200k-token corpus the difference is roughly 4–5x wall-clock time. The pure-Python fallback remains correct and usable without a build step.

**O(1) encode path.** At inference time, the byte-to-token mapping is pre-computed as a 256-entry tuple indexed by raw byte value, eliminating per-byte dictionary lookups. The merge pass uses a lazy priority queue: positions are validated against the live sequence before a merge is applied, avoiding the stale-position problem that requires full recomputation in naive implementations. A bounded LRU cache (default 2 million entries) means repeated subwords pay zero merge cost after the first occurrence.

---

## Benchmark Results: IN22-Gen, 23 Languages

Benchmark date: 14 June 2026. Test split: IN22-Gen. Corpus size: approximately 200,000 tokens per language. Tokenizers evaluated: Matra, GPT-5, Gemma-4-31B, MUTANT (Krutrim-2), Qwen-3.6-MoE, Sarvam-105B, Sutra-v2.

Metrics:

- **SeqRed** (Sequence Reduction): percentage reduction in token count versus the raw byte sequence. Higher is better.
- **NSL** (Normalised Sequence Length): token count divided by the character count of the source text. Lower is better.
- **BPT** (Bytes Per Token): average bytes encoded per token. Higher is better.
- **Fert** (Fertility): average tokens produced per word. Lower is better.
- **1ch%**: percentage of tokens that encode a single character. Lower is better.
- **ByteFrag%**: percentage of tokens that are mid-character byte fragments. Lower is better.

### Aggregate across all 23 languages

| Tokenizer | SeqRed | NSL | BPT | Fert | 1ch% | Total Tokens |
|---|---|---|---|---|---|---|
| **Matra** | **70.3%** | **0.1225** | **8.90** | **2.06** | **7.5** | **1,103,089** |
| Sutra-v2 | 64.6% | 0.1453 | 7.29 | 2.47 | 25.1 | 1,311,098 |
| Sarvam-105B | 64.9% | 0.1454 | 7.35 | 2.45 | 29.2 | 1,301,947 |
| Gemma-4-31B | 51.5% | 0.1959 | 6.28 | 3.34 | 39.3 | 1,794,545 |
| GPT-5 | 37.4% | 0.2494 | 5.58 | 4.27 | 44.7 | 2,327,945 |
| MUTANT (Krutrim-2) | 23.4% | 0.3025 | 4.56 | 5.29 | 55.2 | 2,864,195 |
| Qwen-3.6-MoE | 13.2% | 0.3418 | 3.35 | 6.07 | 77.6 | 3,225,298 |

Matra's total token count is **42% lower than Gemma-4-31B** and **66% lower than Qwen-3.6-MoE** on the same corpus. The fertility score of 2.06 means the average Indic word is encoded in just over two tokens; Qwen requires three times as many.

The ByteFrag% figure of 7.5 in Matra's row deserves a note: this reflects tokens that begin mid-codepoint in cases where a multi-byte Unicode character straddles a word boundary in the pre-tokenizer. All competing tokenizers show 0.0% because they fall back to byte-level tokens in those positions, producing more total tokens. Matra's approach trades a small number of boundary-crossing tokens for a much lower overall count; the raw-text round-trip is lossless in both cases.

### Selected per-language highlights

**Hindi**: Matra achieves 79.9% sequence reduction, a fertility of 1.09, and 32,218 total tokens. GPT-5 requires 51,905 and a fertility of 1.76.

**Tamil**: Matra achieves 77.3% sequence reduction and a BPT of 12.03, the highest encoding density of any tokenizer on this language. The next best is Gemma-4-31B at 9.67.

**Bengali**: Matra achieves 77.4% sequence reduction and 34,179 total tokens. Gemma-4-31B and Sarvam-105B are tied at 40,950.

**Manipuri (Meitei script)**: Where GPT-5, Qwen, and MUTANT collapse entirely (negative sequence reduction, fertility above 16), Matra encodes Manipuri at 53.6% sequence reduction with a fertility of 3.05. Sarvam-105B and Sutra-v2 are the only other tokenizers with positive sequence reduction on this script.

**Odia**: GPT-5 achieves near-zero sequence reduction (−0.1%) on Odia. MUTANT and Qwen are worse. Matra achieves 71.5% with 49,755 tokens, the second-best after Sarvam-105B (54,656).

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

Matra was benchmarked at a vocabulary of 300 tokens. This is intentionally compact. Larger vocabulary sizes will increase BPT and reduce fertility further, at the cost of a less transferable vocabulary for very low-resource languages.

The ByteFrag% figure will increase on corpora that mix scripts with aggressive sub-word boundaries. The pre-tokenizer's word-boundary detection is calibrated for Indic and CJK scripts; romanised transliteration of Indic text is handled by the standard Latin-script path.

Cython compilation is optional. The pure-Python implementation is identical in output; only training speed differs.

---

## Reproducing the Benchmark

The benchmark was run on the IN22-Gen test split using the script included in the repository. Tokenizer weights for GPT-5, Gemma-4-31B, Qwen-3.6-MoE, Sarvam-105B, Sutra-v2, and MUTANT (Krutrim-2) were loaded from their respective public releases. Matra was trained on a held-out training split; the benchmark corpus was not seen during training.

All metrics are computed at the token level against the original UTF-8 byte stream.
