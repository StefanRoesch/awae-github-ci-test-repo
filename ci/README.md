# AWAE CI templates (GitLab and GitHub)

**URL-in, report-out:** Both templates expect you to provide reachable URLs and credentials. They do **not** build or serve your app; you provide `AWAE_URLS` (e.g. from a deploy job, staging URL, or review app).

The backend supports `?format=gitlab` (Pa11y-style, default) and `?format=sarif` (SARIF 2.1.0). The GitLab template uses the default; the GitHub template uses `format=sarif`.

- **GitLab:** `ci/awae.gitlab-ci.yml` — includeable template; publishes the report for the MR Accessibility widget.
- **GitHub:** `ci/awae.github-ci.yml` — workflow you copy to `.github/workflows/`; uploads SARIF for the Code Scanning tab.

Both files are self-contained — one file each, script inlined.

---

## GitLab: `ci/awae.gitlab-ci.yml`

**Requirements:** GitLab 17+ (for `spec:inputs`). Runner with Docker executor; image gets `curl` and `jq` in `before_script`.

### Include in your pipeline

```yaml
include:
  - local: 'ci/awae.gitlab-ci.yml'
```

### Required CI/CD variables

Set in **Settings → CI/CD → Variables**:

| Variable         | Description |
|------------------|-------------|
| `AWAE_HOST`      | Base URL of the AWAE API (no trailing slash). |
| `AWAE_API_TOKEN` | Bearer token for the API (set as masked). |
| `AWAE_USER_ID`   | UUID of the user that owns the report. |
| `AWAE_URLS`      | **Required.** Comma-separated reachable URLs to evaluate. |

### Optional variables / inputs

`AWAE_TITLE`, `AWAE_POLL_INTERVAL`, `AWAE_POLL_MAX_MINUTES`, `AWAE_DOWNLOAD_QUALWEB`. When including, you can pass inputs: `stage`, `poll_interval`, `poll_max_minutes`, `download_qualweb`.

### Artifacts

- `gl-accessibility.json` — GitLab Accessibility Report; set as `reports: accessibility` so the MR widget shows it.
- `qualweb-*.json.gz` — optional, if `download_qualweb` is true.

---

## GitHub: `ci/awae.github-ci.yml`

### Setup

1. Copy `ci/awae.github-ci.yml` → `.github/workflows/awae-accessibility.yml`
2. In the repo: **Settings → Secrets and variables → Actions**
   - Add **Secrets:** `AWAE_HOST`, `AWAE_API_TOKEN`, `AWAE_USER_ID`
   - Add **Variable:** `AWAE_URLS` (comma-separated URLs), or pass `AWAE_URLS` from a previous job

### Required

| Secret / Variable | Description |
|-------------------|-------------|
| `AWAE_HOST` (secret)   | Base URL of the AWAE API (no trailing slash). |
| `AWAE_API_TOKEN` (secret) | Bearer token for the API. |
| `AWAE_USER_ID` (secret)   | UUID of the user that owns the report. |
| `AWAE_URLS` (variable or from previous job) | Comma-separated reachable URLs to evaluate. |

### Output

Uploads `results.sarif` via `github/codeql-action/upload-sarif` so results appear in the **Code Scanning** tab on PRs.

### Passing AWAE_URLS from a previous job

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      awae_urls: ${{ steps.urls.outputs.awae_urls }}
    steps:
      - id: urls
        run: echo "awae_urls=https://your-app.example.com" >> $GITHUB_OUTPUT

  a11y-check:
    needs: deploy
    # ... same steps as in ci/awae.github-ci.yml, and set:
    env:
      AWAE_URLS: ${{ needs.deploy.outputs.awae_urls }}
```

---

## Job behavior (both platforms)

1. **Validate** `AWAE_HOST`, `AWAE_API_TOKEN`, `AWAE_USER_ID`, `AWAE_URLS`; exit with a clear error if missing.
2. **Create report:** `POST /api/reports` with retry (exponential backoff + jitter, up to 3 attempts).
3. **Poll:** `GET /api/reports/:id` until evaluation is `done` or `failed`; backoff and circuit breaker on errors; timeout after `AWAE_POLL_MAX_MINUTES`.
4. **Download** the accessibility report (GitLab format or SARIF depending on platform).
5. **Publish:** GitLab → MR widget; GitHub → Code Scanning (SARIF).

Secrets are not logged; only report ID, status, and violation count appear in logs.
