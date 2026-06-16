---
updates:
  - date: 2026-06-16
    time: "10:00 AM"
    text: "system architecture finalized. target: sub-150ms latency, 2gb vram, multilingual, full-duplex, native tool calling."
    image: "arch.png"
  - date: 2026-06-12
    time: "5:30 PM"
    text: "selected sensevoice-small as audio encoder and jal-turn for turn-taking instead of whisper and fastconformer. non-autoregressive, 0.06 rtf on mobile cpu and emotional detection as well."
  - date: 2026-06-08
    time: "2:45 PM"
    text: "settled on qwen 3.5 2b/4b with deltanet v1 layers. infinite context, linear recurrence, multi-token prediction heads."
  - date: 2026-06-02
    time: "11:15 AM"
    text: "designed flexicodec output layer. 6.25-8.3 hz dynamic frame rate. non-autoregressive upscaler to 24 khz."
---
