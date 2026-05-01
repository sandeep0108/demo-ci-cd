# GitHub Pages CI/CD Demo — Design

**Date:** 2026-05-01
**Status:** Approved (pending written-spec review)

## Goal

A minimal end-to-end demonstration of automated deployment from GitHub to a live URL using GitHub Actions and GitHub Pages. The demo must make the CI/CD loop *visible*: pushing a commit must produce an observable change on the deployed page.

## Non-Goals

- Build pipelines (no bundlers, transpilers, dependency installs)
- Tests (out of scope for a static "hello world"-class demo)
- Custom domains, HTTPS configuration, or Pages branch sources
- Any backend, database, or dynamic runtime

## Constraints

- No external services beyond GitHub itself
- No paid accounts, no credit card, no API keys
- Repo must be public (free GitHub Pages requires this for free accounts)

## Architecture

A static site of three files plus one workflow:

```
demo-ci-cd/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions workflow
├── index.html                  # Single page with two placeholders
├── style.css                   # Minimal styling
└── README.md                   # Setup instructions for the user
```

### Component: `index.html`

A single HTML page containing:
- A title heading ("CI/CD Demo")
- A short paragraph explaining what the page proves
- Two placeholder tokens that the workflow rewrites at deploy time:
  - `__DEPLOY_TIME__` — replaced with the UTC timestamp of the workflow run
  - `__COMMIT_SHA__` — replaced with the short Git SHA of the commit being deployed

The placeholders are the demonstration mechanism. Without them, "the page deployed" looks identical to "the page didn't deploy."

### Component: `style.css`

Minimal CSS — readable typography, centered content, a hint of color. Just enough that the page does not look broken. No frameworks, no resets, no preprocessor.

### Component: `.github/workflows/deploy.yml`

A single workflow with two jobs.

**Trigger:** `push` to branch `main`, plus `workflow_dispatch` for manual runs.

**Permissions:** `pages: write`, `id-token: write`, `contents: read` (the minimum required by `actions/deploy-pages`).

**Concurrency:** group `pages`, cancel in-progress runs that are superseded.

**Job 1 — `build`:**
1. `actions/checkout@v4`
2. Inline shell step: `sed -i` over `index.html` to replace `__DEPLOY_TIME__` with `$(date -u +'%Y-%m-%d %H:%M:%S UTC')` and `__COMMIT_SHA__` with `${GITHUB_SHA::7}`
3. `actions/upload-pages-artifact@v3` with the project root as the artifact path

**Job 2 — `deploy`:**
- `needs: build`
- `environment: github-pages` (required by Pages)
- Single step: `actions/deploy-pages@v4`

### Component: `README.md`

Tells the user, in numbered steps:
1. Create a public GitHub repo
2. Push this code to `main`
3. Open repo Settings → Pages, set Source to "GitHub Actions"
4. Watch the workflow run under the Actions tab
5. Visit `https://<username>.github.io/<repo-name>` and confirm the deploy timestamp matches the workflow run

## Data Flow

```
git push origin main
        │
        ▼
GitHub receives push event
        │
        ▼
deploy.yml workflow starts
        │
        ├─► build job: checkout → sed replace placeholders → upload artifact
        │
        ▼
deploy job: actions/deploy-pages publishes artifact to Pages
        │
        ▼
https://<user>.github.io/<repo>/ shows new timestamp + SHA
```

## Error Handling

The workflow has no application logic and nothing meaningful to recover from. Failure modes are all GitHub-side and surface as a red X in the Actions tab:
- Pages not enabled in repo settings → `actions/deploy-pages` fails with a clear error directing the user to Settings → Pages
- Repo is private on a free account → Pages publish fails; documented in README

No custom error handling is needed.

## Testing

No automated tests. Manual verification is the entire point of the demo:
1. Push a commit
2. See the workflow turn green
3. Refresh the page, see new timestamp/SHA

If the user wants to confirm the loop a second time, change a word in `index.html`, push, refresh.

## Open Questions

None. Design is fully specified.
