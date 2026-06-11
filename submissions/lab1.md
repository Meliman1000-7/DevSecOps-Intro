# Lab 1 — Submission

## Triage Report: OWASP Juice Shop

### Scope \& Asset

* Asset: OWASP Juice Shop (local lab instance)
* Image: `bkimminich/juice-shop:v20.0.0`
* Image digest: `sha256:fd58bdc9745416afce8184ee0666278a436574633ea7880365153a63bfd418b0`
* Host OS: Windows 11
* Docker version: Docker version 29.4.2, build 055a478

### Deployment Details

* Run command used: `docker run -d --name juice-shop -p 127.0.0.1:3000:3000 bkimminich/juice-shop:v20.0.0`
* Access URL: http://127.0.0.1:3000
* Network exposure: 127.0.0.1 only? \[x] Yes
* Container restart policy: default (`no` — no `--restart` flag was specified)

### Health Check

* HTTP code on `/`: 200
* Product count from `/api/Products`: 46 items returned
* Container uptime (from `docker ps`):

```
  NAMES        STATUS          PORTS
  juice-shop   Up 12 minutes   127.0.0.1:3000->3000/tcp
  ```

* Version check (`/rest/admin/application-version`):

```json
  {"version":"20.0.0"}
  ```

### Initial Surface Snapshot (from browser exploration)

* Login/Registration visible: \[x] Yes — Account menu (top-right) leads to `/#/login`; registration available from the login page
* Product listing/search present: \[x] Yes — product grid loads on the homepage with images, prices, and add-to-cart buttons
* Admin or account area discoverable: \[x] Yes — Account menu is visible without authentication; admin panel path (`/#/administration`) is discoverable via JavaScript source
* Client-side errors in DevTools console: \[x] No — Firefox DevTools console showed no errors on page load
* Pre-populated local storage / cookies: Two cookies set automatically on first visit:

  * `language=en` — stores the selected UI language
  * `welcomebanner\_status=dismiss` — tracks dismissal of the welcome banner

### Network Observations (from Firefox DevTools → Network tab)

* `GET /rest/user/whoami?fields=email` — returns `304 Not Modified` (cached); called on every page load **without requiring authentication**, leaking the unauthenticated state check
* `GET /rest/products/1/reviews` — returns `200 OK`; product reviews are **publicly accessible without any authentication token**, returning full review data including user identifiers

### Security Headers (from `Invoke-WebRequest` response headers)

```
Access-Control-Allow-Origin : \*
X-Content-Type-Options      : nosniff
X-Frame-Options             : SAMEORIGIN
Feature-Policy              : payment 'self'
X-Recruiting                : /#/jobs
Content-Type                : text/html; charset=UTF-8
Cache-Control               : public, max-age=0
```

Headers that are **MISSING** (cross-referenced with OWASP Top 10:2025 — A05: Security Misconfiguration):

* \[x] `Content-Security-Policy` — **MISSING**
* \[x] `Strict-Transport-Security` — **MISSING** (app runs over plain HTTP)
* \[ ] `X-Content-Type-Options: nosniff` — present ✅
* \[ ] `X-Frame-Options` — present (`SAMEORIGIN`) ✅

Additional concern: `Access-Control-Allow-Origin: \*` is a wildcard CORS policy, meaning any origin can make cross-origin requests to this API.

### Top 3 Risks Observed

1. **Missing Content-Security-Policy** — The application does not set a `Content-Security-Policy` header, meaning the browser applies no restrictions on which scripts, styles, or frames can be loaded. An attacker who injects malicious JavaScript (e.g., via a stored XSS payload in a product review) can execute arbitrary code in any victim's browser with no CSP barrier. Maps to **A05:2025 – Security Misconfiguration**.
2. **Unauthenticated Access to Reviews and User Endpoints** — The `/rest/products/<id>/reviews` endpoint returns full review data including user references with no authentication required, and `/rest/user/whoami` is queried on every page load without a token. This exposes internal user data and API structure to any unauthenticated visitor or automated scanner. Maps to **A01:2025 – Broken Access Control**.
3. **Wildcard CORS Policy** — Every API response includes Access-Control-Allow-Origin: \*, allowing any website to read API responses from a victim's browser via cross-origin requests. Combined with unauthenticated endpoints, a malicious page can silently enumerate products, read reviews, and probe the API structure on behalf of any visitor — no user interaction required beyond visiting the attacker's site. Maps to A05:2025 – Security Misconfiguration.

\---

## PR Template Setup

* File: `.github/PULL\_REQUEST\_TEMPLATE.md`
* Sections included: Goal / Changes / Testing / Artifacts \& Screenshots
* Checklist items:

  * \[ ] Title is clear (`feat(labN): <topic>` style)
  * \[ ] No secrets or large temp files committed
  * \[ ] Submission file at `submissions/labN.md` exists
* Auto-fill verified: \[x] Yes — PR description showed the template automatically when opening the PR from `feature/lab1`

\---

## Bonus: CI Smoke Test

* Workflow file: `.github/workflows/lab1-smoke.yml`
* Trigger: `pull\_request` on `main`
* Run URL (must be green): *<link to your Actions run — fill in after pushing>*
* Workflow run duration: *<fill in after run completes, e.g. 52s>*
* Curl response excerpt:

```
  {"version":"20.0.0"}
  ```

