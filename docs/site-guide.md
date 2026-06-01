# Content Guide

This guide covers how to add and manage content on yenupam.com.

---

## Blog Posts

### Creating a Post

```sh
hugo new content posts/my-post-title.md
```

This creates `content/posts/my-post-title.md` with auto-generated front matter. Open it in any text editor.

### Front Matter

The block between the `---` lines at the top of the file is the front matter. Here is what each field does:

| Field | Required | What it does |
|-------|----------|--------------|
| `title` | yes | Post title, displayed at the top of the page |
| `date` | yes | Publication date. Format: `2026-06-01` or `2026-06-01T14:30:00+05:30` |
| `draft` | no | Set to `true` to hide the post from the live site |
| `description` | no | Short summary shown on the /posts/ list page and in link previews |
| `tags` | no | List of tags, e.g. `[linux, android]`. Each tag gets its own archive page |
| `toc` | no | Set to `true` to show a table of contents on the post page |
| `math` | no | Set to `true` to enable LaTeX math rendering (KaTeX) |

Example:

```yaml
---
title: "My Post Title"
date: 2026-06-01
draft: false
description: "A short summary of the post"
tags:
    - linux
    - android
---
```

Write the post content in Markdown below the front matter.

### Adding Images to a Post

For images, use a **page bundle** — make a folder instead of a single file:

```
content/posts/my-post/
  index.md          ← your post file
  photo.jpg         ← image file alongside it
  diagram.png
```

Create the folder manually (or rename the `.md` file to `index.md` inside a folder). Reference images with the `img` shortcode:

```markdown
{{< img src="./photo.jpg" w="600" alt="Description" >}}
{{< img src="./diagram.png" w="800" title="Figure 1: Architecture" >}}
```

Parameters: `src` (filename), `w` (width in pixels, aspect ratio preserved), `alt` (alt text), `title` (shown as a caption below the image), `class` (optional CSS class).

### Previewing a Draft

Run `hugo server` to see the site locally at `http://localhost:1313`. Drafts (`draft: true`) are shown by default in local preview. To build the production site (which excludes drafts):

```sh
hugo
```

### Publishing

Once your post is ready, set `draft: false`, commit the file, and push to the `stable` branch:

```sh
git add content/posts/my-post-title.md
git commit -m "add post: My Post Title"
git push origin stable
```

The site auto-deploys to yenupam.com within a couple of minutes.

---

## Projects

### Creating a Project

Projects can be organised in two ways:

**Single file** (simple, one file per project):
```
content/projects/my-project.md
```

**Page bundle** (recommended if you have images):
```
content/projects/my-project/
  index.md               ← project markdown
  architecture.png       ← images live right next to it
  screenshot.jpg
```

Create either with:

```sh
hugo new content projects/my-project.md          # single file
mkdir -p content/projects/my-project && touch content/projects/my-project/index.md   # page bundle
```

### Front Matter

| Field | Required | What it does |
|-------|----------|--------------|
| `title` | yes | Project name |
| `description` | yes | One-line summary — this is what shows on the project card |
| `category` | yes | Group label, e.g. `"Research"`, `"Weekend Projects"`, `"Generative AI"`. Any text is valid |
| `category_order` | no | A number that controls where this group appears on the page. Lower = earlier. Groups without this value sort to the bottom |
| `image` | no | Card cover image. For a page bundle use the filename, e.g. `"photo.jpg"`. For a single file use a path like `"/images/projects/photo.jpg"`. If omitted, shows a placeholder |
| `date` | yes | Publication date |
| `draft` | no | Set to `true` to hide from the live site |
| `math` | no | Set to `true` to enable LaTeX math rendering (KaTeX) on the project page |
| `links.github` | no | GitHub repository URL — shows a GitHub icon on the card |
| `links.huggingface` | no | Hugging Face model/dataset URL — shows a Hugging Face icon |
| `links.paper` | no | Research paper / PDF / ArXiv URL — shows a document icon |
| `links.misc` | no | Any other URL (blog post, demo, etc.) — shows an external link icon |

Example (page bundle):

```yaml
---
title: "Matra"
description: "A lightweight prosody-aware speech synthesis model for Bengali."
category: "Research"
category_order: 10
image: "architecture.png"
date: 2026-06-01
links:
  github: "https://github.com/yumpyy/matra"
  paper: "https://arxiv.org/abs/..."
  huggingface: "https://huggingface.co/yenupam/matra"
draft: false
---
```

The body below the front matter is the full project write-up, rendered when someone clicks through to the project detail page.

### How Categories Work

- Projects sharing the same `category` value are grouped together under one heading.
- Category groups appear in order of `category_order` (lowest first). If two projects share a category but have different `category_order` values, the group uses the lowest one.
- Within a group, projects are listed newest first.
- Categories with no `category_order` default to 999 and sort to the bottom.

Example: to add a new category that appears between Research (order 10) and Weekend Projects (order 20), use `category_order: 15`.

### Adding a Project Image

**Card cover image** — the image that shows on the project card:

- **Page bundle**: place the image inside the project folder and set `image: "filename.jpg"` in front matter.
- **Single file**: place the image in `static/images/projects/` and set `image: "/images/projects/filename.jpg"`.

The cover image is shown at 16:9 on the card and full-width on the detail page. If omitted, the card shows a diagonal hatch placeholder.

### Images in the Project Body

To include images inside the project write-up (the Markdown body below the front matter):

**Page bundle** — images are in the same folder as `index.md`. Use the `img` shortcode:
```markdown
Here is the model architecture:

{{< img src="./architecture.png" w="600" alt="Matra architecture" >}}
```

**Single file** — use regular Markdown with the path starting from `/`:
```markdown
Here is a screenshot:

![](/images/projects/matra/screenshot.jpg)
```

The `img` shortcode resizes images and preserves aspect ratio. Parameters: `src` (filename), `w` (target width), `alt`, `title` (caption below the image).

### Link Icons

Each project card shows a small row of icons at the bottom. The available icons map to front matter fields as follows:

| Front matter field | Icon |
|---|---|
| `links.github` | GitHub logo |
| `links.huggingface` | Hugging Face logo |
| `links.paper` | Document icon (paper, PDF, ArXiv) |
| `links.misc` | External link icon |

Only the link fields you include in front matter will show icons. If a project has no links, the row is hidden.

### LaTeX Math

To include mathematical equations in any blog post or project page, add `math: true` to the front matter. This loads KaTeX, a fast JavaScript math renderer.

Use standard LaTeX delimiters in your Markdown:

```markdown
Inline math: $a^2 + b^2 = c^2$

Display math (centered, larger):

$$\int_{0}^{1} x^2 \, dx = \frac{1}{3}$$
```

KaTeX renders equations instantly in the browser — no server-side processing needed.

### Publishing

Same as blog posts — commit and push to `stable`:

```sh
git add content/projects/my-project.md
git commit -m "add project: My Project"
git push origin stable
```

---

## Common Tasks

### Creating a New Category

Just use a new `category` value in any project's front matter. It will automatically appear as a new group on the /projects/ page. Set `category_order` to control where it sits relative to existing groups.

### Reordering Categories

Change `category_order` values. The site rebuilds with the new order automatically — no template changes needed.

### Moving a Project Between Categories

Change the `category` value in the project's front matter. It moves to that group on the next build.

### Hiding a Page Before It's Ready

Set `draft: true` in the front matter. Draft pages are invisible on the live site but still show in local preview (`hugo server`).

---

## SEO

The site handles the following automatically — you don't need to configure anything:

| Feature | What it does |
|---------|-------------|
| **Sitemap** | Auto-generated at `/sitemap.xml`, submitted to search engines |
| **robots.txt** | Allows all crawlers, points to sitemap |
| **Canonical URL** | Every page has `<link rel="canonical">` to prevent duplicate content issues |
| **Meta description** | Pulled from each page's `description` front matter — used in search result snippets |
| **Open Graph** | Rich previews when shared on social media (Facebook, Discord, etc.) |
| **Twitter Cards** | Rich previews when shared on X/Twitter |
| **JSON-LD Schema** | Structured data for search engines (Person + Article schema) |
| **RSS feed** | Auto-generated at `/index.xml` with `<link>` in page head for autodiscovery |

### What You Should Do

**Write good descriptions.** The `description` field in front matter is the most important SEO lever you control. It appears in:
- Search result snippets (Google)
- Open Graph previews (social media)
- Twitter Cards

Keep descriptions 120–155 characters, unique per page, and accurately summarize the content.

**Submit to Google Search Console** (one-time setup):
1. Go to https://search.google.com/search-console
2. Add your site (`yenupam.com`)
3. Verify ownership (DNS TXT record or HTML file in `static/`)
4. Submit the sitemap URL: `https://yenupam.com/sitemap.xml`
5. Also register at https://www.bing.com/webmasters for Bing indexing

That is all that's needed. The technical SEO is already handled — focus on writing good content and descriptions.
