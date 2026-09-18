# buth.dev

Single-page personal site. There is no build step: `docs/index.html` is the
deployed artifact, with its CSS inline and no JavaScript. The page loads exactly
two things, both same-origin: itself and one font.

## Deploy

GitHub Pages, **`main` branch, `/docs` folder** (Settings → Pages). `docs/CNAME`
pins the apex domain. Pushing to `main` deploys.

> The `master` branch holds a completely different, much older version of this
> site. If Pages is ever pointed there, the live site silently reverts to it.
> One command tells you what is actually published:
>
>     curl -s https://buth.dev/ | grep -i '<title>'
>     # expect: Eric Buth - Software Engineer in NYC

## Local preview

    python3 -m http.server 8000 --directory docs --bind 127.0.0.1
    open http://localhost:8000/

Use `http://`, not `file://`: the asset paths are root-absolute (`/og.png`,
`/fonts/…`), which `file://` resolves against the filesystem root, and Chrome
blocks `file://`-to-`file://` subresource loads, so the font silently fails to
load and you end up looking at the fallback face.

Python's `http.server` serves `.woff2` as `application/octet-stream`, where
GitHub Pages sends `font/woff2`. Harmless — browsers sniff font formats — but
worth knowing if you go looking at response headers locally.

## Fonts

`docs/fonts/` holds the Google Fonts `latin` subset of IBM Plex Sans Regular,
self-hosted so the page has no third-party origin on its critical path. Weight
400 normal is the only face the page uses.

IBM Plex is licensed OFL 1.1, which requires the license accompany the font:
see `docs/fonts/OFL.txt`.

To refresh it:

    npm pack @fontsource/ibm-plex-sans
    # then files/ibm-plex-sans-latin-400-normal.woff2

The `<link rel="preload">` for it **must** keep its `crossorigin` attribute.
Font fetches are CORS-mode `anonymous` even same-origin, so without it the
preload does not match the real request and the browser downloads the font
twice.

## Regenerating the images

Sources live in `assets/`, which is outside `docs/` and therefore never
published. `docs/og.png`, `docs/favicon.ico`, and `docs/apple-touch-icon.png`
are all generated; `docs/favicon.svg` is hand-maintained.

    uv run assets/build.py

Python dependencies are declared inline in `assets/build.py` itself (PEP 723
script metadata) and pinned exactly in `assets/build.py.lock`, so there is no
`pyproject.toml`, no virtualenv to create, and no list of packages to keep in
sync with this README. `uv run` reads both and builds the environment on the
fly. Use `uv run --locked` to make it fail rather than silently re-resolve if
the lockfile has drifted. After changing the dependency list, re-pin with
`uv lock --script assets/build.py`.

Pinning matters more than it looks: the build is deterministic — rerunning it
reproduces byte-identical output — and that only holds as a claim if the
shaping and font libraries are held still. A fontTools or HarfBuzz upgrade can
shift glyph outlines slightly and silently rewrite the committed images.

The one dependency uv cannot supply is `rsvg-convert` (`brew install librsvg`),
which does the rasterizing. Both it and uv are one-time local tools, not
dependencies of the site.

Two things about the sources that are easy to get wrong:

- **`assets/icon.svg` is deliberately not `docs/favicon.svg`.** The shipped
  favicon inverts via `prefers-color-scheme`, which librsvg does not evaluate —
  rasterizing it yields a near-black glyph on transparent, invisible in a dark
  tab strip. The raster masters are opaque and fixed instead, because the
  contexts that consume them (older Safari, iOS home screens, crawlers, link
  preview caches) cannot adapt to a theme.
- **`assets/og.svg` is generated, not hand-edited.** Its text is baked to
  outlines by `build.py`, because librsvg on macOS resolves fonts through
  CoreText rather than fontconfig and would otherwise render tofu unless IBM
  Plex Sans were installed system-wide. Change the layout constants in
  `build.py` and re-run it.

## After deploying

    curl -s https://buth.dev/ | grep -i '<title>'
    curl -sI https://buth.dev/og.png | grep -E '^HTTP|content-type|content-length'
    curl -sI https://buth.dev/favicon.ico | grep -E '^HTTP|content-type'

A broken link preview is almost always a 404 on the image rather than a bad
tag. Then paste the URL into Slack or Bluesky to confirm the card renders.
