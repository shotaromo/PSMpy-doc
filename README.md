# PSMpy-doc

Public documentation site for **PSMpy** (a Pythonic rebuild of the GAMS
PowerSystemModel as a GAMSpy model). Built with [MkDocs](https://www.mkdocs.org/)
+ [Material](https://squidfunk.github.io/mkdocs-material/); published to GitHub
Pages.

> The PSMpy model itself lives in the (private) `PSMpy` repo. This repo holds
> only the **public, user-facing** docs (overview + user guide) and is included
> there as a submodule (mirroring `PowerSystemModel` / `PowerSystemModel-doc`).
> Internal design docs (specs, ADRs, status, deviations) stay in the model repo.

## Build locally

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve     # live preview at http://127.0.0.1:8000
mkdocs build     # static site -> site/
```

Bilingual (EN / 日本語) via `mkdocs-static-i18n` (suffix structure:
`page.md` / `page.ja.md`). Math renders via KaTeX/MathJax (`pymdownx.arithmatex`).

## Publish

`.github/workflows/ci.yml` runs `mkdocs gh-deploy` on push to `main` (or manual
dispatch), publishing to the `gh-pages` branch. Set repo **Settings → Pages →
Deploy from a branch → `gh-pages` / root**. Site:
<https://shotaromo.github.io/PSMpy-doc/>.
