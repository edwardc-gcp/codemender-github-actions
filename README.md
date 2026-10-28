# CodeMender GitHub Actions Starter

[![Google Cloud](https://img.shields.io/badge/Google_Cloud-Gemini_Enterprise_Agent_Platform-4285F4?logo=google-cloud&logoColor=white)](https://cloud.google.com)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD_Automation-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

Production-ready, turn-key **GitHub Actions workflows** and configuration templates for integrating [Google Cloud CodeMender](https://docs.cloud.google.com/gemini-enterprise-agent-platform/codemender) (`cm`) into your continuous integration and deployment pipelines.

CodeMender transforms your CI/CD pipeline from a passive static analyzer into an **autonomous AI Security Co-developer** powered by Gemini reasoning models (`gemini-3.8-flash`, `gemini-3.8-flash-cyber`, and `gemini-4-argon`).

---

## 🚀 Features

* 🔑 **Zero-Trust Keyless Auth & Preflight Diagnostics**: Authenticate GitHub Actions runners to Google Cloud IAM via **Workload Identity Federation (WIF)**—zero static service account JSON keys—and validate credentials, configuration, network reachability, and Vertex AI IAM authorization upfront with `cm health --strict`.
* ⚡ **Adaptive 3-Tier Scan Modes**:
  * **Differential AST PR Scans (`--diff`)**: Restrict Pull Request scanning to modified files and their 1-hop callers/callees (`--diff="origin/$BASE_REF" --diff-workers=16`) for fast PR feedback loops.
  * **Rapid Baseline Audits (`--quick`)**: Run single-session repository scans on `main`/`master` branch pushes in minutes.
  * **Concurrent File-by-File Deep Scans (Default `cm find`)**: Run exhaustive multi-worker deep audits (`--deep-workers=8`) on weekly schedules or on-demand `workflow_dispatch`.
* 🛡️ **Legacy Debt Noise Suppression**: Findings in untouched code during PR scans are labeled `[LEGACY / UNTOUCHED]` in SARIF reports for visibility, without failing the PR merge gate or triggering unrelated autofixes.
* 🚦 **Quality Hard Gates (`--fail-on`)**: Automatically block PR merges with non-zero exit codes when newly introduced Critical or High vulnerabilities are detected, while safely uploading SARIF alerts via `!cancelled() && hashFiles('results.sarif') != ''` guards.
* 🛡️ **OASIS SARIF v2.1.0 & Native Step Summaries**: Export standardized SARIF for GitHub Code Scanning and publish Markdown executive summary tables directly to `$GITHUB_STEP_SUMMARY`.
* 🤖 **Autonomous Remediation & Auto-PR**: Automatically synthesize language-aware security patches (parameterization, input sanitization), verify against regression test suites inside isolated runners, and open review Pull Requests (`create-pull-request@v8`).
* 📦 **Artifact-Driven State Transfer (`cm report import`)**: Seamlessly hand off actionable findings between scanning and remediation jobs via JSON artifacts, eliminating fragile cross-runner SQLite database restoration.
* 🌐 **Polyglot AST Support**: Works natively with JavaScript/TypeScript, Python, Go, Java, C/C++, C#, Rust, Kotlin, Ruby, and PHP without external plugins.

---

## 📦 What's Included

```text
.
├── .github/
│   └── workflows/
│       ├── codemender-scan.yml                   # Continuous scanning, SARIF export & Step Summary
│       ├── codemender-remediate.yml              # Autonomous patch generation & Auto-PR (Public Repos)
│       └── codemender-remediate-private-repo.yml # Full PoC verification & Auto-PR (Private Repos)
├── .codemender/
│   └── config.yaml                               # Headless CI profile & automated test gate
├── .gitignore
├── LICENSE
└── README.md
```

---

## ⚡ Quick Start: Add to Existing Repository

Run the following commands from the root of your existing Git repository to fetch these pre-configured templates:

```bash
# 1. Create target directories
mkdir -p .github/workflows .codemender

# 2. Download Security Scanning & SARIF Export Workflow
curl -fsSL https://raw.githubusercontent.com/edwardc-gcp/codemender-github-actions/main/.github/workflows/codemender-scan.yml \
  -o .github/workflows/codemender-scan.yml

# 3. Download Autonomous Remediation & Auto-PR Workflow
curl -fsSL https://raw.githubusercontent.com/edwardc-gcp/codemender-github-actions/main/.github/workflows/codemender-remediate.yml \
  -o .github/workflows/codemender-remediate.yml

# 4. Download CI Headless Configuration Profile
curl -fsSL https://raw.githubusercontent.com/edwardc-gcp/codemender-github-actions/main/.codemender/config.yaml \
  -o .codemender/config.yaml
```

---

## 🔑 Required GitHub Repository Secrets

Configure the following secrets in your repository settings (**Settings > Secrets and variables > Actions**):

| Secret Name | Example / Description |
| :--- | :--- |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | `projects/1234567890/locations/global/workloadIdentityPools/my-pool/providers/my-provider` |
| `GCP_SERVICE_ACCOUNT` | `codemender-ci-runner@my-project-id.iam.gserviceaccount.com` |

---

## 🛠️ Google Cloud Workload Identity Federation Setup

Run these commands in [Google Cloud Shell](https://shell.cloud.google.com/) to provision keyless authentication:

```bash
# Set your environment variables
export PROJECT_ID=$(gcloud config get-value project)
export GITHUB_ORG="YOUR_GITHUB_ORG_OR_USERNAME"
export GITHUB_REPO="YOUR_REPO_NAME"
export POOL_NAME="codemender-github-pool"
export PROVIDER_NAME="codemender-github-provider"
export SA_NAME="codemender-ci-runner"
export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# 1. Enable required APIs
gcloud services enable iam.googleapis.com iamcredentials.googleapis.com sts.googleapis.com aiplatform.googleapis.com cloudresourcemanager.googleapis.com --project="${PROJECT_ID}"

# 2. Create Service Account and grant Gemini Agent Platform permissions
gcloud iam service-accounts create ${SA_NAME} --display-name="CodeMender CI Runner" --project="${PROJECT_ID}"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/aiplatform.user"

# 3. Create Workload Identity Pool and OIDC Provider
gcloud iam workload-identity-pools create ${POOL_NAME} --location="global" --project="${PROJECT_ID}"
gcloud iam workload-identity-pools providers create-oidc ${PROVIDER_NAME} \
  --workload-identity-pool=${POOL_NAME} \
  --location="global" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository == '${GITHUB_ORG}/${GITHUB_REPO}'" \
  --project="${PROJECT_ID}"

# 4. Bind Service Account Impersonation
export POOL_ID=$(gcloud iam workload-identity-pools describe ${POOL_NAME} \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)")
gcloud iam service-accounts add-iam-policy-binding "${SA_EMAIL}" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${POOL_ID}/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}"

# 5. Output Provider Name for GitHub Secrets
gcloud iam workload-identity-pools providers describe ${PROVIDER_NAME} \
  --workload-identity-pool=${POOL_NAME} \
  --location="global" \
  --project="${PROJECT_ID}" \
  --format="value(name)"
```

---

## ⚙️ Workflow Breakdown

### 1. `codemender-scan.yml` (Scan & SARIF Export)
* **Trigger**: Pull requests and pushes targeting `main` / `master`, scheduled weekly audits, and manual dispatch.
* **Scan Modes**:
  * **Differential PR Scan (`--diff`)**: On `pull_request` events, analyzes modified files and their 1-hop AST callers/callees (`--diff="origin/$BASE_REF" --diff-workers=16 --fail-on="CRITICAL,HIGH"`), blocking merges on newly introduced Critical/High flaws.
  * **Quick Baseline Scan (`--quick`)**: On `push` events to `main`/`master` (and default manual runs), runs a rapid single-session repository scan in 3–5 minutes.
  * **Deep Scan Mode (Default `cm find .`)**: On weekly scheduled cron audits or when toggled via `workflow_dispatch (deep_scan: true)`, runs an exhaustive concurrent file-by-file deep scan (`--deep-workers=8`) across all discovered source files.
* **Key Steps**:
  1. Checks out code with `persist-credentials: false`, authenticates via WIF (`google-github-actions/auth@v2`), configures `.git/info/exclude`, and runs `cm health --strict` preflight diagnostics.
  2. Executes the event-appropriate autonomous scan (`--diff`, `--quick`, or `--deep-workers=8`).
  3. Conditionally exports native OASIS SARIF v2.1.0 (`cm report -f sarif > results.sarif`) and `codemender-findings.json` artifact (filtering out `[LEGACY / UNTOUCHED]`) whenever `!cancelled()`.
  4. Publishes formatted Markdown summary tables to `$GITHUB_STEP_SUMMARY`.
  5. Ingests findings into GitHub Code Scanning via `github/codeql-action/upload-sarif@v4` when `results.sarif` is non-empty (`!cancelled() && hashFiles('results.sarif') != ''`).

### 2. `codemender-remediate.yml` (Autonomous Patch Remediation for Public Repositories)
* **Target Audience**: Public & Open-Source Repositories (Responsible Disclosure Mode).
* **Trigger**: Nightly scheduled runs (`0 3 * * *`) or manual dispatch (`workflow_dispatch`) with optional finding ID target.
* **Key Features**:
  1. Clones repository with full Git history (`fetch-depth: 0`, `persist-credentials: false`), pins `project_paths` to `$GITHUB_WORKSPACE`, and validates readiness via `cm health --strict`.
  2. Resumes instantly without re-scanning by importing `codemender-findings.json` (`cm report import -f ...`), falling back to `cm find . --quick` only when no cached findings exist.
  3. Synthesizes security patches via `cm fix <id> --sandbox=false --yes --bypass-warning < /dev/null` (preserving cumulative edits with `--no-reset` in batch mode).
  4. Purges internal `.cm_project`, `.exploit/`, and temporary metadata so commits strictly touch production code.
  5. Generates dynamic, context-aware PR titles (`🛡️ [CodeMender] Fix: <Title>` or `🛡️ [CodeMender] Fix Finding <ID>`).
  6. Submits an automated, review-ready Pull Request (`peter-evans/create-pull-request@v8`) linking to GitHub Code Scanning SARIF alerts without exposing raw attack vectors publicly.

### 3. `codemender-remediate-private-repo.yml` (Autonomous Patch Remediation for Private Repositories)
* **Target Audience**: Private Repositories & Enterprise Organizations (Full Audit Evidence Mode).
* **Trigger**: Nightly scheduled runs (`0 3 * * *`) or manual dispatch (`workflow_dispatch`) with optional finding ID target.
* **Key Features**:
  1. Everything in the public remediation workflow, plus dynamic PoC exploit verification (`cm verify <id> --no-reset`) prior to patch generation.
  2. **Detailed Context PR Titles**: Displays precise vulnerability headline and severity (e.g. `🛡️ [CodeMender] Fix: SQL Injection in User Authentication Route (CRITICAL)`).
  3. **Structured Evidence Breakdown Table**: Renders complete Finding ID, Severity, CWE category, Headline, and exact file/line coordinates directly in the PR description.
  4. **Embedded Verification Evidence (Zero Repo Pollution)**: Embeds CodeMender's `.exploit/` audit artifacts (`REPORT.md`, `PLAN.md`, `exploit.sh`) inside collapsible sections of the PR description and archives them as a workflow artifact (`codemender-verification-evidence`), while excluding exploit scripts from Git commits.

---

## 🔒 Security Best Practices

1. **Least-Privilege GitHub Permissions & Credential Hygiene**:
   * Scanning job strictly uses `contents: read`, `id-token: write`, `security-events: write`, and `actions: read` (required by CodeQL SARIF uploader on private repositories).
   * Remediation job uses `contents: write`, `id-token: write`, and `pull-requests: write`.
   * All checkouts set `persist-credentials: false` so GitHub tokens are never persisted in `.git/config` while autonomous agents inspect the workspace.
2. **Untrusted Fork & Bot PR Guardrails**:
   * Restrict WIF token issuance with attribute condition: `assertion.repository == '${GITHUB_ORG}/${GITHUB_REPO}'`.
   * The scan workflow includes a job-level `if:` condition that skips untrusted fork Pull Requests and `dependabot[bot]` PRs where repository WIF secrets are unavailable.
3. **Concurrency & Timeout Controls**:
   * Workflows enforce explicit `timeout-minutes` bounds and `concurrency` groups to prevent runaway runner billing and branch merge collisions.

---

## 📄 License

Distributed under the [Apache License, Version 2.0](LICENSE).
