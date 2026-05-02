# DAST scanner stack

Authenticated DAST sweep for `expediotechnologies.com`, `admin.expediotechnologies.com`, and `dashboard.expediotechnologies.com`. Deployed as a Coolify Docker Compose application from this repo.

## Stack

- **reports** — `nginx:alpine` with basic auth + autoindex, serves the `dast-reports` named volume over HTTP. Coolify assigns the FQDN automatically via `SERVICE_FQDN_REPORTS_80`. `/healthz` is unauthenticated for monitoring.
- **scanner** — `docker:27-cli-alpine3.20` with `/var/run/docker.sock` mounted. Orchestrates sibling-container runs of:
  - **Nikto** (`sullo/nikto`) — server-level recon
  - **Nuclei** (`projectdiscovery/nuclei:latest`) — known CVE / misconfig templates, severity-filtered
  - **ZAP** (`zaproxy/zap-stable`) — passive baseline by default, opt-in to full active scan

Reports are written into the shared `dast-reports` volume:

```
/reports/
  scan.log
  YYYY-MM-DD_HHMMSS/
    _meta.txt
    expediotechnologies.com/
      nikto.html
      nuclei.txt
      zap-baseline.html / zap-baseline.json
    admin.expediotechnologies.com/
      ...
    dashboard.expediotechnologies.com/
      ...
```

## Configuration (Coolify env tab)

| Var | Default | Notes |
| --- | --- | --- |
| `TARGETS` | three expediotechnologies domains | comma-separated URLs (must include scheme) |
| `SEVERITY` | `medium,high,critical` | Nuclei severity filter |
| `ZAP_MODE` | `baseline` | `baseline` (passive, ~2 min, prod-safe) or `full` (active, hours, owned sites only) |
| `BASIC_AUTH_USER` | `admin` | reports site basic auth |
| `BASIC_AUTH_PASS` | `changeme-please` | **set this in Coolify** |

## Operate

| Action | How |
| ------ | --- |
| Trigger scan | Redeploy the application in Coolify (or push to `main`). |
| View reports | Open the Coolify-assigned FQDN for the `reports` service. Login with `BASIC_AUTH_USER` / `BASIC_AUTH_PASS`. |
| Live progress | Tail `/reports/scan.log` from the scanner container, or refresh the index. |
| Switch to full ZAP | Set `ZAP_MODE=full` in env, redeploy. **Sites you own only.** |
| Add/remove targets | Edit `TARGETS`, redeploy. |

## Why a docker-socket pattern (not three sidecar services)

Nikto, Nuclei, and ZAP have very different lifecycles (Nikto in seconds, Nuclei in minutes, ZAP up to hours). Modeling each as a long-running Compose service forces awkward synchronization. The orchestrator script driving sibling `docker run` invocations keeps each scanner as a throwaway one-shot and makes the runner trivially shell-scriptable.

The trade-off: the scanner container has the host docker socket — effectively root on the host. Acceptable here because it's running on infrastructure you own and only ever invokes pinned upstream images.

## Not included (yet)

- **Authenticated scanning.** ZAP baseline / Nuclei only see what's reachable without logging in (marketing pages + login form). Real coverage of the post-login app requires either a session cookie passed to ZAP via `-z` or a recorded login script. Worth wiring up before go-live.
- **Notifications.** Findings sit on the reports site; nothing pages anyone. Easiest add: a hook at the end of the scanner script that greps Nuclei for `[critical]` / `[high]` and posts to Slack.
- **Result diffing.** Each run is its own folder; no "what's new since last run" view.

## Authorization

Run only against sites you own or have written authorization to test. ZAP **full scans** and Nuclei active templates send genuinely malicious payloads. Keep `ZAP_MODE=baseline` for any environment that isn't fully under your control.
