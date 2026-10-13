# carlossolar.es

Personal site of Carlos Solares — a Hugo static site deployed to GitHub Pages at
https://carlossolar.es/.

> **Status:** Builds cleanly on Hugo v0.167.0 and deploys to GitHub Pages via the `public/`
> submodule (`mixcas/mixcas.github.io`). The theme is committed in-repo, and since commit
> `bd33ab5` the committed output is **reproducible** — a fresh `hugo` build matches `public/`
> byte-for-byte, so `./deploy.sh` no longer destroys anything. One caveat on fresh clones:
> `public/` is checked out detached, which makes `deploy.sh` silently push nothing — see
> [Known issues](#known-issues--gotchas).

## Stack

| Layer | Technology |
|---|---|
| Static site generator | Hugo (Go), TOML config, no JS toolchain, no package manager |
| Theme | `nightgum` — vendored in-repo under `themes/nightgum/`, MIT, derived from the Jekyll theme *Heather* |
| Content | Markdown with TOML front matter; **currently zero content files** |
| Styling | One hand-written stylesheet + a self-hosted WOFF font (Linux Libertine) |
| Build output | `public/` — a git submodule pointing at `mixcas/mixcas.github.io` |
| Hosting | GitHub Pages (serves `mixcas.github.io`) |
| DNS / CDN | Cloudflare in front of GitHub Pages (`CNAME` file contains `carlossolar.es`) |
| Deploy | `deploy.sh` — build, then commit and push the `public` submodule |

## Repository layout

```text
.
├── .gitignore          # ignores .hugo_build.lock and resources/
├── .gitmodules         # submodule definition (public only)
├── archetypes/
│   └── default.md      # TOML front matter template used by `hugo new`
├── config.toml         # site config (baseurl, locale, title, theme, [params])
├── deploy.sh           # build + commit + push the public/ submodule
├── public/             # SUBMODULE → mixcas/mixcas.github.io (the published site)
└── themes/
    └── nightgum/       # the theme, tracked directly in this repo
```

There is **no `content/` directory** and **no root `layouts/`**; all templates resolve from
`themes/nightgum/`. `hugo new content …` refuses to run until a `content/` directory exists.

The theme is **not** a submodule: its files are committed here, so `git clone` alone is enough to
build (no `--recurse-submodules` needed). It was migrated off the removed `.Site.Author` /
deprecated `.Site.LanguageCode` APIs — see [Hugo version compatibility](#hugo-version-compatibility).

Theme files (`themes/nightgum/`):

| Path | Purpose |
|---|---|
| `layouts/index.html` | Home template: `head` + `description` partials, `<main class="home section">`, `{{ range site.RegularPages }}` |
| `layouts/404.html` | Not-found page |
| `layouts/_default/single.html` | Single page template |
| `layouts/_default/list.html` | List template for `section`, `taxonomy`, and `term` kinds (`tags/`, `categories/`, post sections) |
| `layouts/partials/head.html` | `<!doctype>`, `<head>` (lang, title, author + description meta), the `<header>`, and the `<h2>` subtitle rendered from `[params] description` through `safeHTML` |
| `layouts/partials/description.html` | Hand-written bio paragraphs (Ensenada / Mexico City, Videate, Github, public-CV link); no longer reads config |
| `layouts/partials/foot.html` | Closing `</footer></body></html>` |
| `layouts/shortcodes/gist.html` | `{{< gist >}}` shortcode |
| `layouts/shortcodes/youtube.html` | `{{< youtube >}}` shortcode |
| `static/style.css` | Site stylesheet (104 lines, Heather-derived) |
| `static/fonts/LinLibertine_DR.woff` | Self-hosted Linux Libertine font |
| `static/favicon.png`, `static/apple-touch-icon.png` | Icons |
| `theme.toml` | Theme metadata (`min_version = 0.13`, stale) |
| `README.md`, `archetypes/default.md` | Both **empty (0 bytes)** |

## How it works

```mermaid
graph LR
  A[config.toml + themes/nightgum] --> B[hugo]
  B --> C[public/ submodule working tree]
  C --> D[git commit && git push]
  D --> E[mixcas/mixcas.github.io]
  E --> F[GitHub Pages]
  F --> G[Cloudflare]
  G --> H[carlossolar.es]
```

- Hugo writes the built site **directly into the `public/` submodule's working tree**
  (`publishDir` defaults to `public`), so a plain `hugo` or `hugo server` mutates a tracked
  submodule. Pass `--destination` (or `-d`) to build somewhere else.
- `themes/nightgum` is now a normal directory in this repo: edit templates and stylesheets here,
  in place. No submodule pointer to bump.
- The output committed in `public/` now **matches** this repo's build output (verified by a
  clean `hugo -d` diff against `public/`), so rebuilding and deploying is lossless.

## Requirements

- **Hugo v0.158.0 or newer** (`site.Language.Locale` and the `locale` config key require it).
  Verified working with the current `brew` release, **v0.167.0**.
- Git, with SSH access to `mixcas/carlossolar-es` and `mixcas/mixcas.github.io`.
- Docker, if using the container route (optional).

## Getting started

```bash
git clone git@github.com:mixcas/carlossolar-es.git
cd carlossolar-es
git submodule update --init public    # only needed to build into/publish via public/
```

`git submodule update --init public` checks `public/` out **detached** at the recorded commit.
Before running `deploy.sh`, attach it to the branch: `git -C public switch main`.

### Installing Hugo

`brew install hugo` installs the current release (0.167.0 as of writing), which builds this
theme cleanly:

```bash
brew install hugo
hugo version   # expect: hugo v0.167.0+extended ... darwin/arm64
```

Docker (any supported version, no local install):

```bash
docker run --rm -it -v "$PWD":/project -p 1313:1313 \
  ghcr.io/gohugoio/hugo:v0.167.0 server --bind 0.0.0.0
```

The image's working directory is `/project`, it runs as uid 1000, and `--bind 0.0.0.0` is what
makes the port reachable from the host.

On macOS, Docker Desktop only shares some host paths by default. Mounting an unshared directory
(e.g. under `/private/tmp`) succeeds but appears **empty** inside the container, and Hugo then
fails with `Unable to locate config file or config directory`. Keep the clone under `$HOME`.

## Develop

- `hugo server` → http://localhost:1313/ (defaults: port 1313, bind 127.0.0.1). Add
  `-D`/`--buildDrafts` to include drafts, `--disableFastRender` for exact rebuilds. Live reload
  works. A `content/` directory must exist first if new pages are to be authored.
- **`hugo`/`hugo server` write into `public/` by default** (see [Build](#build)), dirtying the
  submodule. For a preview that leaves `public/` alone, use
  `hugo server -d /tmp/hugo-preview` or the Docker command above (which also builds in place, so
  prefer `-d` there too if you care about the working tree).
- Editing the home page means editing `themes/nightgum/layouts/index.html` (markup),
  `themes/nightgum/layouts/partials/head.html` (the `<h2>` subtitle, from
  `config.toml` `[params] description`), or `themes/nightgum/layouts/partials/description.html`
  (the bio paragraphs, which are hard-coded there).

## Build

```bash
hugo            # writes to ./public (Hugo's default publishDir)
hugo --minify   # optional
hugo -d /tmp/out   # build elsewhere, leaving public/ untouched
```

There are **no `content/` files**, so the build reports `Pages 8` and emits: the home page,
`404.html`, `sitemap.xml`, `index.xml` (RSS), and empty `tags/index.html` +
`categories/index.html` taxonomy list pages (each also gets an `index.xml`), plus a copy of
`themes/nightgum/static/*` (`style.css`, `favicon.png`, `apple-touch-icon.png`,
`fonts/LinLibertine_DR.woff`). The build is clean — no warnings or deprecations.

With content added, `_default/list.html` and `_default/single.html` render sections, taxonomy
terms, and individual posts.

`.hugo_build.lock` is created in the project root during builds and is git-ignored; it is safe to
delete.

## Deploy

```bash
./deploy.sh                 # commit message: "rebuilding site <date>"
./deploy.sh "my message"    # commit message: "my message"
```

Manual equivalent:

```bash
hugo
cd public
git add -A
git commit -m "rebuilding site $(date)"
git push origin HEAD:main   # `main`, NOT `master` — deploy.sh pushes `main`
cd ..
```

> **WARNING:** `hugo` builds into `public/` (Hugo's default `publishDir`) and `deploy.sh` then
> commits and pushes the whole thing to `mixcas/mixcas.github.io`. Because the committed output is
> now reproducible, this is safe — but it pushes **whatever is in `public/`**, so build from a
> clean tree. On fresh clones `public/` is detached and the push silently no-ops; see
> [Known issues](#known-issues--gotchas).

The parent repo must be pushed separately (`git push origin master`) — `deploy.sh` only touches
the `public` submodule.

## Known issues & gotchas

1. **`public/` is checked out detached.** `git submodule update --init public` (and any fresh
   clone) leaves a detached HEAD. `deploy.sh` runs `git push origin main` inside `public/`, which
   — with no local `main` branch — pushes the *local* `main` ref, not `HEAD`: if that branch is
   absent the push reports `Everything up-to-date` and **nothing is deployed**, while the local
   commit is silently discarded. Fix: `git -C public switch main` before deploying, or change the
   line to `git push origin HEAD:main`.
2. **`deploy.sh` has no error handling.** No `set -e` and no `git diff --cached` guard: on an
   unchanged build `git commit` fails with `nothing to commit` (exit 1) but the script continues
   to push and exits 0, so a failed deploy looks successful.
3. **The build is reproducible, but only from a clean tree.** `public/` exactly matches
   `hugo -d <tmpdir>` output as of `bd33ab5`. Stale files in `public/` from an older theme revision
   would be committed as-is, since `deploy.sh` uses `git add -A` (no `--cleanDestinationDir`).
4. **`baseurl` uses `http://` and is lowercase.** Lowercase is fine (Hugo lowercases config keys
   internally) but is non-idiomatic; `baseURL` with `https://` is correct for a
   Cloudflare-fronted HTTPS site. The emitted `sitemap.xml`, `index.xml`, and `<loc>` values
   currently advertise `http://carlossolar.es/`.
5. **`archetypes/default.md` has no `title`/`date`.** `hugo new content` output would need both
   filled in manually.
6. **No `content/` directory exists**, so `hugo new content post.md` fails with
   `no existing content directory configured for this project` until `content/` is created.
7. **`config.toml` `[params] description` is a single string used twice.** In `head.html` it
   becomes both the `<meta name="description">` and the visible `<h2>` subtitle, escaped in the
   former and `safeHTML` in the latter. Multi-line HTML there would break the meta tag.

## Hugo version compatibility

The theme was migrated for Hugo v0.158.0+; minimum supported version is **v0.158.0** (verified —
v0.156.0 fails because `site.Language.Locale` does not exist yet).

| API / key | Before | Now (in this repo) | Why |
|---|---|---|---|
| `.Site.Author` | `head.html:6` | `site.Params.author.name` | Removed in Hugo v0.156.0 (hard build error) |
| `.Site.LanguageCode` | `head.html:2` | `site.Language.Locale` | Deprecated v0.158.0, removed later |
| `languageCode` (config) | `config.toml:2` | `locale` | Deprecated v0.158.0 |
| `.Data.Pages` | `index.html:5` | `site.RegularPages` | Legacy API; also lists only regular pages, not section/taxonomy nodes |
| date layout | `index.html`, `single.html` | `2006-01-02` / `Jan 02, 2006` | The old `01`/`30` tokens are Go's *month*, producing crooked dates |
| taxonomy layouts | *(none)* | `_default/list.html` | Without it, `tags/` and `categories/` render nothing and Hugo warns `found no layout file … for kind "taxonomy"` |

`params.author.name` is read from `config.toml`:

```toml
[params]
  author.name = "Carlos Solares"
```

Because the templates are committed in this repo, upgrading Hugo only requires editing
`themes/nightgum/layouts/*` here — no submodule bump, no separate theme repo.

## License

- Theme `nightgum`: MIT (`themes/nightgum/LICENSE.md`), derived from the Jekyll theme *Heather*.
- No root `LICENSE` file exists; the site content and config are personal and unlicensed.
