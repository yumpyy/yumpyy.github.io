---
title: "Desi Maximalism"
description: "Text-to-Image LoRA made from a hand-curated dataset to flawlessly generate the vibrant, nostalgic retro aesthetics of Desi Indian culture that frontier models fail to reproduce."
category: "Generative AI"
category_order: 30
image: "drama-queen.png"
date: 2026-06-01
links:
  huggingface: "https://huggingface.co/yenupam/desi-max"
draft: false
---

## Project Overview

The Desi Maximalism LoRA is a specialized text-to-image style transfer model engineered to recreate the rich, dense, and vibrant aesthetic of mid-century Indian commercial art. It acts as a cultural bridge for generative AI, accurately reproducing the nostalgic visual language of 1940s–1985s South Asian matchbox labels, product packaging, magazine ads, and hand-painted film posters.

## Sample Generations

{{< gallery "./bournvita.png" "Bournvita retro tin packaging" "./google-2.png" "Google India ad" "./pulse.png" "Pulse candy vintage label" "./bharat-ai.png" "Bharat AI vintage poster" "./lenskart.png" "Lenskart" "./maggi.png" "Maggi noodles" "./gen-2.png" "Generation #2" "./drama-queen.png" "Drama Queen" >}}

## Model Specifications

| Feature | Specification |
| --- | --- |
| **Base Model** | Qwen/Qwen-Image-2512 |
| **Fine-Tuning Method** | LoRA (Low-Rank Adaptation) |
| **Task** | Text-to-image • Style transfer |
| **Trigger Word** | `desi-max` |
| **License** | Apache 2.0 |

## Dataset & Curation Strategy

The model is trained on a highly focused dataset of 78 meticulously handpicked images of vintage South Asian commercial print. Each image was manually selected to represent a distinct visual sub-pattern. This strict curation keeps the dataset tight and entirely avoids the "style collapse" common in broader, noisier generative models.

## The Aesthetic Target

To accurately capture the "Desi" look, the LoRA is explicitly optimized to reproduce the following analog and structural characteristics:

- **Color & Contrast:** Bold, flat color blocking and high-contrast, saturated palettes.
- **Structural Framing:** Decorative borders, concentric rules, starburst layouts, and ornamental cartouche framing.
- **Print Textures:** The distinct feel of mid-century analog printing, specifically halftone dot screens and offset print grain.
- **Typography & Illustration:** Dense, multi-scale typographic hierarchies paired with hand-painted illustration shading and exaggerated perspectives.

## Usage & Prompt Engineering

To activate the style transfer, prepend the trigger word **`desi-max`** to your prompt. *(Adding the term "lora" in the prompt text can optionally yield better stylistic adherence depending on your inference setup).*

**Example Prompt:**

> `desi-max, vintage Indian matchbox label, GAJRAJ AUTO in large red lettering, blue starburst, illustrated autorickshaw in green and pink, yellow background, bold halftone print texture, mid-century South Asian commercial design`

### Sample Generation Targets

The model is highly capable of generating specific cultural and commercial pastiches, such as:

- Bournvita retro tin packaging with ornamental cartouches.
- Google retro Indian ads featuring flat illustrations and bold lettering.
- Pulse candy vintage labels with stacked badges and hand-painted shading.
- Bharat AI vintage posters featuring starburst layouts and offset print textures.

## Current Limitations

- **Typographic Fidelity:** While the model accurately structures typographic hierarchy, exact spelling accuracy for the Devanagari script is bounded by the base model's inherent multilingual capabilities.
- **Stylistic Boundaries:** The LoRA is strictly optimized for flat, illustrated commercial aesthetics and is not intended for photorealism.
