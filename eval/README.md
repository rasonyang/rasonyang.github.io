# Theme evaluation (#4)

Three themes build from the same content, with no content changes:

| Theme | Build | Config |
|---|---|---|
| PaperMod (default, unchanged) | `hugo` | `hugo.toml` |
| Blowfish v3.9.0 | `hugo -e blowfish` | `config/blowfish/` |
| Hextra v0.13.0 | `hugo -e hextra` | `config/hextra/` |

Both candidates are git submodules pinned by commit (`themes/blowfish`,
`themes/hextra`). Hugo merges `hugo.toml` with `config/<env>/`, so the root
config (taxonomies, permalinks, outputs, menus) is inherited and the overlay
only switches the theme and its params. `config/_default` is not used, so a
plain `hugo` (and the CI workflow) is unaffected: the default build output is
byte-identical to `main`.

Preview: `hugo server -e blowfish -p 1313 --disableLiveReload` (or `-e hextra`).
Note `-e <name>` replaces `HUGO_ENVIRONMENT=production`, so GoatCounter (which
is gated on `hugo.IsProduction`) is not emitted in these builds. Screenshots:
`eval/screenshots/<theme>-<home|post>-<light|dark>.jpg`, 1280x800 viewport,
full page, dark mode via `prefers-color-scheme` emulation.

## Shared layout overrides (project root `layouts/`)

- `_partials/comments.html` was renamed to `_partials/giscus.html`; a thin
  `partials/comments.html` calls it. Blowfish probes
  `templates.Exists "partials/comments.html"`, which only matches the legacy
  `partials/` directory name. PaperMod output is unchanged (verified by diff).
- `partials/extend-head.html` (Blowfish) and `_partials/custom/head-end.html`
  (Hextra): GoatCounter. PaperMod keeps `_partials/extend_head.html`.
- `partials/home/custom.html` (Blowfish) and `_partials/custom/content-begin.html`
  + `page-end.html` (Hextra): render `params.homeInfoParams` and, for Hextra, a
  recent-posts list, because the site has no `content/_index.md`.

## Results

All three: `hugo --gc` succeeds, no ERROR. Post URLs `/posts/:slug/`, the
`projects` and `tags` taxonomies, term pages with intro text and per-term RSS
are identical to PaperMod (page/1 redirect stubs are PaperMod-only). SVG
diagram renders in all three.

WARN lines:
- PaperMod: two Hugo-deprecation warnings from the theme (`.Language.LanguageDirection`, `.Language.LanguageCode`).
- Blowfish: none.
- Hextra: `found no layout file for "json" for kind "home"` (Hextra has no
  JSON home template; `/index.json` is not produced; it is only used by PaperMod search).

### Blowfish (colour scheme `ocean`, home layout `custom`)
- Works: menu, search (Cmd-K, uses the JSON output), light/dark following the
  system plus a toggle, TOC (right sidebar), reading time, tags, code copy,
  giscus (root partial), `homeInfoParams` intro, project term pages with intro text.
- Gaps: giscus theme is fixed to `preferred_color_scheme`, so the manual toggle does
  not re-theme comments. Author is configured in
  `languages.en.toml` (Blowfish reads `params.author` as a map). `showEdit`
  link is an icon. Giscus/GoatCounter were not exercised (offline).
- Effort to switch: low, about an hour (move config into `hugo.toml`, keep 3 partial overrides).

### Hextra (blog layout via config `cascade type = "blog"`)
- Works: menu, flexsearch, light/dark following the system plus a toggle, TOC,
  code copy, giscus (built in, theme follows the toggle), term pages with intro.
- Gaps: no reading time (not supported by the theme). Home page has no
  built-in post list outside a `content/_index.md` with its home layout, so the
  list is a partial override. Needs `security.http.urls` for jsdelivr (FlexSearch and
  medium-zoom are fetched at build time, so builds need network). Copyright
  footer is theme i18n only (disabled). `params.author` is a map in this
  overlay. Blog `type` comes from a config cascade, so a switch should either keep
  that or set `type: blog` in content later.
- Effort to switch: low to medium; reading time would need a small override.
