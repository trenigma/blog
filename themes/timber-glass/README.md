# Timber & Glass Hugo Theme

A warm, thoughtfully designed Hugo theme inspired by PNW timber-frame architecture. Natural materials aesthetic with generous whitespace.

**Built by a climber, for readers.** 🏔️🪵✨

---

## Features

✅ **Warm Design Language** - Natural materials palette (wood, stone, glass)  
✅ **Beautiful Typography** - Crimson Pro serif headers + Inter sans body  
✅ **Hero Images** - Mood-setting images for each post  
✅ **Responsive** - Mobile-first, works on all devices  
✅ **Clean Code** - Semantic HTML, organized CSS  
✅ **Fast** - Minimal dependencies, optimized performance  
✅ **Author CTA** - Terracotta button to encourage engagement  

---

## Installation

### Option 1: Clone into themes directory

```bash
cd your-blog-repo
git clone https://github.com/trenigma/timber-glass.git themes/timber-glass
```

### Option 2: Add as Git submodule

```bash
cd your-blog-repo
git submodule add https://github.com/trenigma/timber-glass.git themes/timber-glass
```

### Option 3: Local development (what you're doing now!)

```bash
# Copy the timber-glass-theme folder to your Hugo blog's themes directory
cp -r timber-glass-theme ~/trenigma/blog/themes/timber-glass
```

---

## Configuration

Update your `config.toml`:

```toml
theme = "timber-glass"

[params]
  description = "Infrastructure, climbing, and learning in public"
  author = "Derek Ogletree"
  email = "your-email@example.com"
  
  # Hero section
  heroTitle = "Infrastructure, climbing, and learning in public"
  heroTagline = "SRE/DevOps engineer building in the PNW. Writing about systems, side projects, and the occasional debugging adventure."
  
  # Navigation
  showAboutPage = true
  
  # Custom nav links (optional)
  [[params.customNav]]
    name = "BETA"
    url = "https://beta.trenigma.dev"
  
  # Author CTA at end of posts
  showAuthorCTA = true
  
  [params.authorCTA]
    title = "Thanks for reading"
    text = "If you have questions or want to chat about this post, hit me up. I'm always happy to talk about infrastructure, climbing, or side projects."
    buttonText = "Say Hi →"
    buttonLink = "mailto:your-email@example.com"
  
  # Footer
  footerText = "Built by a climber, for readers. Powered by Hugo + GitHub Pages."
  
  # Syntax highlighting (optional)
  # Options: github-dark, monokai, nord, etc.
  # See: https://highlightjs.org/static/demo/
  highlightStyle = "github-dark"
```

---

## Adding Hero Images to Posts

In your post frontmatter:

```yaml
---
title: "Your Post Title"
date: 2026-03-17
description: "A brief description that appears on post cards"
hero_image: "/images/hero/misty-forest.jpg"
hero_alt: "Misty PNW forest at dawn"
tags: ["infrastructure", "hugo", "migration"]
---
```

**Image specs:**
- Recommended size: 1600x400px
- Format: JPG or WebP
- Keep under 300KB for performance
- Store in `static/images/hero/`

---

## Directory Structure

```
your-blog/
├── config.toml
├── content/
│   └── posts/
│       ├── my-first-post.md
│       └── another-post.md
├── static/
│   └── images/
│       └── hero/
│           ├── forest.jpg
│           └── mountain.jpg
└── themes/
    └── timber-glass/
        ├── layouts/
        ├── static/
        └── theme.toml
```

---

## Customization

### Colors

Edit `themes/timber-glass/static/css/main.css` and modify the CSS variables:

```css
:root {
    /* Wood - change warm tones */
    --wood-lightest: #FAF8F5;
    
    /* Glass - change accent colors */
    --glass-turquoise: #5EC5C1;
    --glass-terracotta: #C67B5C;
}
```

### Typography

The theme uses:
- **Headers:** Crimson Pro (serif) - warm, inviting
- **Body:** Inter (sans-serif) - clear, readable
- **Code:** JetBrains Mono - technical

To change fonts, edit the `@import` statements in `main.css`.

### Layout Widths

```css
:root {
    --content-width: 720px;        /* Reading width */
    --content-width-wide: 1200px;  /* Full layout */
}
```

---

## Testing Locally

```bash
cd ~/trenigma/blog
hugo server -D

# Open http://localhost:1313
```

---

## Deployment (GitHub Pages)

Your existing GitHub Actions workflow should work! The theme is just CSS + HTML templates.

```yaml
# .github/workflows/deploy.yml already configured
# Just commit and push!
```

---

## Design Philosophy

**Timber & Glass** is inspired by PNW timber-frame architecture:

- **Structure is visible** (clear hierarchy, intentional spacing)
- **Natural materials** (warm colors, organic textures)
- **Light and shadow** (depth through elevation)
- **Breathing room** (generous whitespace)
- **Thoughtful details** (subtle animations, typography)
- **Not over-engineered** (clean code, purposeful)

---

## Troubleshooting

### Posts not showing up?

Make sure posts are in `content/posts/` and have `draft: false`.

### Hero images not loading?

Check the path in frontmatter matches the file location in `static/images/hero/`.

### CSS not applying?

Clear browser cache or try incognito mode. Hugo caches aggressively.

### Want different colors?

Edit the CSS variables in `themes/timber-glass/static/css/main.css`.

---

## Credits

**Design & Development:** Tree (Derek Ogletree)  
**Inspiration:** PNW timber-frame homes, Squamish granite, misty forests  
**Color Palette:** Squamish Turquoise (from littlelink!)  

---

## License

MIT - Use it, modify it, make it yours!

---

## Support

Questions? Hit me up:
- Email: your-email@example.com
- GitHub: [@trenigma](https://github.com/trenigma)
- Blog: [blog.trenigma.dev](https://blog.trenigma.dev)

---

**Built with ❤️‍🔥 in the PNW**

🏔️ No hesi! 🏀
