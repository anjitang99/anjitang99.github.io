# Repository Guidelines

## Project Structure & Module Organization
- `_config.yml` controls site metadata, languages (`lang: ko`), analytics, and PWA switches—update it before publishing new sections.
- `_posts/` contains dated Markdown posts (`YYYY-MM-DD-title.md`) with Liquid front matter; drafts live temporarily under `_drafts/` until promoted.
- `_tabs/` defines static pages such as About or Categories, `_data/` stores navigation/localization YAML, and `_plugins/` houses lightweight Ruby helpers required by the Chirpy theme.
- `assets/` holds SCSS, JavaScript, and media (`assets/img/...`); keep post-specific images under `assets/img/<slug>/` and reference them with root-relative paths. `_site/` is the generated output—never edit or commit it.

## Build, Test, and Development Commands
```bash
bundle install                               # install Ruby dependencies from Gemfile
bundle exec jekyll serve --livereload        # local dev server at http://127.0.0.1:4000
JEKYLL_ENV=production bundle exec jekyll build # production build into _site/
bundle exec htmlproofer ./_site              # link & HTML validation suite
```
Run commands from the repo root; delete `_site/` if a stale build causes html-proofer noise.

## Coding Style & Naming Conventions
- Posts use YAML front matter with 2-space indentation, lowercase keys, and kebab-case slugs (`permalink: /posts/<slug>/`).
- Use descriptive H1 titles plus Korean/English summaries as in existing commits; prefer Markdown lists over raw HTML unless you need custom CSS hooks.
- Keep Liquid logic minimal in posts—extract repeated snippets into `_includes/` or `_tabs/` when possible.

## Testing Guidelines
Treat `bundle exec jekyll build` as a pre-flight check; builds must pass without warnings. After any layout, config, or asset change, run `bundle exec htmlproofer ./_site --disable-external true` to surface broken anchors. For JS/CSS tweaks, reload via `jekyll serve` and test both light/dark modes because PWA caching is enabled.

## Commit & Pull Request Guidelines
Recent history (`Binding 변환`, `ButtonStyle 포스팅`) shows short, topic-focused commit subjects, often Korean nouns/phrases without trailing punctuation. Follow that tone: imperative or descriptive, <= 50 characters, scoped to one logical change. PRs should include:
1. Summary of changes and affected directories.
2. Screenshots or GIFs when altering layouts or assets.
3. Confirmation that `jekyll build` and `htmlproofer` were run.
4. Links to related issues or content requests.

## Agent Communication
- Prefer Korean for commit messages, PR descriptions, and review feedback to stay consistent with the repository history and target audience.
- When referencing commands or filenames, keep the literal text in English (e.g., `bundle exec jekyll serve`) but surround it with Korean explanations so instructions remain clear to all collaborators.
