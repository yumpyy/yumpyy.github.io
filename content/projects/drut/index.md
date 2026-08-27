---
title: "Drut (WIP)"
description: "An edge Speech-to-Speech model targeting sub-150ms latency, multilingual (first of its kind), full-duplex interactivity, and native tool calling."
faq:
  - q: "What is a full-duplex speech-to-speech model?"
    a: "Drut is a full-duplex Speech-to-Speech model that can listen and speak at the same time with natural turn-taking, unlike half-duplex systems that wait for the user to finish before responding."
  - q: "Can Drut run on a phone?"
    a: "Drut is designed as an edge model intended to run on-device (for example on a phone) within a 2GB VRAM budget, with sub-150ms latency, native tool calling, and interruption handling."
  - q: "What makes Drut different from other speech models?"
    a: "Drut targets on-device, full-duplex interaction with turn-taking and tool calling, making it one of the first open-source full-duplex speech systems built for mobile hardware."
image: "drut-logo.png"
category: "Research / GenAI"
category_order: 20
date: 2026-06-16
math: true
toc: true
links:
  github: "#"
  paper: "#"
  huggingface: "#"
draft: false
---

> **Status: Work in Progress.** This is an active design exploration. Specifications and targets are subject to change.

## System Architecture

> Interactive diagram: [open in Excalidraw](https://excalidraw.com/#json=CwPjzUReQDcjjYbpOnAPn,E95GNaDS59Nuvv91phPKxA)

{{< gallery "./arch.png" "System Arch." >}}

The goal is a speech to speech model that runs entirely on a phone. Not a cloud API with a thin client. The model should live on device, listening continuously, ready to respond in under 150 milliseconds. It should call tools, handle interruptions mid sentence, and work across languages. Here is the current design direction.

## The Problem with Speech Models

Most speech AI today works as a pipeline. Speech to text, then text to a language model, then text back to speech. Each step adds latency. Errors compound. And the model cannot hear your tone, your hesitation, or the fact that you were about to say something else but paused to think.

Newer end to end models like Moshi and LFM2-Audio fix some of this, but they are too large for mobile hardware. They eat through VRAM, thermal throttle on NPUs, and assume clean turn taking where no one interrupts anyone.

The target for Drut is a strict 2GB VRAM budget. It should understand when you are thinking out loud versus when you are done speaking. It should call a weather API, wait for the result, and keep the conversation going naturally without awkward silence.

## The Encoder: Hearing Everything

SenseVoice-Small is the proposed audio encoder. It is non-autoregressive, which means it processes audio in parallel instead of token by token. On a mobile CPU it runs at a real time factor of 0.06. It also natively classifies acoustic events and detects emotion, so the model could know not just what you said but how you said it.

But the harder problem is knowing when to speak. Traditional voice activity detection treats any silence as a turn change. If you pause for three seconds to gather your thoughts, a VAD based system cuts you off and starts responding.

JAL-Turn is designed to solve this. It processes audio in 500 millisecond windows with overlap, and looks at two things simultaneously. First, syntactic completeness: did your sentence sound finished? Second, acoustic cues: does your pitch or breathing pattern suggest you are still formulating? A lightweight attention module fuses these signals and outputs a simple hold or shift decision. Hold means keep listening. Shift means it is the model's turn.

## The Backbone: Qwen 3.5 with DeltaNet

The language model under consideration is Qwen 3.5 at 2B or 4B parameters depending on the target device. These models ship with hybrid DeltaNet v1 layers, which use linear recurrence instead of full attention. This provides infinite context without quadratic memory growth. They also include native Multi-Token Prediction heads, so decoding should be faster without needing a separate draft model.

The DeltaNet gating mechanism is particularly interesting here. It lets the model surgically erase information from its compressed memory state. Conversational filler like "um" and "let me see" can be aggressively cleared. JSON tool returns stay preserved. This selective memory management should keep the model responsive over long conversations.

Four additional optimizations are planned to keep everything within 2GB:

Block contiguous KV allocation aligned to the 500ms audio chunks, so the NPU reads memory sequentially without cache misses. TurboQuant, which applies a Walsh-Hadamard transform followed by a 3 bit Lloyd-Max codebook, targeting roughly 3.5 bits per parameter with negligible quality loss. Attention sinks in the sparse standard attention layers that Qwen 3.5 retains alongside DeltaNet, pinning the system prompt and tool call tokens so they never fade from recurrent memory. And state flushing: when JAL-Turn detects a user interruption, the recurrent state rolls back to a snapshot taken right before the model started speaking, instantly clearing its own voice from working memory.

## The Output: FlexiCodec

Generating speech autoregressively on a phone is too expensive. The current plan uses FlexiCodec, a dynamic frame rate codec that operates between 6.25 and 8.3 Hz. The LLM would only need to predict 6 to 8 structural tokens per second. An on device non-autoregressive upscaler would expand these into full 24 kHz audio.

FlexiCodec's semantic layer would be fine tuned on the Emilia dataset to ensure multilingual prosody is preserved. The goal is for the model to handle code switching and non English languages naturally because the codec can understand the acoustic boundaries of each language.

## Tool Calling and the Muttering Trick

When the model decides to call a tool, it emits a special token. A lightweight C++ orchestrator intercepts this, fires the API request, and keeps the model running. But the model does not sit idle. It generates filler audio. A casual "let me check on that." If the API takes longer than a few seconds, it transitions to ambient room tone. The user never hears dead silence.

The longer stretch of filler is called contextual musing. The orchestrator extracts key arguments from the tool call, splices them with cached acoustic tokens of natural stalling sounds, and the model audibly mutters to itself. "Let's see. Flights to Tokyo." The moment the API returns, the JSON payload is injected through decoupled Rotary Position Embeddings. The text tokens operate on their own positional index, so the acoustic timeline is not disrupted. The model picks up smoothly and delivers the answer.

## Training

The training plan has three stages on H100 clusters. Stage one freezes the encoder and language model and trains only the projector to align acoustic features with text representations using Mozilla Common Voice. Stage two unfreezes everything and trains on synthetic interleaved data: audio, tool call, filler, wait, silence, tool result, answer. The model learns the rhythm of tool augmented conversation.

Stage three uses GRPO reinforcement learning with a dual reward. A timing reward penalizes the model for speaking during mid utterance pauses or failing to stop on interruption. A semantic judge preserves instruction following and JSON formatting quality. Pure supervised fine tuning tends to produce models that are too polite and clumsy with turn taking. The RL stage is intended to make the interaction feel natural.

## Expected Targets

The design targets 2GB of VRAM, with Apple Silicon and Snapdragon NPUs as primary deployment targets. The goal is a model that handles interruptions, calls tools, mutters while thinking, and does not cut you off when you pause to find the right word.
