---
description: Publish a project to GitHub — secret scan, push, Pages via Actions, README, and repo About
argument-hint: "[repo URL or owner/name] [optional notes, e.g. site folder]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch
---

# Publish to GitHub

Publish the current project to GitHub end to end: secret scan → push code → GitHub Pages via Actions → screenshots of the live site → README → repo About with the Pages link.

**Target repo (from the user):** $ARGUMENTS

If `$ARGUMENTS` is empty, check `git remote -v` for an existing `origin`. If there is no remote and no argument, ask the user for the repo URL or `owner/name` before doing anything else.

## Ground rules

- Run the secret scan (Step 1) **before** the first push. Never push code that has not been scanned.
- Stop and ask the user before: force-pushing, rewriting history, deleting anything, or making a private repo public.
- Prefer the `gh` CLI. If `gh` is missing or not authenticated (`gh auth status`), tell the user how to fix it (`brew install gh && gh auth login`) and fall back to plain `git` for the parts that don't need the API.
- Report what you actually did at the end, including anything you skipped and why.

---

## Step 1 — Scan for sensitive data (blocking)

Scan everything that would be committed, plus files already tracked by git.

1. List candidate files: `git status --porcelain` and `git ls-files`. Also check for files that are staged but should not be.
2. Grep the working tree (excluding `.git/`, `node_modules/`, build output) for high-signal patterns:
   - Cloud keys: `AKIA[0-9A-Z]{16}`, `ASIA[0-9A-Z]{16}`, `AIza[0-9A-Za-z_-]{35}`
   - Provider tokens: `sk-ant-`, `sk-[A-Za-z0-9]{20,}`, `ghp_`, `gho_`, `ghs_`, `github_pat_`, `xox[baprs]-`, `glpat-`, `SG\.`, `AC[0-9a-f]{32}`
   - Generic assignments: `(api[_-]?key|secret|passwd|password|token|client[_-]?secret|access[_-]?key|auth)\s*[:=]\s*["'][^"']{8,}`
   - Private keys / certs: `-----BEGIN (RSA |EC |OPENSSH |PGP |DSA )?PRIVATE KEY-----`, `-----BEGIN CERTIFICATE-----`
   - Connection strings: `(postgres|mysql|mongodb\+srv|redis|amqp)://[^\s"']*:[^\s"']*@`
   - JWTs: `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.`
   - PII-ish: email addresses in config/data files, phone numbers, national ID numbers in fixtures
3. Flag risky **filenames** regardless of content: `.env*` (except `.env.example`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks`, `*.keystore`, `id_rsa*`, `*.ovpn`, `credentials`, `*credentials*.json`, `serviceAccount*.json`, `*.kdbx`, `*.sqlite`/`*.db` with user data, `.npmrc`/`.pypirc` with tokens, `*.mobileprovision`, cloud config dirs (`.aws/`, `.ssh/`, `.gnupg/`).
4. If `gitleaks` is installed, also run `gitleaks detect --no-banner` and fold its findings in.
5. Check history too, since a push publishes it: `git log --all --diff-filter=A --name-only --pretty=format:` for any of the risky filenames above.

**Triage:**
- Test fixtures, obvious placeholders (`your-key-here`, `xxx`, `changeme`), and public keys are not findings — say so and move on.
- For each real finding, report `file:line`, what it looks like, and the fix.

**If anything real is found, STOP.** Show the user the list and offer:
- add to `.gitignore` and `git rm --cached` (if untracked/newly added),
- replace with an env var + `.env.example`,
- rotate the credential (always recommend this for anything that was ever committed),
- history rewrite with `git filter-repo` / BFG (only with explicit approval).

Do not continue to Step 2 until the user resolves them or explicitly says to proceed anyway. If clean, say so in one line and continue.

While here, make sure `.gitignore` covers the usual suspects for this project's stack (`.env`, `node_modules/`, `dist/`, `__pycache__/`, `.DS_Store`, IDE dirs, credential files).

## Step 2 — Upload the code

1. `git rev-parse --is-inside-work-tree` — if not a repo, `git init` and set the branch to `main`.
2. Resolve the target repo from `$ARGUMENTS` (full URL, `owner/name`, or existing `origin`).
   - If the repo does not exist on GitHub, ask whether to create it and whether **public or private** (Pages on a private repo needs a paid plan — flag that), then `gh repo create <owner/name> --source=. --remote=origin --<public|private>`.
   - If `origin` exists but differs from `$ARGUMENTS`, ask before changing it.
3. Stage and commit only what belongs (`git add -A` after the scan; review `git status` first). Write a clear commit message. Do not add Claude attribution unless the user asks.
4. Push: `git push -u origin main`. If the remote has commits you don't, pull/rebase rather than force-pushing.
5. Confirm with `git log --oneline -1` and the repo URL.

## Step 3 — GitHub Pages via GitHub Actions

1. Work out what to publish: a static site at the repo root (`index.html`), a `docs/` folder, or a build (Vite/Next/Astro/Jekyll/MkDocs/Hugo — detect from `package.json`, config files, or `Gemfile`).
2. Create or update `.github/workflows/pages.yml`. If one already exists, **edit it** rather than replacing it wholesale, and preserve any customizations. Baseline for a static site:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .          # or ./docs, or ./dist for a build
      - id: deployment
        uses: actions/deploy-pages@v4
```

   For a build-based site, insert `actions/setup-node@v4` + install/build steps before the upload and point `path` at the build output. Set the framework's base path to `/<repo-name>/` when the site is served from a project subpath.
3. Enable Pages with the Actions source: `gh api -X POST repos/{owner}/{repo}/pages -f build_type=workflow` (use `-X PUT` if it already exists). Fall back to telling the user to set Settings → Pages → Source = GitHub Actions.
4. Commit and push the workflow, then watch the run: `gh run watch` or `gh run list --limit 1`. If it fails, read the logs (`gh run view --log-failed`), fix, and re-run — don't hand back a red build.
5. Capture the live URL: `gh api repos/{owner}/{repo}/pages --jq .html_url` (usually `https://<owner>.github.io/<repo>/`).

## Step 4 — Screenshots of the live site

Capture the deployed site with the Playwright MCP server and put the images in the README.

Requires the `playwright` MCP server (`.mcp.json`):

```json
{ "mcpServers": { "playwright": { "command": "npx", "args": ["-y", "@playwright/mcp@latest"] } } }
```

If it is not configured or its tools are unavailable, say so and skip to Step 5 rather than inventing image links.

1. `browser_resize` to a desktop viewport — 1440x900 is a good default.
2. `browser_navigate` to the **live Pages URL** from Step 3, not a `file://` path or `localhost`, so the screenshot shows what visitors actually get. If the deploy just finished, the URL can 404 for a minute — retry before giving up.
3. `browser_take_screenshot` with `scale: "device"` for a crisp image. Use `fullPage: true` for a page whose content runs past the fold; for a short page, shrink the viewport height instead and take a viewport shot, so the image is not mostly blank space.
4. Repeat for each page worth showing (landing page, plus the main app or demo pages). Two or three images is plenty.
5. The MCP server writes into the current directory. Move the files to a stable home — `docs/screenshots/` — with descriptive names (`kanban-board.png`, not `page-2026-01-01T00-00-00.png`), and add its scratch dir to `.gitignore`:

```
.playwright-mcp/
```

6. Check each image before committing it: open it and confirm it rendered (no error page, no half-loaded layout, no blank viewport) and shows nothing sensitive — real names, internal URLs, tokens on screen, a logged-in session. Re-take or crop anything that does.
7. Reference them from the README with relative paths and real alt text describing what is in the shot:

```markdown
![Kanban board with four columns of task cards](docs/screenshots/kanban-board.png)
```

Keep the images reasonably sized (a few hundred KB each); a PNG over ~1 MB should be resized or saved as JPEG/WebP.

## Step 5 — README

Create `README.md`, or update the existing one in place (keep the user's wording and structure; don't flatten it into a template).

Base it on what the code actually does — read the source, don't guess. Include:
- Project title and a one/two-sentence description
- Live demo link (the Pages URL from Step 3)
- Features / what's inside
- Install and usage, with real commands for this project
- Configuration, including any env vars, referencing `.env.example` — never real values
- Project structure (only if it helps)
- License, if the repo has one

No invented badges, benchmarks, or features. Commit and push.

## Step 6 — Repo About (description, homepage, topics)

Set the About sidebar, with the Pages URL as the homepage:

```bash
gh repo edit <owner>/<repo> \
  --description "<one-line description>" \
  --homepage "<pages-url>" \
  --add-topic <topic1> --add-topic <topic2>
```

Pick a concise description matching the README, and 3–6 relevant topics from the stack. If `gh repo edit` is unavailable, use `gh api -X PATCH repos/{owner}/{repo} -f description=... -f homepage=...`. Also enable "Use your GitHub Pages website" in About if it isn't already reflected by the homepage field.

## Step 7 — Verify and report

- `gh repo view --web` URL, Pages URL, and the workflow run status
- Fetch the Pages URL to confirm it serves (it can take a minute after first deploy)
- Summarize: what was pushed, secret-scan result, Pages status + link, screenshots captured, README changes, About fields set, and anything left for the user to do (e.g. rotate a key, pick a license, custom domain).
