# awae-github-ci-test-repo

Test repository for the customer-facing **`ci/awae.github-ci.yml`** — the GitHub equivalent of `ci/awae.gitlab-ci.yml`. One self-contained file; the customer copies it to `.github/workflows/awae-accessibility.yml`, sets secrets and `AWAE_URLS`, done. The backend supports `?format=gitlab` (default) and `?format=sarif`; this template uses `format=sarif` for Code Scanning.

## Layout

| Path | Purpose |
|------|--------|
| `ci/awae.github-ci.yml` | **Customer-facing.** GitHub Actions workflow (URL-in, report-out; SARIF → Code Scanning). |
| `ci/awae.gitlab-ci.yml` | GitLab CI includeable template (URL-in, report-out; MR Accessibility widget). Kept for reference. |
| `ci/README.md` | Docs for both templates. |
| `.github/workflows/test-awae-accessibility.yml` | **Test-only.** Serves `public/` + tunnel → runs the same a11y script → uploads SARIF. Not for customers. |
| `public/` | ACT test pages; used only by the test workflow. |

## Required GitHub secrets (for this test repo)

Configure in **Settings → Secrets and variables → Actions → Secrets**:

| Secret | Description |
|--------|-------------|
| `AWAE_HOST` | Base URL of the AWAE API (e.g. ngrok URL to your local awae), no trailing slash. |
| `AWAE_API_TOKEN` | Bearer token for the API. |
| `AWAE_USER_ID` | UUID of the user that owns the report. |

The test workflow builds `AWAE_URLS` from the tunnel; you do not set it manually.

## Running a test

1. Run **awae** locally and expose it (e.g. ngrok on port 5000). Set that URL as **AWAE_HOST** in this repo's secrets.
2. Push or open a PR. The test workflow serves `public/`, gets a tunnel URL, sends those URLs to AWAE, polls until done, and uploads SARIF to the Code Scanning tab.

## Offering the template to customers

Give them **`ci/awae.github-ci.yml`** — one file, copy to `.github/workflows/awae-accessibility.yml`. Point them to `ci/README.md` for required secrets/variables. They set `AWAE_URLS`; the template does not build or serve their app.
