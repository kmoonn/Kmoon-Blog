# Kmoon Blog

A Hugo-based personal site for technical writing, essays, and personal pages. Live at [blog.kmoon.fun](https://blog.kmoon.fun/).

<p>
  <img src="https://cdn.kmoon.fun/2026/2026-06-23T02-46-49-044Z.png" alt="Light mode" width="45%" style="border-radius:12px" />
  <img src="https://cdn.kmoon.fun/2026/2026-06-23T02-47-48-876Z.png" alt="Dark mode" width="45%" style="border-radius:12px" />
</p>

## Development

- `git submodule update --init --recursive` — fetch the `hugo-bearneo` theme submodule.
- `hugo server -D` — run the local development server and include drafts.
- `hugo` — build the production site into `public/`.

## Content structure

| Path | Description |
|------|-------------|
| `content/_index.md` | Home page |
| `content/blog/` | Blog posts (tech, tools, AI) |
| `content/essay/` | Essays & personal writing |
| `content/about.md` | About page |
| `content/resume.md` | Resume page |
| `content/links.md` | Links / bookmark page |

Archetypes: `blog.md`, `essay.md`, `default.md`.

## Site features

- **Theme** — [hugo-bearneo](https://github.com/kmoonn/hugo-bearneo) with custom layouts in `layouts/`.
- **CJK support** — `hasCJKLanguage = true` for accurate Chinese word count and reading time.
- **Comments** — [Giscus](https://giscus.app) powered, auto light/dark mode switching.
- **TOC** — h2–h3 table of contents on article pages.
- **Image zoom** — click-to-enlarge images on article pages.
- **Post navigator** — prev/next links between posts.
- **SEO** — `enableRobotsTXT`, Open Graph, Twitter Card meta.
- **Code highlighting** — Chroma with `github` style, external CSS in `assets/css/`.

## Authoring notes

- Edit source files under `content/`, `layouts/`, `static/`, `assets/`, and `hugo.toml`.
- Do not hand-edit generated files under `public/`; rebuild them with Hugo instead.
- Blog permalinks are configured as `/:slug/`, so keep front matter `slug` stable when URLs matter.

## Verification

- Run `hugo` before publishing to verify templates and regenerate the site output.
