# Tech Notes — Hugo theme

An editorial Hugo theme: paper-and-ink palette, Source Serif 4 + Inter,
slow-reading layout. Built for `SandeepVaman/myblog`.

## What's here

```
themes/tech-notes/
├── theme.toml
├── assets/css/main.css
└── layouts/
    ├── _default/
    │   ├── baseof.html        # base wrapper (head, fonts, header, footer)
    │   ├── list.html          # /posts/ index
    │   ├── single.html        # individual post
    │   ├── page.html          # generic standalone pages
    │   ├── taxonomy.html      # /tags/ — rendered as the Playlists grid
    │   └── term.html          # /tags/databases/ — list of posts in a tag
    ├── index.html             # homepage
    ├── page/
    │   └── about.html         # custom About page (colophon + contact)
    └── partials/
        ├── header.html
        ├── footer.html
        ├── post-row.html
        └── pl-mark.html       # SVG cover marks for playlist cards
```

## Install

The theme already lives at `themes/tech-notes/` in this repo. Update
your `hugo.toml`:

```toml
theme = 'tech-notes'   # was: cleantech
```

That's it. Push to `main` and the existing `.github/workflows/hugo.yml`
will build and deploy.

## Mapping to your content

| Hugo concept | URL | Template | Design |
|---|---|---|---|
| Home | `/` | `layouts/index.html` | Editorial post list |
| Section | `/posts/` | `_default/list.html` | Same post-row style |
| Single | `/posts/hello-world/` | `_default/single.html` | Reading page |
| Taxonomy | `/tags/` | `_default/taxonomy.html` | **Playlists grid** (every tag = one playlist) |
| Term | `/tags/distributed-systems/` | `_default/term.html` | Filtered post list |
| Page | `/about/` | `page/about.html` | Custom About w/ colophon |

The "Playlists" page from the design comp is wired to your existing
`tag` taxonomy. Every tag automatically becomes a card; the card
expands on hover to reveal the posts in that tag.

If you later want hand-curated playlists (a different sequence,
custom intro notes, mixing posts across tags), add a new taxonomy:

```toml
[taxonomies]
  tag = 'tags'
  category = 'categories'
  playlist = 'playlists'
```

…then add `playlists: ["boring-tech", "papers"]` to post frontmatter
and add a `Playlists` menu item pointing at `/playlists/`. The same
`taxonomy.html` template will render it.

## About page — optional structured frontmatter

The About page picks up two optional frontmatter blocks for the
colophon facts and contact strip. Edit `content/about.md`:

```yaml
---
title: "About"
hideMetadata: true
since: 2019

facts:
  - { key: "Name",        value: "Sandeep Vaman" }
  - { key: "Role",        value: "Staff engineer, infra" }
  - { key: "Based in",    value: "Bengaluru, IN" }
  - { key: "Working on",  value: "Storage & query layers" }
  - { key: "Writing since", value: "2019" }
  - { key: "Tools",       value: "Vim, Postgres, a paper notebook" }

contact:
  - { key: "Email",    value: "you@example.com" }
  - { key: "GitHub",   value: "@SandeepVaman" }
  - { key: "LinkedIn", value: "in/sandeepvaman" }
---

Welcome to my blog. I write about software engineering, distributed
systems, and *boring* technology that survives.
```

If you omit `facts` / `contact`, the page just renders your prose.

## Recommended menu update (`hugo.toml`)

The current menu has Home / Posts / Tags / About. Consider renaming
**Tags** to **Playlists** — same URL, friendlier label:

```toml
[[menus.main]]
  name = 'Playlists'
  pageRef = '/tags'
  weight = 30
```

## Author / social

Add to `[params]` so the footer links light up:

```toml
[params]
  author = 'Sandeep'
  description = 'Long-form writing on databases & distributed systems.'

  [params.social]
    github = 'SandeepVaman'
    linkedin = 'sandeepvaman'
```

## Local preview

```bash
hugo server -D
```

Open `http://localhost:1313`.

## Notes

- Reading time uses Hugo's built-in `.ReadingTime` (whole minutes).
- Fonts come from Google Fonts via the document head — no build step.
- CSS is a single file, fingerprinted and minified by Hugo's pipes.
- No JS dependency. Hover preview on playlist cards is pure CSS.
