# HieuPN Astro Blog

This repository powers a personal blog and profile site built with Astro 5, MDX, Tailwind CSS 4, and client-side search. It combines editorial content, author/category/tag archives, a series index, an about page, and a contact page into a statically generated site.

The current production site URL configured in the project is `https://hieupn.com`.

## Stack

- Astro 5
- MDX via `@astrojs/mdx`
- Tailwind CSS 4 via `@tailwindcss/vite`
- `astro-icon` with Material Design Icons
- Fuse.js for client-side search
- `@astrojs/rss` for RSS generation
- `@astrojs/sitemap` for sitemap generation

## Features

- Homepage with hero section and latest post cards
- Blog index with pagination
- Category, tag, and author archive pages with pagination
- Series index page plus in-post series navigation
- Post detail pages with reading progress, breadcrumbs, table of contents, related posts, and share links
- Client-side search powered by `/search.json`
- SEO metadata with canonical URLs, Open Graph, Twitter cards, and JSON-LD
- RSS feed at `/rss.xml`
- Sitemap generation with search and paginated pages filtered out
- Dark mode toggle with localStorage persistence and flash-of-incorrect-theme prevention
- Reusable MDX widgets available without per-file imports

## Quick Start

### Prerequisites

- A recent Node.js LTS release. Node.js 20 LTS is the safest default for this project.
- npm, pnpm, or yarn

### Install and Run

```bash
npm install
npm run dev
```

Open `http://localhost:4321` in your browser.

### Build for Production

```bash
npm run build
npm run preview
```

## Scripts

- `npm run dev`: start the Astro dev server
- `npm run build`: generate the production build into `dist/`
- `npm run preview`: preview the production build locally
- `npm run astro`: run the Astro CLI directly

## Project Structure

```text
.
|- public/                     # Static files served as-is
|- src/
|  |- assets/                 # Source images and favicon assets
|  |- config/                 # Site, menu, social, and contact configuration
|  |- content/                # Content collections
|  |  |- about/               # Single about entry
|  |  |- authors/             # Author bios
|  |  |- pages/               # Standalone pages like privacy and terms
|  |  `- posts/               # Blog posts
|  |- layouts/                # Shared layouts and UI components
|  |- pages/                  # Astro routes and generated endpoints
|  |- styles/                 # Global Tailwind CSS entry point
|  `- utils/                  # Pagination, dates, reading time helpers
|- astro.config.mjs           # Astro config and integrations
|- package.json               # Dependencies and scripts
`- tsconfig.json              # TypeScript config and import aliases
```

## Configuration

### Runtime Site URL

Set the production domain in `astro.config.mjs` via the `site` field. This value is used by canonical URLs, RSS, sitemap generation, and SEO metadata.

```js
export default defineConfig({
    site: 'https://hieupn.com',
})
```

### Main Site Metadata

`src/config/site.ts` is the main runtime config for site-wide metadata and archive behavior.

- `title`: default page title
- `description`: default SEO description
- `lang`: document language
- `ogImage`: default social image
- `postsPerPage`: archive pagination size
- `noindex.tags`: whether tag archives should be noindexed
- `noindex.categories`: whether category archives should be noindexed
- `noindex.authors`: whether author archives should be noindexed

### Navigation

Update `src/config/menu.json` to control header and footer navigation.

- `main`: top navigation, including dropdown groups
- `footer`: footer links

### Social Links

Update `src/config/social.json` to control the social icons shown in the homepage hero and footer.

### Contact Page

`src/config/config.json` is used by the contact page.

- `params.contact_form_action`: external form POST target
- `contactinfo.address`: displayed on the contact page
- `contactinfo.email`: displayed on the contact page
- `contactinfo.phone`: displayed on the contact page

This project does not include a backend form handler. The contact form posts directly to whatever external endpoint you configure.

### Logo and Favicon

The current site logo and favicon are imported directly from `src/assets/favicons/hieupn_logo_better.svg`.

If you want to rebrand the site, replacing that asset is the simplest starting point.

## Content Model

The project uses Astro content collections defined in `src/content/config.ts`.

### Posts

Create posts in `src/content/posts/`.

```md
---
title: "Your Post Title"
meta_title: "Optional SEO title"
description: "Short summary used in cards, SEO, and search"
date: 2026-04-11
image: "../../assets/images/your-post-image.png"
authors: ["Pham Nhat Hieu"]
categories: ["Web Development"]
tags: ["astro", "mdx", "tailwind"]
series: ["Astro Basics", "1"]
canonical: "https://example.com/your-post-title/"
draft: false
---

Your content goes here.
```

Notes:

- `image` is required for posts and uses Astro's content image pipeline, so use a relative path from the markdown file.
- `series` is optional and must be a two-item tuple: `[series name, position]`.
- Category, tag, and author archive URLs are derived by lowercasing the values you enter. Prefer clean, URL-safe names.

### Authors

Create author bios in `src/content/authors/`.

```md
---
title: "Pham Nhat Hieu"
meta_title: "Optional SEO title"
image: "/images/authors/pham-nhat-hieu.webp"
description: "Short author bio"
social:
    facebook: "https://facebook.com/username"
    twitter: "https://twitter.com/username"
    instagram: "https://instagram.com/username"
draft: false
---

Author bio content goes here.
```

Important:

- Author archive pages are generated from the `authors` values inside post frontmatter.
- The author bio card on an author archive page is matched against the author entry `title`, case-insensitively.
- In practice, the post `authors` value should match the author entry `title` if you want the bio block to appear.

### Standalone Pages

Create standalone pages in `src/content/pages/`. Each file becomes a route handled by `src/pages/[slug].astro`.

```md
---
title: "Privacy Policy"
meta_title: "Privacy Policy"
description: "Privacy policy for the site"
image: "/images/privacy-cover.png"
draft: false
---

Page content goes here.
```

The filename becomes the route slug. The `404.md` entry is excluded from the generic page route and is handled separately.

### About Page

The about page is a dedicated collection entry at `src/content/about/index.md`.

```md
---
title: "About Hieu"
meta_title: "About Hieu"
image: "/images/profile.png"
description: "Short about-page description"
draft: false
what_i_do:
    title: "What I Bring"
    items:
        - title: "Backend Engineering"
            description: "Designing APIs and application logic for production systems"
        - title: "Cloud and DevOps"
            description: "CI/CD, containers, cloud workflows, and delivery discipline"
---

About page body content goes here.
```

### MDX Components

Posts and standalone pages automatically expose the components re-exported by `src/layouts/components/MarkdownComponents.astro`.

Currently available widgets:

- `Accordion`
- `Button`
- `ListCheck`
- `Notice`
- `Tab`
- `Tabs`
- `YouTubeEmbed`
- `SeriesWidget`

That means you can use them directly inside MDX content without adding local imports in each file.

## Routes and Generated Output

The current codebase generates these main route groups:

- `/`: homepage
- `/blog/` and `/blog/page/[page]/`: blog archive pages
- `/<post-slug>/`: blog posts from `src/content/posts/`
- `/about/`: dedicated about page
- `/contact/`: contact page
- `/authors/`, `/authors/[author]/`, `/authors/[author]/page/[page]/`
- `/categories/`, `/categories/[category]/`, `/categories/[category]/page/[page]/`
- `/tags/`, `/tags/[tag]/`, `/tags/[tag]/page/[page]/`
- `/series/`: list of all post series
- `/search/`: search UI
- `/search.json`: generated search index consumed by Fuse.js
- `/rss.xml`: RSS feed
- `/privacy/`, `/terms/`, and other standalone pages created from `src/content/pages/`

## Styling

Global styling lives in `src/styles/global.css` and is powered by Tailwind CSS 4.

The layout also includes:

- a persistent light/dark theme toggle
- critical inline theme logic to avoid a flash of the wrong theme on first load
- shared header, footer, and SEO handling across all pages

## Development Notes

- Import aliases are defined in `tsconfig.json` for `@components`, `@layouts`, `@config`, `@utils`, `@styles`, and `@assets`.
- The Astro dev server is configured to allow direct access to `src/assets` so string-based asset paths used by content can resolve during development.
- The sitemap configuration filters out paginated pages and the search page.

## Deployment

The site builds to static output in `dist/`, which makes it suitable for static hosting providers such as Cloudflare Pages, Netlify, Vercel static output, GitHub Pages, or any standard web server.

Build command:

```bash
npm run build
```

## License

This project is licensed under the MIT License. See `LICENSE` for details.