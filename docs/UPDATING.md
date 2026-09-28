# Keeping the blog up to date

This site is built from [Chirpy Starter](https://github.com/cotes2020/chirpy-starter) and uses the
[jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) gem. Updates are mostly handled by
[Renovate](https://docs.renovatebot.com/) (see [Automated updates](#automated-updates-renovate)); this page
describes what is tracked, how to review Renovate PRs, and how to update by hand.

> The `docs/` folder is excluded from the Jekyll build (`exclude` in `_config.yml`), so this file is not published.

## What needs updating

| Component | Defined in | Upstream reference |
| --- | --- | --- |
| Chirpy theme gem | `Gemfile` (`jekyll-theme-chirpy`) | [theme releases](https://github.com/cotes2020/jekyll-theme-chirpy/releases) |
| html-proofer | `Gemfile` | [rubygems](https://rubygems.org/gems/html-proofer) |
| Other gems (Jekyll, plugins, …) | resolved at install time (`Gemfile.lock` is not committed) | – |
| Ruby | `.ruby-version` (read by `ruby/setup-ruby` in CI) | [ruby-lang.org](https://www.ruby-lang.org/en/downloads/) |
| GitHub Actions | `.github/workflows/pages-deploy.yml` | starter's [`pages-deploy.yml`](https://github.com/cotes2020/chirpy-starter/blob/main/.github/workflows/pages-deploy.yml) |
| Static assets submodule | `assets/lib` (see `.gitmodules`) | [chirpy-static-assets](https://github.com/cotes2020/chirpy-static-assets) |
| Dev container image | `.devcontainer/devcontainer.json` | [devcontainers/jekyll](https://mcr.microsoft.com/en-us/artifact/mar/devcontainers/jekyll/about) |

Constraints to keep in mind:

- **Ruby must stay on 3.x.** The theme's gemspec declares `required_ruby_version = "~> 3.1"`, so Ruby 4 will not
  install it. Renovate is configured to ignore Ruby 4.
- **Follow the starter, not just the gem.** A theme release often comes with changes to the starter files
  (`_config.yml`, workflow, `Gemfile`, `_tabs/`, `.devcontainer/`). Compare the starter's tag for the new theme
  version with this repo when doing a minor/major theme update.

## Automated updates (Renovate)

Configuration lives in [`.github/renovate.json`](../.github/renovate.json):

- Runs **Monday before 06:00 (Europe/Zurich)**, max 5 open PRs, labelled `dependencies`.
- Tracks the `Gemfile`, `.ruby-version`, GitHub Actions and the `assets/lib` submodule.
- Because `Gemfile.lock` is not committed, Bundler updates use `rangeStrategy: bump`, i.e. the range in the
  `Gemfile` itself is raised (e.g. `~> 7.4` → `~> 7.6`).
- The theme gem and the static-assets submodule are grouped into one **"Chirpy theme"** PR.
- Minor/patch GitHub Actions updates are grouped into one **"GitHub Actions"** PR; major action bumps get their own PR.
- **Major** theme updates wait for approval on the **Dependency Dashboard** issue.

### One-time setup

1. Install the [Renovate GitHub App](https://github.com/apps/renovate) and grant it access to
   `egoorbis/egoorbis.github.io`.
2. Renovate detects `.github/renovate.json` and starts from there. The **Dependency Dashboard** issue lists all
   pending and scheduled updates; tick a checkbox there to run one right away.
3. Optional: to validate config changes locally run
   `npx --package renovate -- renovate-config-validator --strict`.

### Reviewing a Renovate PR

The deploy workflow runs only on `main`/`master`, so PR branches are **not** built in CI. Before merging:

1. Read the release notes Renovate links in the PR (for the theme also the
   [changelog](https://github.com/cotes2020/jekyll-theme-chirpy/blob/master/docs/CHANGELOG.md)).
2. For theme updates, diff the starter between the old and new tag, e.g.
   `https://github.com/cotes2020/chirpy-starter/compare/v7.4.1...v7.6.0`, and port relevant changes.
3. Build and test locally (see below), then merge. The merge triggers the deploy; check the
   **Build and Deploy** run in the Actions tab.

## Manual update

```bash
# 1. Theme and gems: raise the range in Gemfile, e.g.
#      gem "jekyll-theme-chirpy", "~> 7.6"
bundle update

# 2. Ruby: edit .ruby-version (stay on 3.x), then install that version locally

# 3. GitHub Actions: bump the @vN tags in .github/workflows/pages-deploy.yml
#    to match the starter's pages-deploy.yml

# 4. Static assets submodule (only needed if you use it locally)
git submodule update --init --remote assets/lib
```

### Verify locally

```bash
bundle install
bash tools/run.sh          # serve at http://127.0.0.1:4000 and click through
bash tools/test.sh         # production build + html-proofer, same as CI
```

Then commit, push to a branch, open a PR, and merge once it looks good.
