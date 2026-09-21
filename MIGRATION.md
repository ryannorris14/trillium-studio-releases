# MIGRATION.md — trillium-studio-releases

Written 2026-09-21 as part of the Windows → Linux desktop migration. This
repo is a **release channel**, not an app: it holds only a static download
page (`index.html`) and receives compiled installer binaries as GitHub
Release assets. There is no source code, no build step, and no student data
in this repo.

## What this repo is

- Public GitHub repo: `ryannorris14/trillium-studio-releases`, default
  branch `main`.
- Serves the public download page for **Trillium Studio** (the Trillium
  Academy schedule-building desktop app) via GitHub Pages:
  `https://ryannorris14.github.io/trillium-studio-releases/` — built from
  branch `main`, path `/` (confirmed via `gh api repos/.../pages`).
- `index.html` is a static, self-contained page (inline CSS/JS, Google Fonts
  preconnect). At load it:
  - Detects the visitor's OS/chip (`navigator.userAgentData` /
    `navigator.platform`, with explicit handling for the fact that
    `navigator.platform` reports `MacIntel` on Apple Silicon too) to badge
    the "for this computer" download card. Fails safe to no badge if it
    can't tell.
  - Calls `https://api.github.com/repos/ryannorris14/trillium-studio-releases/releases/latest`
    to display live version numbers next to each download button. Static
    fallback if the fetch fails.
  - Download buttons point at
    `.../releases/latest/download/<asset>.zip` (zips, not bare `.exe`/`.dmg`,
    because browsers block direct exe/dmg downloads).
- The actual app source lives in a **separate private repo**,
  `ryannorris14/trillium-studio` (nested locally at
  `C:\Users\Ryan Norris\trillium\trillium-scheduler\studio\`, default branch
  `master`). See that project's own migration manifest
  (`trillium-scheduler.md`) for its stack — cited here, not duplicated.

## How releases are produced (documented in the sibling repo, not here)

Per the `trillium-scheduler` migration manifest (section on `studio/`):

- Installers are built by a GitHub Actions workflow,
  `studio/.github/workflows/release.yml`, in the **private** `trillium-studio`
  repo, triggered by pushing a `v*` tag (from any OS, including Linux — the
  trigger is just a git tag push).
- That workflow runs on `windows-latest`, `macos-latest`, and
  `macos-15-intel` GitHub-hosted runners:
  - Windows: PyInstaller (`studio/packaging/trillium.spec`) + Inno Setup 6
    (`installer.iss`) via `build.ps1` → `TrilliumStudio-Setup.exe`.
  - Mac: `build-mac.sh` → arm64 and x86_64 `.dmg`.
  - The Node/npm SPA under `studio/app/` is built first (`npm run build`)
    and packaged into each installer.
- The workflow then publishes/uploads the built installers (plus zipped
  copies) as assets on a GitHub Release in **this** repo
  (`trillium-studio-releases`), tagged to match.
- **None of this build tooling lives in this repo**, and none of it needs to
  run on the new Linux box — it's GitHub-hosted Windows/Mac CI. This repo's
  only "build step" is that GitHub Pages serves `index.html` as-is.

## Current release state (as of 2026-09-21)

- Latest release: `v0.1.10` (2026-08-06), assets: `TrilliumStudio-Setup.exe`,
  `TrilliumStudio-Setup.zip`, `TrilliumStudio-mac-arm64.dmg`,
  `TrilliumStudio-mac-arm64.zip`, `TrilliumStudio-mac-x86_64.dmg`,
  `TrilliumStudio-mac-x86_64.zip`.
- 11 releases total on the remote: `v0.1.0` through `v0.1.10` (tags exist
  only on the GitHub remote; a fresh clone must `git fetch --tags` to see
  them locally — plain `git clone` does pull tags by default, so this is
  only a note for anyone who cloned with `--no-tags`).
- Local working tree was clean and already in sync with `origin/main`
  (0 ahead / 0 behind) when this migration pass ran — nothing to commit or
  push for repo content itself.

## Fresh-clone steps (Linux)

```bash
git clone https://github.com/ryannorris14/trillium-studio-releases.git
cd trillium-studio-releases
git fetch --tags   # picks up v0.1.0 .. v0.1.10 (release tags)
```

No install step. To preview the download page locally:

```bash
python3 -m http.server 8000   # then open http://localhost:8000/index.html
```

(The live page's version numbers come from a `fetch()` to the GitHub API at
page-load time, so the API-driven version text will populate even when
served locally, as long as the machine has internet access.)

## To cut a new release (from Linux, once `trillium-studio` is restored)

1. Land the change in the private `trillium-studio` repo (source of truth
   for app code — this repo has none).
2. Tag it `vX.Y.Z` and push the tag: `git tag vX.Y.Z && git push origin vX.Y.Z`.
3. GitHub Actions builds Windows + Mac installers on hosted runners
   (nothing to run locally, no Windows/Mac machine needed).
4. The workflow publishes the built installers as a GitHub Release in
   **this** repo. Confirm with `gh release view vX.Y.Z -R
   ryannorris14/trillium-studio-releases`.
5. `index.html` requires no changes per release — it always points at
   `.../releases/latest/download/...`, which GitHub resolves dynamically.

## Windows-isms

None found in this repo itself. `index.html` is plain static HTML/CSS/JS
with no OS-specific paths, no `.bat`/`.ps1`/`.cmd` scripts, and no
Windows-only build outputs — those live in the sibling private repo and are
built by GitHub-hosted CI runners, not locally. Line endings were not
audited here as trivial (2 small text files); if it matters, check with
`file index.html README.md` and normalize with `git config core.autocrlf`
as needed on the new machine.

## Known bugs / open items

None documented in this repo (no issue tracker entries found via
`gh api .../pages` and repo metadata; `open_issues_count: 0`). Nothing
in-flight — working tree was clean, no stash, no other local branches.

## Secrets

None found as files (no `.env`, no key files) and nothing referenced by
`index.html` (it calls only the unauthenticated public GitHub REST API).
The private `trillium-studio` repo's release workflow almost certainly uses
a GitHub Actions secret (e.g., a `GITHUB_TOKEN`/PAT with cross-repo release
permissions) to publish assets here — that secret lives in that repo's
Actions settings on GitHub, not in any local file, and is out of scope for
this repo's migration.
