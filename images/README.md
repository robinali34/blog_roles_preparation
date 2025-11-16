# Images Directory

This directory contains images used in blog posts.

## Usage in Posts

To reference an image in a post, use one of these formats:

### Markdown Image Syntax
```markdown
![Alt text]({{ site.baseurl }}/images/filename.png)
```

### HTML Image Tag (for more control)
```html
<img src="{{ site.baseurl }}/images/filename.png" alt="Alt text" width="800" />
```

### With Caption
```html
<figure>
  <img src="{{ site.baseurl }}/images/filename.png" alt="Alt text" />
  <figcaption>Your caption here</figcaption>
</figure>
```

## Notes
- Use descriptive filenames (e.g., `order-matching-architecture.png` instead of `image1.png`)
- Optimize images for web (compress large images)
- Supported formats: PNG, JPG, JPEG, GIF, SVG, WebP

