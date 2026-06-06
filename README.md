# Kmoon Blog

A Hugo-based personal site for technical writing, essays, and personal pages.

## Development

- `git submodule update --init --recursive` — fetch the `themes/hugo-bearneo` theme submodule.
- `hugo server -D` — run the local development server and include drafts.
- `hugo` — build the production site into `public/`.

## Content structure

- `content/_index.md` — home page content.
- `content/blog/` — blog posts.
- `content/essay/` — essays.
- `content/about.md`, `content/resume.md` — standalone navigation pages.
- `archetypes/` — starter templates for new content.

## Authoring notes

- Edit source files under `content/`, `layouts/`, `static/`, and `hugo.toml`.
- Do not hand-edit generated files under `public/`; rebuild them with Hugo instead.
- Blog permalinks are configured as `/:slug/`, so keep front matter `slug` stable when URLs matter.

## Verification

- Run `hugo` before publishing to verify templates and regenerate the site output.
