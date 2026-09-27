# CodeMender GitHub Actions Starter

[![Google Cloud](https://img.shields.io/badge/Google_Cloud-Gemini_Enterprise_Agent_Platform-4285F4?logo=google-cloud&logoColor=white)](https://cloud.google.com)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD_Automation-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

Production-ready, turn-key **GitHub Actions workflows** and configuration templates for integrating [Google Cloud CodeMender](https://docs.cloud.google.com/gemini-enterprise-agent-platform/codemender) (`cm`) into your continuous integration and deployment pipelines.

CodeMender transforms your CI/CD pipeline from a passive static analyzer into an **autonomous AI Security Co-developer** powered by Gemini reasoning models (`gemini-3.8-flash`, `gemini-3.8-flash-cyber`, and `gemini-3.1-pro-preview`).

---

## 🚀 Features

* 🔑 **Zero-Trust Keyless Authentication**: Authenticate GitHub Actions runners to Google Cloud IAM via **Workload Identity Federation (WIF)**—zero static service account JSON keys.
* ⚡ **Differential AST PR Scans (`--diff`)**: In Pull Requests, automatically restrict scanning to modified files and their 1-hop dependent callers, delivering sub-15-second feedback loops.
* 🚦 **Quality Hard Gates (`--fail-on`)**: Automatically block PR merges with non-zero exit codes when Critical or High vulnerabilities are detected, while gracefully uploading SARIF alerts.
* 🛡️ **OASIS SARIF v2.1.0 & Native Step Summaries**: Export standardized SARIF for GitHub Code Scanning and publish Markdown executive summary tables directly to `$GITHUB_STEP_SUMMARY`.
* 🤖 **Autonomous Remediation & Auto-PR**: Automatically synthesize language-aware security patches (parameterization, input sanitization), verify against regression test suites inside runner sandboxes, and open review Pull Requests.
* ⚡ **Dual Caching Architecture**: Cache CLI binaries and the SQLite findings database (`~/.codemender/state.db`) to slash incremental scan times from ~45s to **< 10s** and eliminate redundant Gemini API token usage.
* 🔒 **2-Tier Exploit Verification**: Fast Tier 1 semantic reachability checks (< 25s) for PR gatekeeping paired with Tier 2 sandboxed PoC exploit execution for scheduled sweeps.
* 🌐 **Polyglot AST Support**: Works natively with JavaScript/TypeScript, Python, Go, Java, C/C++, C#, Rust, Kotlin, Ruby, and PHP without external plugins.

---

## 📦 What's Included

```text
.
├── .github/
│   └── workflows/
│       ├── codemender-scan.yml        # Continuous scanning, SARIF export & Step Summary
│       └── codemender-remediate.yml   # Autonomous patch generation & Auto-PR creation
├── .codemender/
│   └── config.yaml                    # Headless CI profile & automated test gate
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
gcloud services enable iamcredentials.googleapis.com sts.googleapis.com aiplatform.googleapis.com

# 2. Create Service Account and grant Gemini Agent Platform permissions
gcloud iam service-accounts create ${SA_NAME} --display-name="CodeMender CI Runner"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/aiplatform.user"

# 3. Create Workload Identity Pool and OIDC Provider
gcloud iam workload-identity-pools create ${POOL_NAME} --location="global"
gcloud iam workload-identity-pools providers create-oidc ${PROVIDER_NAME} \
  --workload-identity-pool=${POOL_NAME} \
  --location="global" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository == '${GITHUB_ORG}/${GITHUB_REPO}'"

# 4. Bind Service Account Impersonation
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
gcloud iam service-accounts add-iam-policy-binding ${SA_EMAIL} \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_NAME}/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}"

# 5. Output Provider Name for GitHub Secrets
gcloud iam workload-identity-pools providers describe ${PROVIDER_NAME} \
  --workload-identity-pool=${POOL_NAME} \
  --location="global" \
  --format="value(name)"
```

---

## ⚙️ Workflow Breakdown

### 1. `codemender-scan.yml` (Scan & SARIF Export)
* **Trigger**: Pull requests and pushes targeting `main` / `master`, scheduled weekly audits, and manual dispatch.
* **Scan Modes**:
  * **Standard Fast Mode (Default on PR / Push)**: Scans core application logic (controllers, routes, models) with 1-hop AST impact analysis in 3–5 minutes.
  * **Deep Scan Mode (`--deep`, CodeMender 0.10.0+)**: Broadens AST analysis across all supported auxiliary code files, database migrations (`migrations/*.sql`), devops scripts, and tooling. Automatically enabled during weekly scheduled audits (`cron`) or manually toggled via `workflow_dispatch (deep_scan: true)`.
* **Key Steps**:
  1. Authenticates via WIF (`google-github-actions/auth@v2`).
  2. Restores CLI binary and findings cache (`state.db`).
  3. Executes autonomous scan (`cm find . -y --unrestricted --model gemini-3.8-flash` with optional `--deep`).
  4. Generates and normalizes OASIS SARIF v2.1.0 (`cm report -f sarif`).
  5. Publishes formatted Markdown table to `$GITHUB_STEP_SUMMARY`.
  6. Ingests findings into GitHub Code Scanning via `github/codeql-action/upload-sarif@v4`.

### 2. `codemender-remediate.yml` (Autonomous Patch Remediation for Public Repositories)
* **Target Audience**: Public & Open-Source Repositories (Responsible Disclosure Mode).
* **Trigger**: Nightly scheduled runs (`0 3 * * *`) or manual dispatch (`workflow_dispatch`) with optional finding ID target.
* **Key Features**:
  1. Clones repository with full Git history (`fetch-depth: 0`).
  2. Queries open vulnerabilities from local findings database (or targets specific finding ID).
  3. Synthesizes security patches via `cm fix <id> -y --bypass-warning --unrestricted --model gemini-3.8-flash`.
  4. Purges internal `.cm_project` and temporary metadata to ensure zero repository pollution.
  5. Generates dynamic, context-aware PR title (`🛡️ [CodeMender] Fix: <Title>` or `🛡️ [CodeMender] Fix Finding <ID>`).
  6. Submits automated, review-ready Pull Request (`peter-evans/create-pull-request@v6`) linking to GitHub Code Scanning SARIF alerts without exposing raw attack vectors publicly.

### 3. `codemender-remediate-private-repo.yml` (Autonomous Patch Remediation for Private Repositories)
* **Target Audience**: Private Repositories & Enterprise Organizations (Full Audit Evidence Mode).
* **Trigger**: Nightly scheduled runs (`0 3 * * *`) or manual dispatch (`workflow_dispatch`) with optional finding ID target.
* **Key Features**:
  1. Everything in the public remediation workflow, plus:
  2. **Detailed Context PR Titles**: Displays precise vulnerability headline and severity (e.g. `🛡️ [CodeMender] Fix: SQL Injection in User Authentication Route (CRITICAL)`).
  3. **Structured Evidence Breakdown Table**: Renders complete Finding ID, Severity, CWE category, Headline, and exact file/line coordinates directly in the PR description.
  4. **Direct PR Inclusion of Verification Artifacts**: Commits CodeMender's core audit artifacts (`PLAN.md`, `LOG.md`, `exploit.sh`, `REPORT.md`) directly into the Pull Request, providing complete visibility into the AI's attack planning, sandbox verification traces, and exploit PoC for internal security review.

---

## 🔒 Security Best Practices

1. **Least-Privilege GitHub Permissions**:
   * Scanning job strictly uses `contents: read`, `id-token: write`, and `security-events: write`.
   * Remediation job uses `contents: write`, `id-token: write`, and `pull-requests: write`.
2. **Untrusted Fork PR Guardrail**:
   * Do not pass `--unrestricted` on untrusted external fork Pull Requests.
   * Prefer `pull_request` over `pull_request_target` to prevent unauthorized Service Account impersonation.
3. **Concurrency Controls**:
   * Both workflows include `concurrency: group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: true` to prevent redundant runner billing and branch merge collisions.

---

## 📄 License

Distributed under the [Apache License, Version 2.0](LICENSE).
