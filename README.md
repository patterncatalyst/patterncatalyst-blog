# Pattern Catalyst Blog

Field notes mined from our workshops, tutorials, and reference builds.

**Live site:** <https://patterncatalyst.github.io/patterncatalyst-blog/>
· **RSS:** <https://patterncatalyst.github.io/patterncatalyst-blog/feed.xml>

Part of the PatternCatalyst family; the
[Workshops and Tutorials hub](https://patterncatalyst.github.io/patterncatalyst-workshops-tutorials-list/)
links to posts here.

## How it's built

A [Jekyll](https://jekyllrb.com/) blog in the PatternCatalyst house style
(Red Hat fonts, amber accent), extending the shared site template with a
`_posts` collection, `jekyll-paginate-v2` (pagination plus tag and category
pages), and `jekyll-feed` (RSS). Deployed to GitHub Pages by
`.github/workflows/pages.yml`. Post listings render as tables.

## Add a post

Create `_posts/YYYY-MM-DD-slug.md` with the front matter shown in
[`CLAUDE.md`](CLAUDE.md), write the body in the house voice, embed any diagrams
as paired SVG + `.excalidraw` assets, and link to the sample code in the source
project. `CLAUDE.md` has the full checklist.

## Run locally

```bash
bundle install
bundle exec jekyll serve --baseurl ""
# http://localhost:4000/
```
