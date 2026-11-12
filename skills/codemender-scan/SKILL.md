---
name: codemender-scan
description: >-
  Simple, non-developer-friendly workflow to initialize Google CodeMender (cm),
  verify GCP project enablement for Fairwind Cyber models, select scan targets,
  models, custom context, and run automated vulnerability scans with plain-English reporting.
---

# CodeMender Easy Scan (`codemender-scan`)

A streamlined, beginner-friendly Antigravity skill and plugin for scanning codebases with **Google CodeMender (`cm`)**. Designed for security analysts, project managers, and non-developers who want straightforward vulnerability scanning without complex CI/CD hooks or developer gate scripts.

---

## 🎯 Guided Workflow Overview

When activated (or invoked via `/codemender-scan`), the agent guides you through 5 clear phases:

0. **Preflight CLI Verification** (`which cm`)
1. **GCP Project & Cyber Model Access Check**
2. **Interactive Scan Parameter Setup** (Mode, Target, Model, Custom Context, Report Path)
3. **Automated Scan Execution**
4. **Plain-English Findings & 1-Click Action Steps**

---

## Phase 0: Preflight CLI Check (`which cm`)

The agent checks if the CodeMender CLI (`cm`) is installed in your `$PATH`:
```bash
which cm
```

### If `cm` is NOT installed:
The agent stops and provides the quick download instructions:

**For Linux (x86_64) / Cloudtop:**
```bash
curl -L -o cm-linux-amd64.zip \
  "https://artifactregistry.googleapis.com/download/v1/projects/cmoc-prod/locations/us/repositories/codemender-cli-production/files/cm%3Astable%3Acm-linux-amd64.zip:download?alt=media"
unzip -o cm-linux-amd64.zip
chmod +x cm
sudo mv cm /usr/local/bin/cm
```

**For macOS (or via gcloud):**
```bash
gcloud artifacts generic download \
  --project=cmoc-prod \
  --location=us \
  --repository=codemender-cli-production \
  --package=cm \
  --version=stable \
  --name=cm-linux-amd64.zip \
  --destination=./
unzip -o cm-linux-amd64.zip
chmod +x cm
sudo mv cm /usr/local/bin/cm
```

---

## Phase 1: GCP Project & Cyber Model Access

The agent verifies your GCP environment and allowlisting:

### 1. Identify Project
The agent checks your default active project:
```bash
gcloud config get-value project
```
It prompts:
> *"Which GCP project allowlisted for CodeMender / Fairwind Cyber models should we use? (Default: `codemender-oss` or `<active-project>`)"*

### 2. Verify Health
The agent runs:
```bash
CLOUDSDK_CORE_PROJECT=<PROJECT_ID> cm health
```

- **If credentials expired:** The agent instructs you to run:
  ```bash
  gcloud auth application-default login
  ```
- **If HTTP 403 Forbidden:** The project is not yet allowlisted on the CodeMender backend. The agent prompts for an alternate allowlisted project.

---

## Phase 2: Interactive Scan Setup (5 Parameters)

The agent prompts for the following 5 settings:

### 1. Scan Mode
- **Quick Scan (`--quick`)**: Fast single-session scan (typically 3–8 minutes).
- **Deep Scan (Default)**: Exhaustive file-by-file concurrent audit across the entire codebase.

### 2. Target Directory or Repository
- **Current Workspace**: Scan the current folder (`.`)
- **Specific Subfolder**: e.g. `./src/`, `./app/`, or a cloned project
- **Single High-Risk File**: e.g. `./src/auth/login.py`

### 3. Model Version
Option | Model Name | Best Used For
:--- | :--- | :---
**1. Fast Scan (Default)** | `gemini-3.8-flash` | Rapid general preview scanning with standard Gemini intelligence.
**2. Fairwind Cyber Flash** | `gemini-3.8-flash-cyber` | Specialized security reasoning model for vulnerability discovery (requires Fairwind Tier 1/2 allowlist).
**3. Fairwind Cyber Argon** | `gemini-4-argon-cyber-permissive` | Deep cyber reasoning model for complex exploit path analysis (requires Fairwind Tier 1/2 allowlist).
**4. Argon Standard** | `gemini-4-argon` | High-capability standard model for exhaustive code audits.

### 4. Custom Scan Context (`-c, --context`)
The agent asks:
> *"Do you want to add context to this scan? (e.g. 'Focus on authentication and session hijacking', 'Prioritize SQL injection and input sanitization', or press Enter to run a general scan)"*

If provided, the agent passes `-c "<context>"` to guide CodeMender's reasoning.

### 5. Report Destination
Default: `./reports/codemender/`

---

## Phase 3: Execute & Export

The agent executes the scan with auto-approval (`-y`):

```bash
mkdir -p <REPORT_DIR>
CLOUDSDK_CORE_PROJECT=<PROJECT_ID> cm find <TARGET> \
  --model <MODEL_NAME> \
  [--quick] \
  [-c "<CONTEXT>"] \
  -y
```

Once complete, the agent automatically exports the report files:
```bash
cm report --format json > <REPORT_DIR>/findings.json
cm report --format table > <REPORT_DIR>/summary.txt
```

---

## Phase 4: Plain-English Findings & Next Steps

Instead of raw compiler traces, the agent presents:

1. **Executive Summary Table**: Total vulnerabilities categorized by severity (**Critical**, **High**, **Medium**, **Low**).
2. **Finding Breakdown**:
   - **What it is**: Plain-English explanation of the flaw.
   - **Where it is**: Exact file and line numbers.
   - **Why it matters**: The business and security risk.
3. **1-Click Next Steps**:
   - **Prove Exploitability**: `cm find verify <finding-id>` (Generates and runs an active PoC exploit in an isolated sandbox).
   - **Auto-Fix**: `cm fix <finding-id>` (Generates a verified, non-regressing patch).
   - **Open Visual Report**: `cm report --format html --open` (Opens full interactive HTML report in browser).
