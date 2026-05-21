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
