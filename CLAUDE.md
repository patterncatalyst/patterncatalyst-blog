# CLAUDE.md — patterncatalyst-blog

Rules and conventions for working on this repo. Read before making changes.

## What this is

The Pattern Catalyst Blog: a post-centric Jekyll + GitHub Pages site holding
articles mined from our workshops, tutorials, and reference builds. It is a
separate repo from the hub (`patterncatalyst-workshops-tutorials-list`); the
hub's "Our Blogs" page links to posts here. Live at
<https://patterncatalyst.github.io/patterncatalyst-blog/>.

## How it's built

Extends the `lgtm-jekyll` house template (Red Hat fonts, amber accent
`#e8870c`, shared `assets/css/site.css`) with a blog layer:

- `_posts/` collection with dated permalinks (`/:year/:month/:day/:title/`).
- `jekyll-paginate-v2` for pagination and auto tag/category index pages. This
  works only because `.github/workflows/pages.yml` builds with
  `bundle exec jekyll build`; do not switch to `actions/jekyll-build-pages`, or
  the plugin allowlist would reject paginate-v2.
- `jekyll-feed` RSS at `/feed.xml`.
- Layouts: `post.html` (single post), `post_list.html` (tag/category pages).
- Post listings render as a **table** via `_includes/post_table.html` (home +
  tag/category pages). Keep listings tabular, not card grids.

## How to add a post

Create `_posts/YYYY-MM-DD-slug.md` with this front matter:

```yaml
---
title: "A specific, concrete title"
date: 2026-10-07
author: "Pattern Catalyst"
tags: [tag-a, tag-b]
categories: [cloud-native]
canonical_project:
  name: "Source Project Name"
  repo: "patterncatalyst/<source-repo>"
  url: "https://patterncatalyst.github.io/<source-repo>/"
excerpt: "One-sentence summary used on the listing and in the feed."
# hero: /assets/img/heroes/<name>.svg   # optional
---
```

Then:

- Write the body in the `lgtm-professional-voice` register: peer-to-peer, for
  engineers. **No em-dash overuse** (use commas, semicolons, colons, or
  separate sentences) and no contrived/AI-sounding phrasing. Run
  `lgtm-professional-voice`'s `scan.sh` on the file before committing.
- Embed diagrams with `{% include excalidraw.html file="name" alt="…"
  caption="…" %}` and commit the paired `assets/diagrams/name.svg` +
  `name.excalidraw`. Generate with `lgtm-diagram-generator`.
- Link to sample code in the source project, pinned to a tag/branch (for example
  `stage/06`) so the link stays in sync with the prose.
- End with a **Source project** note linking to `canonical_project.url`.
- Feature the post on the hub: add an entry to the hub repo's `_data/blogs.yml`
  with the post's live URL.

Internal links back to the hub use `{{ site.hub_url }}`, not a hardcoded URL.

## Build, preview, publish

```bash
bundle install
bundle exec jekyll serve --baseurl ""   # preview at http://localhost:4000/
bundle exec jekyll build                # must exit 0, no Liquid/YAML errors
```

Pushing to `main` builds and deploys to GitHub Pages via the Actions workflow.
Confirm the run is green after a push.
