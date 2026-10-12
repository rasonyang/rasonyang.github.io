# rasonyang.dev

Source for [rasonyang.dev](https://rasonyang.dev): engineering notes on
[AICC](https://github.com/rasonyang/ai-native-callcenter),
[FreeSBC](https://github.com/rasonyang/freesbc),
[Cascade Realtime Gateway](https://github.com/rasonyang/cascade-realtime-gateway) and
[Web SIP Phone](https://github.com/rasonyang/web-sip-phone).

Built with [Hugo](https://gohugo.io) (pinned in `.github/workflows/pages.yml`)
and the [Blowfish](https://github.com/nunocoracao/blowfish) theme
(git submodule at `themes/blowfish`, pinned by commit). Deployed to GitHub Pages by GitHub Actions
on every push to `main`.

## Write

```sh
git clone --recurse-submodules git@github.com:rasonyang/rasonyang.github.io.git
hugo new content posts/<slug>.md      # created with draft: true
hugo server -D                        # preview drafts at http://localhost:1313
```

One post per pull request. Remove `draft: true` and merge to publish.

- `projects:` in front matter puts the post on a project page
  (`/projects/<slug>/`, with its own RSS feed). Values: `aicc`, `freesbc`,
  `cascade-realtime-gateway`, `web-sip-phone`.
- Optional `featured.jpg` (1200x630) in the post's bundle becomes its card and
  social image; without it the site default `assets/img/default-feature.jpg`
  is used.
- New tag? Add `content/tags/<tag>/_index.md` with the display `title`
  (e.g. `SIP`, `WebRTC`) so it is not auto-capitalised.
- Slugs are permanent once published.
- Link code by commit SHA, not `main`.
- Diagrams are committed as SVG, not rendered at runtime.
- Post ideas go in `notes/ideas.md`.

## Configure once

- Comments: install the [giscus](https://giscus.app) app on this repo, enable
  Discussions with a `Comments` category, then set `giscus.repoId` and
  `categoryId` in `config/_default/params.toml`.
- Analytics: set `analytics.goatcounter.code` in `config/_default/params.toml`.
