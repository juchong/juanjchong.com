# juanjchong.com

A personal blog built with [Hugo](https://gohugo.io/) using the [Terminal](https://github.com/panr/hugo-theme-terminal) theme.

## Requirements

- Hugo version 0.87.0 or higher

## Quick Start

1. Clone this repository:
   ```bash
   git clone --recurse-submodules https://github.com/juchong/juanjchong.com.git
   cd juanjchong.com
   ```

2. Start the development server:
   ```bash
   hugo server -t terminal
   ```

3. Open http://localhost:1313 in your browser.

## Deployment

Build the static site:
```bash
hugo
```

The generated site will be in the `public/` directory.

## Structure

- `hugo.yaml` - Site configuration
- `content/posts/` - Blog posts
- `content/about.md` - About page
- `static/images/` - Images and media
- `themes/terminal/` - Terminal theme (git submodule)
