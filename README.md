# juanjchong.com

A personal portfolio and blog built with [Hugo](https://gohugo.io/) using the [Hugo Profile](https://github.com/gurusabarish/hugo-profile) theme.

## Requirements

- Hugo version 0.87.0 or higher

## Quick Start

1. Clone this repository:
   ```bash
   git clone https://github.com/juchong/juanjchong.com.git
   cd juanjchong.com
   ```

2. Add the Hugo Profile theme as a submodule:
   ```bash
   git submodule add https://github.com/gurusabarish/hugo-profile.git themes/hugo-profile
   ```

3. Start the development server:
   ```bash
   hugo server
   ```

4. Open http://localhost:1313 in your browser.

## Deployment

Build the static site:
```bash
hugo
```

The generated site will be in the `public/` directory.

## Structure

- `hugo.yaml` - Site configuration
- `content/blogs/` - Blog posts
- `static/images/` - Images and media
- `themes/hugo-profile/` - Hugo Profile theme (git submodule)
