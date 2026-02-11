# Duckquill Quick Notes

## Install / Structure
- Theme as submodule: `git submodule add https://codeberg.org/daudix/duckquill.git themes/duckquill`
- Config: `zola.toml` -> `theme = "duckquill"`, `compile_sass = true`, `generate_feeds = true`, `build_search_index = true`, `taxonomies = [{ name = "tags", feed = true }]`.
- Content: home `content/_index.md`, blog `content/blog/_index.md` + posts under `content/blog/*.md`.
- Run/Build: `zola serve` for preview, `zola build` -> `public/` for deploy.

## Key Config (`zola.toml`)
- Site meta: `title`, `base_url`, `description`, `default_language`, `author`.
- Markdown: `[markdown]` with `smart_punctuation`, `github_alerts`. (Highlighting uses defaults.)
- Extra (nav/footer/styles) example:
  ```toml
  [extra]
  accent_color = "#627eea"
  accent_color_dark = "#93b4ff"
  show_copy_button = true
  show_reading_time = true
  show_share_button = true
  show_backlinks = true

  [extra.nav]
  show_feed = true
  show_theme_switcher = true
  show_repo = true
  links = [
    { url = "@/blog/_index.md", name = "Blog" },
    { url = "https://github.com", name = "GitHub" }
  ]

  [extra.footer]
  links = [
    { url = "@/blog/_index.md", name = "Blog" }
  ]
  socials = [
    { url = "https://github.com", name = "GitHub", icon = "<svg...>" }
  ]
  ```

## MODS (https://duckquill.daudix.one/mods/)
- Purpose: optional Sass modules to tweak visuals/layout without editing theme core.
- How to use:
  1) Create `sass/mods.scss`, import needed mods, e.g.  
     `@use "../themes/duckquill/sass/mods/classic-article-list";`
  2) Add custom CSS (e.g., background image) in the same file.
  3) Register the compiled CSS in `zola.toml`:  
     `[extra] styles = ["mods.css"]`
- Common mods:
  - `classic-article-list`: older-style post list.
  - `classic-nav` / `sticked-nav`: traditional nav / sticky top (not together).
  - `classic-del`: simpler strikethrough.
  - `modern-headings`: toned-down headings.
  - `modern-hr`: simpler hr.
  - `no-edge-highlight`: remove edge highlight on translucent elements.
  - Background image: set `body { background-image: var(--bg-overlay), url(...); }` and tweak `--bg-overlay` via `_variables.scss`.

## Tips
- Nav/footer won’t show unless `[extra.nav]` / `[extra.footer]` links are set.
- For deploy with submodules: in GitHub Actions, use `actions/checkout` with `submodules: true`.
- For i18n: add `i18n/<lang>.toml` following theme demo if needed.

## Options cheat sheet (from duckquill.daudix.one)
- Global overrides (front matter `[extra]` per page/section): `default_theme` (light/dark), `accent_color`, `accent_color_dark`, `emoji_favicon`, `styles` (extra CSS in `static/`), `scripts` (extra JS), `katex` (LaTeX), `toc`, `toc_inline`, `toc_ordered`, `toc_sidebar`, `card` (share preview off).
- Asset filenames (colocated with page): `apple_touch_icon`, `favicon`, `card`.
- Notices (front matter `[extra]`): `archive`, `trigger`, `disclaimer`, `go_to_top`.
- Post-specific (front matter): `banner` (2:1 ~1920x960), `banner_pixels` (nearest-neighbor), `archived`/`featured`/`hot`/`poor` (visual badges).
- Comments (front matter `[extra.comments]`): `host`, `user`, `id` (Mastodon-powered comments/backlinks).
- Localization: add `i18n/<lang>.toml` (ISO 639-1/BCP 47). Copy `en.toml` from theme to override English strings.
- Custom styles: put Sass/CSS in `sass/*.scss` → enable via `[extra] styles = ["mods.css"]` or per-page `[extra].styles`.
- Accent color: `[extra] accent_color = "#3584e4"` and optional `accent_color_dark`.
- Favicons: `static/favicon.png`, `static/apple-touch-icon.png` (APNG allowed).
