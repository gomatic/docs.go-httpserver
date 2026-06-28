# `docs.go-httpserver` — public documentation repository

This is the **public** documentation site for [`gomatic/go-httpserver`](https://github.com/gomatic/go-httpserver), created from [`nicerobot/template.repo-docs`](https://github.com/nicerobot/template.repo-docs). It is published to GitHub Pages.

## Conventions

- **Everything here is public.** Documentation is built as a self-contained [Hugo](https://gohugo.io) site at the repo root. Markdown content goes under [`content/`](../content/).
- **Private content does NOT live here.** Ideas, tasks, internal notes, and specs live in the project's private hub repo, never in this repo. There is no `public/`/`private/` split — a docs repo is wholly public.
- **Cross-links stay relative.** Link between pages with the target's source path (e.g. `[Usage](usage.md)`); Hugo's embedded link render hook (`markup.goldmark.renderHooks.link`, enabled in [`../hugo.json`](../hugo.json)) resolves them to the right URLs, so the same links also work when browsing the markdown on GitHub.
- **Keep docs in sync with the source.** When [`gomatic/go-httpserver`](https://github.com/gomatic/go-httpserver)'s API changes, update [`content/`](../content/) to match.
- Preview locally with `make serve`; build with `make build`.
