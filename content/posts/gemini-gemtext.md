---
title: "Gemini / Gemtext Notes"
date: 2024-06-01
lastmod: 2024-06-01
draft: false
tags: ["notes", "gemini"]
description: "Quick reference for Gemtext markup format used in the Gemini protocol."
---

**Gemtext**: Gemtext is the lightweight markup format used in the Gemini protocol, an alternative to HTTP for simple document publishing. It uses a minimal syntax where lines starting with `=>` are links, `#` are headings, `*` are list items, `>` are quotes, and triple backticks toggle preformatted text. Each line is a standalone entity with no inline formatting, making it simpler than Markdown but more structured than plain text.

## Text Lines

- Each individual line in a Gemtext document is a stand-alone entity. Hence, a separate line break syntax is not needed.
- Text lines which don't fit in client's display get wrapped automatically.

## Link Lines

Lines starting from `=>` are considered link lines.

Syntax:
```
=> url/link   link-name
```

link-name is optional.

eg. `=> gemini://example.org/  Example Website`

## Preformatted Toggle Lines

Lines starting from backticks (```) are preformatted toggle lines. They switch the parser from normal mode to pre-formatted mode. The text after the backticks is used as alt text.

## Heading Lines

Lines starting from `#` are heading lines.

- `#`   = heading
- `##`  = sub-heading
- `###` = sub-sub-heading

eg. `# This is an example heading`

## List Items

Lines starting from `*` are list items.

```
* Shopping List
  * Banana
  * Apple
```

## Quote Lines

Lines starting from `>` are quote lines.

eg. `> This is a quote.`

## Related Reading

- [Connman systemd-resolved](/posts/connman-systemd-resolved/) — Network configuration on Arch Linux
- [C++ Notes](/posts/cpp-notes/) — Programming fundamentals reference
