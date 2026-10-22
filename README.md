# Shared CI for aiesec-testimonialPro

## `security-scan.yml` — vulnerability assessment

One reusable workflow, called by every repo in the org. Blocks on fixable
CRITICAL/HIGH dependency CVEs and committed secrets; reports IaC/Dockerfile
misconfiguration, gosec/Semgrep findings and MEDIUM/LOW CVEs without blocking.

IaC rules are advisory on purpose: Trivy's `misconfig` checks are best-practice
opinions rather than vulnerabilities. "Specify a USER command" is meaningless
for a Lambda container image and DS-0029 is about image size — neither is a
reason to stop a deploy. Pass `blocking-scanners: vuln,secret,misconfig` in a
repo that wants them enforced.

### Gating a deploy

Add a `security` job to the deploy workflow and make `deploy` depend on it.
A failing scan then means the deploy never starts — which is what you want,
because a deploy cannot be reliably cancelled once it has called
`aws lambda update-function-code`.

```yaml
jobs:
  security:
    uses: gid-support/.github/.github/workflows/security-scan.yml@main
    with:
      language: go      # or: node

  deploy:
    needs: security     # <- the gate
    runs-on: ubuntu-latest
    ...
```

### Gating pull requests

Repos also carry a `security.yml` that runs on PRs to `main`/`dev`/`develop`,
weekly, and on demand. Mark **Vulnerability assessment** as a required status
check in branch protection on `main` so nothing unscanned can land.

The weekly run matters more than it looks: CVEs get published against code that
has not changed in months, and only a scheduled scan catches those.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `language` | *required* | `go` adds govulncheck + gosec; `node` adds Semgrep |
| `severity` | `CRITICAL,HIGH` | Severities that fail the run |
| `blocking-scanners` | `vuln,secret` | Add `misconfig` to enforce IaC rules too |
| `go-version-file` | `go.mod` | |
| `scan-ref` | `.` | Subdirectory to scan |

### Suppressing a finding

Commit a `.trivyignore` at the repo root, one CVE id per line, each with a
comment saying why and when it will be revisited. Trivy picks it up
automatically. Do not widen `severity` to silence a single CVE.
