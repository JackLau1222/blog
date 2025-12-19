# My Personal Blog

This is a personal blog built using [Docusaurus](https://docusaurus.io/), a modern static website generator.

## Installation

Install dependencies:

```bash
npm install
```

## Local Development

Start the development server:

```bash
npm run start
```

This command starts a local development server and opens up a browser window at `http://localhost:3000`. Most changes are reflected live without having to restart the server.

## Writing Blog Posts

1. Create a new markdown file in the `blog/` directory
2. Name it with the format: `YYYY-MM-DD-post-title.md`
3. Add front matter at the top:

```markdown
---
slug: post-slug
title: Your Post Title
authors: [jacklau]
tags: [tag1, tag2]
---

Your post content here...

<!--truncate-->

More content after the "Read more" link...
```

## Adding Custom Pages

You can add custom pages (like About, Projects, etc.) in two ways:

### Option 1: Markdown Pages (Recommended)

Create `.md` files in `src/pages/`:

```markdown
---
title: Page Title
description: Page description
---

# Your Content Here

Write your content in Markdown format.
```

Example: `src/pages/about.md` → accessible at `/about`

### Option 2: React Pages

Create `.tsx` files in `src/pages/` for more complex pages with custom components.

Example: `src/pages/projects.tsx` → accessible at `/projects`

## Build

Build the static site:

```bash
npm run build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

## Project Structure

```
blog/
├── blog/                      # 📝 Blog posts (markdown files)
│   ├── 2024-12-19-welcome.md  # Example: YYYY-MM-DD-title.md
│   └── authors.yml            # Author information
├── src/
│   ├── css/                   # 🎨 Custom CSS (not modified)
│   ├── pages/                 # 📄 Custom pages
│   │   ├── index.tsx          # Home page (React component)
│   │   └── about.md           # About page (Markdown)
│   └── components/            # React components
├── static/                    # 🖼️ Static assets
│   └── img/                   # Images, icons, etc.
├── docusaurus.config.ts       # ⚙️ Site configuration
└── package.json               # 📦 Dependencies
```

## Navigation Structure

- **Home** (`/`) - Landing page with author intro
- **Blog** (`/blog`) - All blog posts
- **About** (`/about`) - About page (example custom page)
- All pages are accessible via the top navigation bar

## Customization

- Edit `blog/authors.yml` to update author information
- Edit `docusaurus.config.ts` to change site title, tagline, and other settings
- Add your blog posts to the `blog/` directory

Happy blogging! 🎉
