# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Fetch the theme submodule (required after clone)
git submodule update --init --recursive

# Local dev server with drafts
hugo server -D

# Production build (output goes to public/)
hugo

# Create a new blog post from archetype
hugo new blog/my-post-slug.md

# Create a new essay from archetype
hugo new essay/my-essay-slug.md
```

A Windows `hugo.exe` binary is committed at the repo root for CI convenience. On macOS/Linux, use a locally installed Hugo binary.

## Architecture

This is a **Hugo static site** (blog.kmoon.fun) built on the **hugo-bearneo** theme (git submodule at `themes/hugo-bearneo`). The project overrides several theme templates and adds custom partials in `layouts/`.

### Content sections

| Section | Front matter type | Permalink pattern | Notes |
|---------|-------------------|-------------------|-------|
| `content/blog/` | `blog` | `/:slug/` | Tech posts; tags, `status: landing` for WIP indicator |
| `content/essay/` | `essay` | default | Personal/annual essays |
| `content/gists/` | `gists` | default | Short snippets |
| `content/*.md` | — | default | Standalone pages: about, resume, links |

Blog permalinks are `/:slug/` (configured in `hugo.toml` `[permalinks]`). **Keep `slug` stable** when changing existing blog post URLs.

### Layout override chain

Hugo resolves templates by checking `layouts/` before `themes/hugo-bearneo/layouts/`. Key overrides:

- `layouts/_default/single.html` — article page: word count, reading time, upvote button, Giscus comments (blog/essay only), TOC
- `layouts/_default/list.html` — post list with year grouping, search, WIP spinners
- `layouts/_default/_markup/render-image.html` — image zoom (checkbox-based CSS-only overlay) with lazy loading; hero gif exempted
- `layouts/index.html` — home page (just renders `.Content`)
- `layouts/partials/` — all custom partials override theme equivalents:
  - `custom_head.html` — LXGW WenKai font, theme toggle (system/light/dark cycle with localStorage)
  - `style.html` — **all CSS** (variables, base, nav, code/Chroma, TOC, upvote, image zoom, Giscus, solar animation, responsive)
  - `comments.html` — Giscus with auto dark/light sync via MutationObserver
  - `toc.html` — scroll-spy TOC with rAF-throttled active link tracking
  - `nav.html`, `header.html`, `footer.html`, `seo_tags.html`, `favicon.html` — nav, footer, SEO meta, favicon

### CSS and styling

All styles live in the single `layouts/partials/style.html` file as an inline `<style>` block. There is no separate `assets/` directory or pipeline. Code highlighting uses Chroma with `noClasses = false` (CSS class-based), styled via `--code-*` CSS custom properties defined in `style.html`.

### Theme system

Three-way toggle: `system` → `light` → `dark`. Logic is in `custom_head.html`. CSS uses `:root` (light), `@media (prefers-color-scheme: dark)` for system-preference, and `:root[data-theme="dark"]` / `:root[data-theme="light"]` for explicit overrides.

### CJK support

`hasCJKLanguage = true` in `hugo.toml` — Hugo counts CJK characters correctly for `.WordCount` and `.ReadingTime`. Blog posts display word count and reading time in Chinese ("约 X 字 · Y min").

## Authoring conventions

### Archetypes

- **blog**: `title`, `date`, `tags`, `hidden`, `status` (set `status = "landing"` for WIP spinner)
- **essay**: `title`, `date`, `hidden`
- **default**: `title`, `date`, `menu`, `menuHidden`

### Image handling

Images rendered via `render-image.html` get: lazy loading, `<figure>`/`<figcaption>`, and click-to-zoom (disabled for the home hero gif). Use `#minipic` fragment in `src` for half-width images.

### Content is primarily Chinese

Blog posts and essays are written in Chinese. The site locale is `zh-CN`. Giscus language is `zh-CN`.

## What not to edit

- `public/` — generated output; rebuild with `hugo`
- `themes/hugo-bearneo/` — git submodule; customize via `layouts/` overrides instead
- `resume.md` — gitignored (contains personal info)

## Tolaria vault

This repo doubles as a Tolaria knowledge vault. Agent instructions are in `AGENTS.md` (CLAUDE.md and GEMINI.md are compatibility shims that reference it). See `AGENTS.md` for note conventions, types, wikilinks, and view rules.
