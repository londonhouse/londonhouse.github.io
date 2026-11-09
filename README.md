# londonhouse.github.io
Public-facing repos from London House Group (LHG) — featuring real-world infrastructure, AI workflows, and open project blueprints for learning, consulting, and collaboration. Built for engineers, recruiters, and teams who value practical, scalable systems.

## Publishing an article

Articles are Markdown files rendered by the Jekyll build that GitHub Pages already
runs on this repo. No local tooling or build step is required.

1. Copy the template into the articles folder and give it a URL-friendly name:

   ```bash
   cp ARTICLE-TEMPLATE.md _articles/my-article-slug.md
   ```

2. Fill in the front matter — `title` and `date` are required; `description` and `tags`
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

### Drafts

Work-in-progress articles live in `drafts/`. The folder is excluded from the Jekyll
build (`_config.yml`) and from git (`.gitignore`), so nothing in it is rendered or
pushed. When a draft is ready, move it into `_articles/` and follow the steps above.

### Review

Before publishing, run the review process in Claude Code:

1. `/article-review drafts/my-article.md` writes an assessment report and a proposed
   auto-fix copy to `scratch/` (`<slug>-review.md`, `<slug>.autofix.md`,
   `<slug>.autofix.diff`). Spelling fixes are applied to the article right away and
   reported as a count; the original is kept as `scratch/<slug>.prespell.md`. Nothing
   else in the article is changed until you apply.
2. Read the report, then `/article-review apply drafts/my-article.md` (optionally
   "except G4, A2") applies the approved fixes.
3. `/article-convert drafts/my-article.md` moves the draft to `_articles/<slug>.md` with
   front matter, removes draft scaffolding, adds series links, and draws the planned
   diagrams (SVG in `assets/diagrams/`, PNG in `assets/images/`). The article is set to
   `published: false` and nothing is committed. Run a final `/article-review` on it,
   then remove `published: false` to publish.

The process, grading rubric, voice profile, and auto-fix tuning log live in
`.claude/skills/article-review/`. Rejected auto-fixes are recorded in the tuning log
so later reviews follow them.

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
