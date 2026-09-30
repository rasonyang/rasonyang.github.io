# rasonyang.dev

Source for [rasonyang.dev](https://rasonyang.dev): engineering notes on
[AICC](https://github.com/rasonyang/ai-native-callcenter),
[FreeSBC](https://github.com/rasonyang/freesbc),
[Cascade Realtime Gateway](https://github.com/rasonyang/cascade-realtime-gateway) and
[Web SIP Phone](https://github.com/rasonyang/web-sip-phone).

Built with [Hugo](https://gohugo.io) (pinned in `.github/workflows/pages.yml`)
and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme
(git submodule, pinned by commit). Deployed to GitHub Pages by GitHub Actions
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
- Slugs are permanent once published.
- Link code by commit SHA, not `main`.
- Diagrams are committed as SVG, not rendered at runtime.
- Post ideas go in `notes/ideas.md`.

## Configure once

- Comments: install the [giscus](https://giscus.app) app on this repo, enable
  Discussions with a `Comments` category, then set `params.giscus.repoId` and
  `categoryId` in `hugo.toml`.
- Analytics: set `params.analytics.goatcounter.code` in `hugo.toml`.
