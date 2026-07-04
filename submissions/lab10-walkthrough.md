# 5-Minute DevSecOps Program Walkthrough — Juice Shop

## (0:00–0:30) Context

I built a full DevSecOps program around OWASP Juice Shop v20.0.0 as a deliberately vulnerable Node.js target, implementing nine sequential security controls from pre-commit secrets detection through runtime threat detection and unified vulnerability management. The program covers commit signing (SSH), SBOM generation, SCA, SAST, DAST, IaC scanning, container image signing with Cosign, Falco runtime detection, and DefectDojo aggregation — all on a Mac ARM laptop using Docker and open-source tooling only.

## (0:30–2:00) Layers

The program follows a shift-left architecture with five layers:

**Pre-commit** (Lab 3): Every commit is SSH-signed (ED25519, verified badge on GitHub). gitleaks v8.21.2 runs as a pre-commit hook and blocks secrets from entering the repo — we validated this by staging a fake GitHub PAT and watching gitleaks block the commit with entropy score 4.14. The `git filter-repo` bonus demonstrated how to rewrite history when a secret does slip through, with the mandatory second step of secret rotation.

**Build** (Lab 4): Syft generates a CycloneDX 1.6 SBOM with 3,068 components from the Juice Shop image. Grype scans the SBOM and finds 103 vulnerabilities (7 Critical), Trivy adds 112 more with OS-layer coverage. An in-toto v1 attestation ties the SBOM to the image digest `sha256:fd58bdc...`.

**SAST + DAST** (Lab 5): Semgrep runs 151 rules and finds 22 findings — the highest-signal ones are `express-sequelize-injection` (6 hits) and `run-shell-injection` (5 hits). ZAP baseline scan finds 10 alerts; authenticated scan adds SQL Injection at `/rest/products/search`. The correlated finding — SQL injection caught by both Semgrep at `routes/search.ts:23` and ZAP at the live endpoint — is the strongest signal in the program.

**Pre-deploy IaC** (Lab 6): Checkov finds 78 failures on Terraform (top rule: IAM wildcard actions, 4 hits each for CKV_AWS_289/355). KICS finds 9 HIGH hardcoded secrets in Ansible playbooks. A custom Checkov policy `CKV2_CUSTOM_1` enforces `iam_database_authentication_enabled=true` on RDS instances and fires on both `unencrypted_db` and `weak_db`. Lab 7 adds Cosign signing of the image and Conftest gates on K8s manifests. Lab 9 extends Conftest with 4 Rego v1 policies: runAsNonRoot, allowPrivilegeEscalation=false, capabilities.drop ALL, and memory limits.

**Runtime** (Lab 9): Falco 0.43.1 runs with modern eBPF on Docker Desktop (linuxkit 6.12.76). The event-generator triggered 16 built-in rules including "Run shell untrusted" and "Read sensitive file untrusted." Two custom rules were written: "Write to /tmp by container" (WARNING) and "Possible Cryptominer Activity" (CRITICAL) — the latter fired when we renamed `/bin/sh` to `xmrig` and executed it, triggering both our rule and the built-in "Drop and execute new binary in container" simultaneously.

**Program** (Lab 10): DefectDojo aggregates all 8 scan types into one engagement — 390 raw findings. SLA matrix: Critical 24h, High 7d, Medium 30d, Low 90d.

## (2:00–3:00) Findings + Closures

We did not close findings in this engagement — Juice Shop is intentionally vulnerable and runs in an isolated lab network, so remediation is out of scope. However, two findings were risk-accepted with explicit 90-day expiry dates:

"CVE-2026-5450 in libc6" — Critical, no upstream fix available as of the scan date. Risk-accepted until 2026-10-04 with a calendar reminder to re-evaluate when a patch ships.

"GHSA-5mrr-rgp6-x4gr in marsdb" — Critical, NoSQL injection in an unmaintained package used only by Juice Shop's challenge engine. Replacing marsdb would break the lab; isolated environment justifies acceptance until 2026-10-04.

The strongest correlated finding in the program: SQL Injection at `/rest/products/search` — caught by Semgrep as a source-code pattern (`express-sequelize-injection` at `routes/search.ts:23`) AND confirmed live by ZAP's active scan returning a 200 with injected payload `'(`. This is the finding I would show first in a PR review because it has both static evidence (the vulnerable code) and dynamic proof (working exploit), making it impossible to dismiss as a false positive.

## (3:00–4:00) Metrics

- **MTTD:** ~17 days (Juice Shop v20.0.0 release to first scan in Lab 4)
- **MTTR:** undefined — no findings remediated (intentional for lab scope)
- **Vuln-age median:** 17 days for all open findings
- **Backlog:** 390 findings, trending flat (single engagement, no remediation cycle yet)
- **SLA compliance:** 0% for Critical/High — all 177 Critical+High findings exceed their SLA window

For context: DORA Elite performers achieve MTTR < 1 day for production incidents (DORA 2024 report). Our program is at SAMM SM1 — findings are tracked but not yet measured or assigned. The gap to Elite is significant and expected for a first-engagement baseline.

## (4:00–4:30) Next Steps

If I had another quarter, I would ship the DefectDojo↔Jira integration to auto-create tickets for every Critical/High finding on import, add MTTR measurement to the weekly pipeline, and ingest Falco alerts as a live runtime finding source via a custom parser — moving from SAMM SM1 (ad-hoc defect tracking) to SM2 (tracked, measured, and feedback-looped into the development team's sprint cadence).

## (4:30–5:00) Q&A Anticipation

**Q1: "How would you handle a Log4Shell scenario?"**

Log4Shell (CVE-2021-44228) is exactly the scenario our SBOM pipeline is designed for. With a CycloneDX SBOM covering 3,068 components, I would run `grype sbom:juice-shop.cdx.json --add-cpes-if-none` against an updated vulnerability database within minutes of the CVE being published. The SBOM already knows every log4j version in the dependency tree — the query is instantaneous rather than requiring a manual audit of source code or Docker layers. If log4j were present, I would trigger the Checkov/KICS pipeline to verify no IaC is pulling an affected image tag, then use Cosign's `verify --certificate-oidc-issuer` to confirm the image digest in production matches a signed build from before the vulnerability window. The full detection-to-triage cycle would be under 30 minutes.

**Q2: "Why didn't you use IAST or paid tools like Snyk or Veracode?"**

Honest tradeoff: IAST (like Contrast Security) gives lower false-positive rates on DAST findings because it instruments the running application and sees actual data flows rather than inferring them from HTTP responses. For a production program I would evaluate it seriously. For this course, open-source tooling (Semgrep, ZAP, Grype, Trivy, Checkov, Falco) was the deliberate choice because it demonstrates the same architectural patterns — shift-left scanning, SBOM-driven SCA, runtime detection — without licensing cost, and every tool integrates with DefectDojo's parsers out of the box. The skills transfer directly: if a future employer uses Snyk instead of Grype, the SBOM-first mental model and the DefectDojo aggregation layer are identical.
