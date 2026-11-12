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
which cm          # Linux / macOS
where.exe cm      # Windows (PowerShell)
```

If `cm` is found, skip to Phase 1. If it's outdated, the user can run `cm update` (prefix with `sudo` if it lives in a system directory).

### If `cm` is NOT installed

> **Access note:** The CodeMender CLI is in Public Preview for a limited set of customers. If downloads fail with 403/404, the user needs to request access through their Google Cloud sales team.

The agent **does not install anything itself**. It detects the platform, then shows the user the exact commands to run.

#### 1. Detect the platform
```bash
uname -sm                          # Linux / macOS, e.g. "Linux x86_64", "Darwin arm64"
```
```powershell
$env:PROCESSOR_ARCHITECTURE        # Windows, e.g. "AMD64", "ARM64"
```

#### 2. Pick the matching file

| Platform | `uname -sm` / Windows arch | `<FILE>` |
| :--- | :--- | :--- |
| Linux x86_64 | `Linux x86_64` | `cm-linux-amd64.zip` |
| Linux ARM64 | `Linux aarch64` | `cm-linux-arm64.zip` |
| macOS Intel | `Darwin x86_64` | `cm-darwin-amd64.zip` |
| macOS Apple silicon | `Darwin arm64` | `cm-darwin-arm64.zip` |
| Windows x86_64 | `AMD64` | `cm-windows-amd64.zip` |
| Windows ARM64 | `ARM64` | `cm-windows-arm64.zip` |

#### 3. Download (replace `<FILE>`)

**Option A: gcloud (all platforms, preferred)**
```bash
gcloud artifacts generic download \
  --project=cmoc-prod \
  --location=us \
  --repository=codemender-cli-production \
  --package=cm \
  --version=stable \
  --name=<FILE> \
  --destination=./
```
(On Windows PowerShell, use a backtick `` ` `` instead of `\` for line continuation.)

**Option B: curl (Linux / macOS)**
```bash
curl -L -o <FILE> \
  "https://artifactregistry.googleapis.com/download/v1/projects/cmoc-prod/locations/us/repositories/codemender-cli-production/files/cm%3Astable%3A<FILE>:download?alt=media"
```

**Option C: PowerShell (Windows)**
```powershell
Invoke-WebRequest `
  -Uri "https://artifactregistry.googleapis.com/download/v1/projects/cmoc-prod/locations/us/repositories/codemender-cli-production/files/cm%3Astable%3A<FILE>:download?alt=media" `
  -OutFile <FILE>
```

#### 4. Install

**Linux / macOS**
```bash
unzip -o <FILE>
chmod +x cm
sudo mv cm /usr/local/bin/cm
cm --version
```

**Windows (PowerShell)**
```powershell
Expand-Archive -Path <FILE> -DestinationPath ./
# Move cm.exe to a permanent folder (e.g. C:\Tools\cm) and add that folder to your PATH.
cm --version
```

If anything here fails, the source of truth is the official guide:
https://docs.cloud.google.com/gemini-enterprise-agent-platform/agents/codemender/set-up-environment#download-cli

---

## Phase 1: GCP Project & Cyber Model Access

The agent verifies your GCP environment and allowlisting:

### 1. Identify Project
The agent checks your default active project:
```bash
gcloud config get-value project
```
It prompts:
> *"Which GCP project allowlisted for CodeMender / Fairwind Cyber models should we use? (Default: `<active-project>`)"*

### 2. Check Required APIs
The project needs two APIs turned on. The agent checks them (read-only):
```bash
gcloud services list --enabled --project=<PROJECT_ID> \
  --filter="config.name:(aiplatform.googleapis.com cloudresourcemanager.googleapis.com)" \
  --format="value(config.name)"
```

- **Both names printed:** continue.
- **One or both missing:** the agent shows the command below and runs it **only if the user confirms**. Turning on APIs changes the project and needs the Service Usage Admin or Owner role:
  ```bash
  gcloud services enable aiplatform.googleapis.com cloudresourcemanager.googleapis.com \
    --project=<PROJECT_ID>
  ```
  If the user doesn't have permission, they should send this command to their project admin.

### 3. Prepare the Repo
Every `cm` command:
- Runs from the root of the repo being scanned.
- Is prefixed with `CM_HOME="$PWD/.codemender"`, so each repo keeps its own CodeMender state, findings, and logs.
- Is also prefixed with `GOOGLE_CLOUD_PROJECT=<PROJECT_ID>` if it talks to the backend (`health`, `init`, `find`, `find verify`, `fix`).

Before the first `cm` command, the agent adds `.codemender/` to the repo's `.gitignore` (only if it isn't already ignored), then tells the user it did:
```bash
git check-ignore -q .codemender/ || {
  [ -s .gitignore ] && [ -n "$(tail -c1 .gitignore)" ] && echo >> .gitignore
  echo ".codemender/" >> .gitignore
}
```
This keeps CodeMender's state out of commits and preserves it when `cm` cleans the working tree (e.g. during `cm find verify`).

### 4. Verify Access & Health
`cm health` checks credentials, config, reachability, and authorization in one go. The agent uses it as the access check:
```bash
CM_HOME="$PWD/.codemender" GOOGLE_CLOUD_PROJECT=<PROJECT_ID> cm health
```

Confirm the `Project:` line in the output matches `<PROJECT_ID>` before scanning.

- **Credentials expired or missing:** the user runs:
  ```bash
  gcloud auth application-default login
  ```
- **HTTP 403 / authorization failed:** check these in order:
  1. **Missing IAM role.** The signed-in account needs `roles/aiplatform.user` (Agent Platform User). A project admin can grant it:
     ```bash
     gcloud projects add-iam-policy-binding <PROJECT_ID> \
       --member="user:<EMAIL>" \
       --role="roles/aiplatform.user"
     ```
     `cm` uses Application Default Credentials, so `<EMAIL>` is the account the user signed in with during `gcloud auth application-default login`. That may differ from `gcloud config get-value account`.
  2. **Project not allowlisted.** CodeMender is in limited Public Preview. If the role is in place and it still fails, the project isn't allowlisted. The agent asks for another allowlisted project, or tells the user to request access through their Google Cloud sales team.

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
CM_HOME="$PWD/.codemender" GOOGLE_CLOUD_PROJECT=<PROJECT_ID> cm find <TARGET> \
  --model <MODEL_NAME> \
  [--quick] \
  [-c "<CONTEXT>"] \
  -y
```

Once complete, the agent automatically exports the report files:
```bash
CM_HOME="$PWD/.codemender" cm report --format json > <REPORT_DIR>/findings.json
CM_HOME="$PWD/.codemender" cm report --format table > <REPORT_DIR>/summary.txt
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
   - **Prove Exploitability**: `CM_HOME="$PWD/.codemender" GOOGLE_CLOUD_PROJECT=<PROJECT_ID> cm find verify <finding-id>` (Generates and runs an active PoC exploit in an isolated sandbox).
   - **Auto-Fix**: `CM_HOME="$PWD/.codemender" GOOGLE_CLOUD_PROJECT=<PROJECT_ID> cm fix <finding-id>` (Generates a verified, non-regressing patch).
   - **Open Visual Report**: `CM_HOME="$PWD/.codemender" cm report --format html --open` (Opens full interactive HTML report in browser).
