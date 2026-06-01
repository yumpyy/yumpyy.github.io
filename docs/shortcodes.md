# Custom Shortcodes

## Image (`img`)

Resizes and renders an image from the page bundle using Hugo's image processing.

```
{{< img src="photo.jpg" w="400" alt="My photo" >}}
{{< img src="photo.jpg" w="800" class="float-right" alt="" >}}
{{< img src="photo.jpg" w="600" alt="Sunset" title="A beautiful sunset" >}}
```

### Parameters

| Param | Required | Description |
|-------|----------|-------------|
| `src` | yes | Image filename in the page bundle |
| `w` | yes | Target width in pixels (aspect ratio preserved) |
| `alt` | no | Alt text |
| `title` | no | Rendered as a caption below the image |
| `class` | no | CSS class(es) for styling |

### How it works

- Looks up the image via `.Resources.Get` from the page bundle
- Resizes to `w` width (maintains aspect ratio)
- Outputs `<img>` with `width`/`height` attributes (prevents layout shift)
- Caches processed images for fast rebuilds

---

# Projects Page

## Overview

The projects page (`/projects/`) is a custom section layout that groups project cards by category. It uses a dedicated `list.html` for the overview and `single.html` for individual project detail pages.

## File Structure

```
content/projects/_index.md              # section index
content/projects/<project-slug>.md    # one file per project
layouts/projects/list.html            # projects listing page
layouts/projects/single.html          # individual project detail page
assets/css/projects.css               # project-specific styles
static/images/projects/               # project images
```

## Project Front Matter Schema

```yaml
---
title: "Project Title"
description: "Short description shown on the card"
category: "Research"             # any string — used as the group label
category_order: 10               # optional, controls position vs other categories
image: "/images/projects/image.jpg"
date: 2026-01-01
links:
  github: "https://github.com/..."
  huggingface: "https://huggingface.co/..."   # optional
  misc: "https://example.com"                # optional
draft: false
---
```

The body below the front matter is the full project explanation, rendered on the single page.

## Category Order

Projects are grouped by `category`. The `category_order` value controls where each group appears. Projects sharing the same `category` are grouped together; the group takes the lowest `category_order` among them. Groups with no explicit order sort to the bottom.

Within a group, projects are listed by date (newest first).

## Design

- **Card layout**: One vertical card — image on top (16:9), unified text area below. The text area contains the project title and description, with a visible horizontal line at the bottom separating a small links row. Links are pinned to the bottom of the text area (`margin-top: auto`) so all cards in a row align uniformly.
- **Category label**: Monospace uppercase box with a flush horizontal extending line.
- **Hover effect**: Border turns `var(--primary)` with a subtle 2px lift. No shadows.
- **Missing image**: Diagonal hatching placeholder (`repeating-linear-gradient`).
- **Icon links**: GitHub (octocat), Hugging Face (smiley), Misc (clipboard). Clicking an icon link does not navigate to the project page (`event.stopPropagation()`).
- **Description clamping**: Limited to 3 lines for uniform card heights (`-webkit-line-clamp`).
- **Single page**: Breadcrumb back link, project image, external links, and rendered markdown content.
- **Styling**: All colors, fonts, and borders use the theme's CSS variables (`--bg-color`, `--bg-variant`, `--primary`, `--font-color`, `Literata`, `Red Hat Mono`) so it respects light/dark mode.

## Adding a New Project

1. Add the project image to `static/images/projects/` (or reference an external URL).
2. Create `content/projects/<slug>.md` with the front matter schema above.
3. Set `category` to any label (e.g. `"Research"`, `"Weekend Projects"`, `"Generative AI"`).
4. Set `category_order` to control where the category group appears (lower = earlier).
5. Write the project body in markdown below the front matter.
6. Run `hugo` to build.
