---
updates:
  - date: 2026-06-15
    time: "3:45 PM"
    text: "Benchmarked against GPT-5, Gemini 3.5 Flash, and Gemma-4-31B on IN22-Gen. Matra scores lowest fertility (2.06) and highest BPT (8.90) across all 23 languages."
    images:
      - "matra.png"
  - date: 2026-06-10
    time: "11:20 AM"
    text: "Implemented 2-pass streaming architecture. Peak RAM drops from ~75 GB to ~5 GB on 300 MB corpus. Training now fits on a 16 GB machine."
  - date: 2026-06-05
    time: "9:00 PM"
    text: "Added Cython-accelerated hot paths for the inner merge loop. Pure-Python fallback still intact. 4x wall-clock speedup when compiled."
  - date: 2026-05-28
    time: "4:15 PM"
    text: "Refactored encode path to O(1) byte-to-token mapping with 256-entry tuple and 2M-entry LRU cache."
  - date: 2026-05-20
    time: "1:30 PM"
    text: "Built RAM Shield with singleton-pair purge and early stopping. Stage 2 now hard-capped at ~2.5 GB regardless of corpus size."
---
