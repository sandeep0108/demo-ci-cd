# GitHub Pages CI/CD Demo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal static site that auto-deploys to GitHub Pages on every push to `main`, with the deploy timestamp and commit SHA visible on the page so the CI/CD loop is observable.

**Architecture:** Three static files (`index.html`, `style.css`, `README.md`) plus one GitHub Actions workflow (`.github/workflows/deploy.yml`). The workflow runs `sed` to substitute two placeholder tokens (`__DEPLOY_TIME__`, `__COMMIT_SHA__`) inside `index.html` at deploy time, then publishes the project root to GitHub Pages via `actions/deploy-pages@v4`.

**Tech Stack:** Plain HTML/CSS, GitHub Actions, GitHub Pages. No build tools, no dependencies, no runtime.

**Testing approach:** This project has no application logic to unit-test. Verification is manual: render the HTML in a browser locally, run the workflow's `sed` substitution by hand on a temp copy of `index.html` to confirm the placeholder swap works, and lint the YAML. End-to-end verification only happens once the user pushes to GitHub and watches the Actions run go green.

---

### Task 1: Create the HTML page with placeholder tokens

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with the two placeholder tokens**

Create `/var/www/html/demo-ci-cd/index.html` with exactly this content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CI/CD Demo</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <main>
        <h1>CI/CD Demo</h1>
        <p>This page is auto-deployed from GitHub to GitHub Pages on every push to <code>main</code>.</p>
        <p>The values below are injected by the GitHub Actions workflow at deploy time. If they update after a push, the pipeline is working.</p>
        <dl>
            <dt>Last deploy</dt>
            <dd id="deploy-time">__DEPLOY_TIME__</dd>
            <dt>Commit</dt>
            <dd id="commit-sha"><code>__COMMIT_SHA__</code></dd>
        </dl>
    </main>
</body>
</html>
```

The two strings `__DEPLOY_TIME__` and `__COMMIT_SHA__` must appear **exactly** as written — the workflow's `sed` commands match them literally.

- [ ] **Step 2: Verify it opens in a browser**

Run: `xdg-open /var/www/html/demo-ci-cd/index.html` (or open the file path in any browser manually).

Expected: A page that says "CI/CD Demo" with the literal placeholder tokens visible (e.g. "Last deploy: __DEPLOY_TIME__"). The unstyled placeholders are correct at this stage.

- [ ] **Step 3: Commit**

```bash
cd /var/www/html/demo-ci-cd
git add index.html
git commit -m "feat: add index.html with deploy-time placeholders"
```

---

### Task 2: Add minimal CSS

**Files:**
- Create: `style.css`

- [ ] **Step 1: Create `style.css`**

Create `/var/www/html/demo-ci-cd/style.css` with exactly this content:

```css
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: system-ui, -apple-system, sans-serif;
    background: #0f172a;
    color: #e2e8f0;
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 2rem;
}

main {
    max-width: 640px;
    width: 100%;
    background: #1e293b;
    padding: 2.5rem;
    border-radius: 12px;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
}

h1 {
    margin-top: 0;
    color: #38bdf8;
}

p {
    line-height: 1.6;
}

dl {
    margin-top: 2rem;
    padding-top: 1.5rem;
    border-top: 1px solid #334155;
}

dt {
    font-size: 0.85rem;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: #94a3b8;
    margin-top: 1rem;
}

dd {
    margin: 0.25rem 0 0 0;
    font-size: 1.1rem;
}

code {
    font-family: ui-monospace, "SF Mono", Menlo, monospace;
    background: #0f172a;
    padding: 0.15rem 0.4rem;
    border-radius: 4px;
    font-size: 0.95em;
}
```

- [ ] **Step 2: Reload the page in the browser and confirm it is styled**

Refresh the browser tab from Task 1. Expected: dark navy background, centered card, light blue heading. The placeholders `__DEPLOY_TIME__` and `__COMMIT_SHA__` are still literal text — that is correct.

- [ ] **Step 3: Commit**

```bash
cd /var/www/html/demo-ci-cd
git add style.css
git commit -m "feat: add minimal styling"
```

---

### Task 3: Verify the placeholder-substitution logic before encoding it in the workflow

**Files:**
- (no files modified — this is a manual verification step using a temp copy)

The workflow will run two `sed -i` substitutions inside the runner. Before committing those into YAML, run them by hand on a temp copy to make sure the substitutions hit and the page output is what we expect. Catching a typo here is much faster than catching it via a failed Actions run.

- [ ] **Step 1: Make a temp copy and run the substitution commands**

```bash
cd /var/www/html/demo-ci-cd
cp index.html /tmp/index-test.html
sed -i "s|__DEPLOY_TIME__|$(date -u +'%Y-%m-%d %H:%M:%S UTC')|g" /tmp/index-test.html
sed -i "s|__COMMIT_SHA__|abc1234|g" /tmp/index-test.html
```

- [ ] **Step 2: Confirm the placeholders were replaced**

```bash
grep -c '__DEPLOY_TIME__\|__COMMIT_SHA__' /tmp/index-test.html
```

Expected output: `0` (zero matches — both placeholders gone).

```bash
grep -E 'Last deploy|Commit' /tmp/index-test.html
```

Expected: lines showing the real timestamp and `abc1234` in place of the placeholders.

- [ ] **Step 3: Clean up the temp file**

```bash
rm /tmp/index-test.html
```

No commit — this task created no files.

---

### Task 4: Create the GitHub Actions workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Create the workflow file**

Create `/var/www/html/demo-ci-cd/.github/workflows/deploy.yml` with exactly this content:

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
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Inject deploy timestamp and commit SHA
        run: |
          DEPLOY_TIME="$(date -u +'%Y-%m-%d %H:%M:%S UTC')"
          SHORT_SHA="${GITHUB_SHA::7}"
          sed -i "s|__DEPLOY_TIME__|${DEPLOY_TIME}|g" index.html
          sed -i "s|__COMMIT_SHA__|${SHORT_SHA}|g" index.html

      - name: Verify placeholders were replaced
        run: |
          if grep -q '__DEPLOY_TIME__\|__COMMIT_SHA__' index.html; then
            echo "ERROR: placeholders still present in index.html after substitution"
            exit 1
          fi

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: .

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Notes for the engineer:
- `permissions:` must include `pages: write` and `id-token: write` — `actions/deploy-pages@v4` uses OIDC and will fail without `id-token: write`.
- `concurrency: group: pages` prevents two deploys racing each other; `cancel-in-progress: true` means a fast follow-up push supersedes the previous run.
- The "Verify placeholders were replaced" step is a guardrail — if someone renames a placeholder in `index.html` without updating the `sed` line, the workflow fails loudly instead of silently shipping a page with literal `__DEPLOY_TIME__` text.
- `actions/upload-pages-artifact@v3` with `path: .` uploads the whole repo root. That is fine for this demo (only `index.html`, `style.css`, plus a few markdown files; no large assets). If we add many files later, we should switch to a dedicated `dist/` directory.

- [ ] **Step 2: Lint the YAML for syntax errors**

```bash
python3 -c "import yaml; yaml.safe_load(open('/var/www/html/demo-ci-cd/.github/workflows/deploy.yml'))" && echo OK
```

Expected output: `OK`

If `python3` is unavailable, use any other YAML validator (`yamllint`, an online validator, or VS Code's built-in YAML extension).

- [ ] **Step 3: Commit**

```bash
cd /var/www/html/demo-ci-cd
git add .github/workflows/deploy.yml
git commit -m "ci: add GitHub Pages deploy workflow"
```

---

### Task 5: Create the README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Create `README.md`**

Create `/var/www/html/demo-ci-cd/README.md` with exactly this content:

````markdown
# demo-ci-cd

A minimal demo of automated deployment from GitHub to GitHub Pages using GitHub Actions.

Every push to `main` triggers a workflow that:
1. Substitutes the current UTC timestamp and the short commit SHA into `index.html`.
2. Publishes the result to GitHub Pages.

The page itself displays those two values, so any successful deploy is immediately visible after a refresh.

## One-time setup

1. Create a **public** GitHub repository (free GitHub Pages requires public repos on free accounts).
2. Push this project to the new repo's `main` branch:
   ```bash
   git remote add origin https://github.com/<your-username>/<your-repo>.git
   git push -u origin main
   ```
3. In your GitHub repo, go to **Settings → Pages**.
4. Under **Build and deployment**, set **Source** to **GitHub Actions**.

That's it. The next push to `main` (or a manual run from the Actions tab) will deploy.

## Verifying the pipeline works

1. Open the **Actions** tab on GitHub. You should see a "Deploy to GitHub Pages" run after each push.
2. Once the run is green, visit `https://<your-username>.github.io/<your-repo>/`.
3. The page shows the deploy timestamp and commit SHA. Push another commit and the values update on refresh.

## Files

- `index.html` — the page itself, with `__DEPLOY_TIME__` and `__COMMIT_SHA__` placeholders.
- `style.css` — minimal styling.
- `.github/workflows/deploy.yml` — the GitHub Actions workflow.
- `docs/superpowers/specs/` — design doc for this project.
- `docs/superpowers/plans/` — the implementation plan.

## Troubleshooting

- **Workflow fails at "Deploy to GitHub Pages" with a permissions error:** confirm Settings → Pages → Source is set to "GitHub Actions" (not "Deploy from a branch").
- **Page 404s:** GitHub Pages can take a minute to propagate after the first successful deploy. Wait 60s and retry.
- **Placeholders still visible on the live page:** the workflow's "Verify placeholders were replaced" step would have failed the build before reaching deploy. Check the Actions log for the failing step.
````

- [ ] **Step 2: Commit**

```bash
cd /var/www/html/demo-ci-cd
git add README.md
git commit -m "docs: add README with setup and verification steps"
```

---

### Task 6: Final local verification

**Files:**
- (none — this is a sanity check before the project is handed to the user)

- [ ] **Step 1: Confirm the working tree is clean and the log is sensible**

```bash
cd /var/www/html/demo-ci-cd
git status
git log --oneline
```

Expected `git status` output: `nothing to commit, working tree clean`.
Expected `git log --oneline` output: 5 commits, in order:
- `docs: add README with setup and verification steps`
- `ci: add GitHub Pages deploy workflow`
- `feat: add minimal styling`
- `feat: add index.html with deploy-time placeholders`
- `docs: add design for GitHub Pages CI/CD demo`

- [ ] **Step 2: Confirm the file tree matches the design**

```bash
cd /var/www/html/demo-ci-cd
find . -type f -not -path './.git/*' | sort
```

Expected output:
```
./.github/workflows/deploy.yml
./README.md
./docs/superpowers/plans/2026-05-01-github-pages-cicd-demo.md
./docs/superpowers/specs/2026-05-01-github-pages-cicd-demo-design.md
./index.html
./style.css
```

- [ ] **Step 3: Tell the user the project is ready and remind them of the GitHub-side setup**

The implementation is complete. The user still has to:
1. Create a public GitHub repo
2. `git remote add origin ...` and `git push -u origin main`
3. Settings → Pages → Source: GitHub Actions

Those steps are documented in `README.md`. End-to-end verification (workflow runs green, live page updates on push) only happens after the user completes them.

---

## Out of scope / not done

- No automated tests — the design explicitly lists testing as a non-goal.
- No custom domain.
- No build pipeline (no bundlers, no transpilers).
- No remote — `git remote add` is the user's responsibility because we do not know their GitHub username.
