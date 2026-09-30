# londonhouse.github.io
Public-facing repos from London House Group (LHG) — featuring real-world infrastructure, AI workflows, and open project blueprints for learning, consulting, and collaboration. Built for engineers, recruiters, and teams who value practical, scalable systems.

## Publishing an article

Articles are Markdown files rendered by the Jekyll build that GitHub Pages already
runs on this repo. No local tooling or build step is required.

1. Copy the template into the articles folder and give it a URL-friendly name:

   ```bash
   cp ARTICLE-TEMPLATE.md _articles/my-article-slug.md
   ```

2. Fill in the front matter — `title` and `date` are required; `summary` and `tags`
   are recommended. All fields are documented in `ARTICLE-TEMPLATE.md`.

3. Write the body in Markdown. Put images and diagrams in `assets/images/` or
   `assets/diagrams/` and reference them with an absolute path:

   ```markdown
   ![Alt text](/assets/diagrams/my-diagram.svg)
   ```

4. Commit and push to `main`:

   ```bash
   git add . && git commit -m "Add article: my article slug" && git push
   ```

Pages rebuilds automatically (~1 minute). The article then appears at
`/articles/my-article-slug/`, is listed on `/articles/`, and is included in
`/articles/feed.xml`.

### Syndication

The canonical URL of every article points at this site, so republishing elsewhere
attributes the original here.

- **Medium** — use Import a story and paste the article's URL; Medium sets the
  canonical link back automatically.
- **Dev.to** — point its RSS import at `https://londonhouse.github.io/articles/feed.xml`.
- Record where a piece has been syndicated using the optional `syndication` field
  in the front matter; the links render in the article footer.

### Layout notes

- `_config.yml` — Jekyll config and the `articles` collection.
- `_layouts/`, `_includes/` — shell used by article pages only.
- `index.html` — the home page, intentionally a plain static file that Jekyll copies
  verbatim. Its nav markup is duplicated in `_includes/site-nav.html`; update both
  if the nav changes.
- `css/styles.css` — shared styles. `css/article.css` — article pages only.
