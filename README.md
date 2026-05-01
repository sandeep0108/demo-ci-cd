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
   git remote add origin https://github.com/sandeep0108/demo-ci-cd.git
   git push -u origin main
   ```
3. In your GitHub repo, go to **Settings → Pages**.
4. Under **Build and deployment**, set **Source** to **GitHub Actions**.

That's it. The next push to `main` (or a manual run from the Actions tab) will deploy.

## Verifying the pipeline works

1. Open the **Actions** tab on GitHub. You should see a "Deploy to GitHub Pages" run after each push.
2. Once the run is green, visit `https://sandeep0108.github.io/demo-ci-cd/`.
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
