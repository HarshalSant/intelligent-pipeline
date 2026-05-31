# IntelliPipeline

> **AI-Native CI/CD Autopilot** — Enterprise-grade intelligent pipeline with SAST scanning, secret detection, CVE analysis, auto-fix, GitOps drift detection, Kubernetes cognitive layer, and pre-deploy risk scoring.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-green.svg)](https://fastapi.tiangolo.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Helm%20Ready-326CE5.svg)](helm/)

---

## What is IntelliPipeline?

IntelliPipeline is an **AI-native CI/CD intelligence layer** that sits on top of any existing pipeline (GitHub Actions, GitLab CI, Jenkins) and adds capabilities that traditional tools simply do not have:

- **Thinks before it deploys** — a 5-dimension pre-deploy risk score produces one of four decisions: `APPROVE`, `CANARY`, `HOLD`, or `BLOCK`, before a single container is ever pushed to production.
- **Understands your code, not just your syntax** — SAST vulnerability detection covers OWASP Top 10 with file-specific context, not just generic warnings.
- **Fixes what it finds** — an LLM-backed auto-fix engine generates targeted code patches for discovered vulnerabilities and failed build logs, with rule-based fallback when no API key is configured.
- **Watches your cluster** — a GitOps drift detector compares Git state against live ArgoCD/Flux state; a Kubernetes health monitor spots CrashLoopBackOff, OOMKill, and Pending pods and predicts scale-up needs before load arrives.
- **Works without Docker** — the entire platform runs as a single process with `python dev_server.py`. No external services required.

---

## Architecture Overview

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                   IntelliPipeline API  (FastAPI · Port 9001)         │
  │              REST + Webhook ingestion + /docs (Swagger UI)           │
  └──────┬──────────┬──────────┬──────────┬──────────┬──────────────────┘
         │          │          │          │          │
  ┌──────▼──┐ ┌─────▼──┐ ┌────▼────┐ ┌───▼────┐ ┌───▼────────────────┐
  │ Scanner │ │ Risk   │ │ AutoFix │ │ GitOps │ │  Kubernetes Layer  │
  │  Layer  │ │ Scorer │ │ Engine  │ │ Drift  │ │  Health + Predict  │
  └──────┬──┘ └─────┬──┘ └────┬────┘ └───┬────┘ └───┬────────────────┘
         │          │          │          │           │
  ┌──────▼──────────▼──────────▼──────────▼───────────▼────────────────┐
  │                      Pipeline Orchestrator                           │
  │   Stage 1: Secret Scan  →  Stage 2: SAST  →  Stage 3: Config Audit  │
  │   Stage 4: Dep Scan     →  Stage 5: Risk  →  Stage 6: AutoFix       │
  │   Stage 7: Optimize                                                  │
  └─────────────────────────────────────────────────────────────────────┘
         │                                          │
  ┌──────▼──────────────────┐    ┌──────────────────▼──────────────────┐
  │    Event Bus             │    │   Integrations                      │
  │  GitHub Push / PR        │    │   GitHub · GitLab · Jenkins         │
  │  GitLab Push             │    │   ArgoCD · Flux · Kubernetes        │
  │  Jenkins Trigger         │    └─────────────────────────────────────┘
  │  Manual / Scheduled      │
  └──────────────────────────┘
```

### How a pipeline run flows

```
  Git Push / PR
       │
       ▼
  [Webhook] ──► Event Bus ──► Orchestrator
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
               [Stage 1]      [Stage 2]      [Stage 3]
              Secret Scan    SAST / OWASP   Config Audit
              22 patterns    12 rule sets   Dockerfile +
              + entropy       OWASP Top 10  CI YAML + Helm
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
               [Stage 4]      [Stage 5]      [Stage 6]
               Dep Scan       Risk Score     Auto-Fix
               30-CVE DB     5-dimension    LLM patches
               + pinning      0-100 score   + rule fallback
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                              [Stage 7]
                              Optimize
                              CI YAML + build
                              history analysis
                                   │
                              ┌────▼────┐
                              │Decision │
                              │APPROVE  │
                              │CANARY   │
                              │HOLD     │
                              │BLOCK    │
                              └─────────┘
```

---

## Intelligence Modules

### Scanner Layer

| Module | What it does | Depth |
|--------|-------------|-------|
| `SecretScanner` | Detects leaked credentials, API keys, tokens | 22 regex patterns + Shannon entropy analysis |
| `VulnerabilityScanner` | SAST for OWASP Top 10 vulnerabilities | 12 rule sets: SQLi, CMDi, XSS, Path Traversal, XXE, SSRF, unsafe deserialization, hardcoded secrets, debug flags, unsafe YAML/pickle/eval |
| `ConfigAuditor` | Audits Dockerfile, GitHub Actions YAML, Helm values | Auto-detects config type; flags `FROM latest`, `USER root`, exposed secrets, missing health checks, privileged containers |
| `DependencyScanner` | CVE lookup for Python, Node, Java packages | Embedded 30-CVE database; parses `requirements.txt` directly |

### Risk Scorer

5-dimension weighted model that produces a score from 0–100:

| Dimension | Weight | What it measures |
|-----------|--------|-----------------|
| `branch_risk` | 20% | Production branch (`main`, `master`, `release/*`) vs feature branch |
| `finding_risk` | 30% | Severity-weighted count of all scanner findings |
| `change_risk` | 20% | Auth, infra, payment, dependency manifest, or k8s file changes |
| `history_risk` | 15% | Recent failure rate, incidents, and rollbacks on this repo |
| `blast_radius` | 15% | Number of dependent services and affected user count |

**Deployment decisions:**

| Score | Level | Decision | Action |
|-------|-------|----------|--------|
| 0–25 | LOW | `APPROVE` | Deploy directly |
| 26–50 | MEDIUM | `APPROVE` | Deploy with standard monitoring |
| 51–70 | HIGH | `CANARY` | Canary at 10–25% first |
| 71–85 | VERY_HIGH | `HOLD` | Wait for off-peak + senior review |
| 86–100 | CRITICAL | `BLOCK` | Block until findings resolved |

### Auto-Fix Engine

- **Build failure analysis** — parses CI error logs to identify error type (import error, test failure, compile error, dependency conflict) and generates targeted fix steps
- **LLM-powered patches** — uses Claude (`claude-sonnet-4-6`) when `ANTHROPIC_API_KEY` is set to generate context-aware code patches
- **Rule-based fallback** — pattern-matched fix templates for common failure classes require zero API credentials
- **PR generator** — wraps auto-fix suggestions into a structured draft PR payload ready for GitHub/GitLab

### Pipeline Optimizer

Analyses CI pipeline YAML and build history to surface:
- Missing dependency caching (pip, npm, Maven, Gradle)
- Unpinned action versions (`@main` → `@v3`)
- Sequential stages that can run in parallel
- Missing test parallelisation
- Oversized Docker build contexts
- Estimates **monthly cost savings** in USD and minutes based on `runs_per_day`

### GitOps Drift Detector

Compares Git-desired state against live cluster state for:
- **ArgoCD** — app `sync.status` (OutOfSync → drift event) and `health.status` (Degraded → health alert)
- **Flux** — `Kustomization` ready/suspended state and reconciliation failures
- **Raw YAML diff** — line-by-line comparison of desired vs actual manifests
- Produces a **reconciliation plan** with prioritised actions (critical → high → medium)

### Kubernetes Intelligence Layer

**Health Monitor**
- Detects `CrashLoopBackOff` (exit code 137 = OOMKill)
- High restart count alerts (>10 restarts)
- Pending pod diagnosis (insufficient CPU/memory/nodes)
- Resource over-utilisation alerts (>80% CPU or memory)
- Cluster health summary: `healthy` / `degraded` / `critical`

**Predictive Scaler**
- Exponential smoothing (α = 0.3) over a CPU/memory metrics history window
- Time-of-day seasonality factor (peak hours 9–18 = 1.3×, off-peak = 0.8×)
- Scale-up trigger: predicted utilisation >70% → recommends replica count
- Scale-down trigger: predicted utilisation <30% → recommends reducing replicas
- **Right-sizing** — recommends CPU/memory request adjustments based on observed usage patterns

### Integrations

| System | Capabilities |
|--------|-------------|
| **GitHub** | Webhook ingestion (push + pull_request), PR comment creation, status check updates |
| **GitLab** | Push event ingestion, MR status, pipeline trigger |
| **Jenkins** | Pipeline trigger via REST API, build status polling |

---

## API Endpoints

All endpoints return JSON. Interactive docs at `http://localhost:9001/docs`.

### Scanning

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/scan/code` | SAST vulnerability scan on source code |
| `POST` | `/api/v1/scan/secrets` | Secret / credential leak detection |
| `POST` | `/api/v1/scan/config` | Dockerfile / CI YAML / Helm config audit |
| `POST` | `/api/v1/scan/dependencies` | CVE scan for package dependencies |
| `POST` | `/api/v1/scan/requirements-txt` | Upload and scan a raw `requirements.txt` |

### Pipeline

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/pipeline/run` | Execute full 7-stage pipeline analysis |
| `GET` | `/api/v1/pipeline/runs` | List recent pipeline runs |
| `GET` | `/api/v1/pipeline/runs/{run_id}` | Get details for a specific run |

### Auto-Fix & Optimization

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/autofix/build-failure` | Analyse CI error log and generate fix suggestions |
| `POST` | `/api/v1/optimize/pipeline` | Optimise CI YAML with savings estimate + draft PR |

### Risk

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/risk/score` | 5-dimension pre-deploy risk score + `APPROVE/CANARY/HOLD/BLOCK` decision |

### GitOps

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/gitops/drift` | Detect drift across ArgoCD apps, Flux kustomizations, or raw YAML diff |

### Kubernetes

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/k8s/health` | Analyse pod status and resource metrics for health issues |
| `POST` | `/api/v1/k8s/predict-scaling` | Predict load spikes and recommend pre-emptive scaling |

### Webhooks

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/webhook/github` | Ingest a GitHub push or pull_request webhook event |
| `GET` | `/api/v1/webhook/events` | List recent received webhook events |

### Analytics

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/analytics/summary` | Global stats: runs, decisions, findings, risk distribution |
| `GET` | `/api/v1/analytics/audit-log` | Full audit log of all IntelliPipeline actions |

### Demo

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/demo/seed` | Seeds realistic demo data across all capabilities |

---

## Project Structure

```
intelligent-pipeline/
│
├── dev_server.py              # All-in-one FastAPI server — single entry point
├── requirements.txt           # Python dependencies
├── pytest.ini                 # Test configuration
├── Dockerfile                 # Production container (non-root, healthcheck)
├── validate.py                # Compile + encoding + stub validation script
│
├── core/
│   ├── config.py              # All env-var configuration and constants
│   ├── event_bus.py           # Event ingestion and routing (GitHub/GitLab/Jenkins/manual)
│   └── orchestrator.py        # 7-stage pipeline orchestrator, PipelineRun lifecycle
│
├── scanners/
│   ├── vulnerability.py       # SAST — 12 OWASP rule sets
│   ├── secrets.py             # Secret/credential detection — 22 patterns + entropy
│   ├── config_auditor.py      # Dockerfile / CI YAML / Helm misconfiguration audit
│   └── dependency.py         # CVE scan — 30-CVE embedded DB + requirements.txt parser
│
├── autofix/
│   ├── bug_fixer.py           # Build failure analyser + LLM patch generator
│   ├── optimizer.py           # CI pipeline optimiser with monthly savings estimate
│   └── pr_generator.py        # Draft PR payload builder from fix/optimise results
│
├── risk/
│   └── scorer.py              # 5-dimension risk scorer → APPROVE/CANARY/HOLD/BLOCK
│
├── gitops/
│   └── drift_detector.py      # ArgoCD / Flux / raw YAML drift detection + reconciliation plan
│
├── k8s/
│   ├── health_monitor.py      # Pod health, CrashLoopBackOff, OOM, resource utilisation
│   └── predictor.py           # Exponential smoothing predictive scaler + right-sizing
│
├── integrations/
│   ├── github.py              # GitHub API client (webhooks, PR comments, status checks)
│   ├── gitlab.py              # GitLab API client (push events, MR status)
│   └── jenkins.py             # Jenkins REST client (trigger, poll, status)
│
├── tests/
│   ├── test_vulnerability_scanner.py
│   ├── test_secret_scanner.py
│   ├── test_config_auditor.py
│   ├── test_dependency_scanner.py
│   ├── test_risk_scorer.py
│   ├── test_orchestrator.py
│   ├── test_gitops_drift.py
│   └── test_k8s_health.py
│
└── helm/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── hpa.yaml
        └── secrets.yaml
```

---

## Quick Start (No Docker Required)

The entire platform runs as a single process — no databases, no message brokers, no containers needed.

### Prerequisites

- Python 3.11 or higher
- pip

### 1. Clone the repository

```bash
git clone https://github.com/HarshalSant/intelligent-pipeline.git
cd intelligent-pipeline
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. (Optional) Set environment variables

```bash
# For LLM-powered auto-fix and enhanced analysis
export ANTHROPIC_API_KEY=your_key_here

# For GitHub webhook verification and PR creation
export GITHUB_TOKEN=your_github_token
export GITHUB_WEBHOOK_SECRET=your_webhook_secret

# For GitLab integration
export GITLAB_TOKEN=your_gitlab_token
```

On Windows (PowerShell):
```powershell
$env:ANTHROPIC_API_KEY = "your_key_here"
$env:GITHUB_TOKEN = "your_github_token"
```

### 4. Start the server

```bash
python dev_server.py
```

```
============================================================
  IntelliPipeline v1.0.0
  AI-Native CI/CD Autopilot
============================================================
  API:  http://localhost:9001
  Docs: http://localhost:9001/docs
  Demo: POST http://localhost:9001/api/v1/demo/seed
============================================================
```

### 5. Open the interactive API docs

```
http://localhost:9001/docs
```

Swagger UI lets you explore and call every endpoint directly in the browser — no curl required.

### 6. Seed demo data (optional)

```bash
curl -X POST http://localhost:9001/api/v1/demo/seed
```

This runs all capabilities against realistic sample data and returns a summary like:

```json
{
  "vulnerabilities_found": 4,
  "secrets_found": 3,
  "config_issues": 6,
  "cves_found": 5,
  "risk_score": 78,
  "deploy_decision": "hold",
  "optimizer_suggestions": 4,
  "gitops_drift_events": 2,
  "k8s_issues": 2,
  "scaling_action_needed": true
}
```

---

## Usage Examples

### Run a full pipeline analysis

```bash
curl -X POST http://localhost:9001/api/v1/pipeline/run \
  -H "Content-Type: application/json" \
  -d '{
    "repository": "myorg/my-service",
    "branch": "main",
    "commit_sha": "abc1234",
    "code": "import os\ndef run(cmd): os.system(cmd)",
    "config_content": "FROM ubuntu:latest\nUSER root\nCOPY . .\n",
    "dependencies": [
      {"name": "django", "version": "3.2.0", "ecosystem": "python"}
    ],
    "files_changed": ["auth/views.py", "requirements.txt"],
    "triggered_by": "developer"
  }'
```

**Response:**
```json
{
  "run_id": "a3f7c1b2",
  "repository": "myorg/my-service",
  "branch": "main",
  "risk_score": 72.4,
  "deploy_decision": "canary",
  "stages": [
    {"stage": "secret_scan", "passed": true, "finding_count": 0},
    {"stage": "vuln_scan",   "passed": false, "finding_count": 1},
    {"stage": "config_audit","passed": false, "finding_count": 4},
    {"stage": "dep_scan",    "passed": false, "finding_count": 2},
    {"stage": "risk_score",  "passed": true},
    {"stage": "auto_fix",    "passed": true, "fixes_applied": [...]},
    {"stage": "optimize",    "passed": true}
  ]
}
```

### Scan code for secrets

```bash
curl -X POST http://localhost:9001/api/v1/scan/secrets \
  -H "Content-Type: application/json" \
  -d '{
    "code": "GITHUB_TOKEN = \"ghp_abc123DEF456ghi789\"",
    "file_path": "config.py"
  }'
```

### Score pre-deploy risk

```bash
curl -X POST http://localhost:9001/api/v1/risk/score \
  -H "Content-Type: application/json" \
  -d '{
    "branch": "main",
    "files_changed": ["auth/models.py", "Dockerfile"],
    "service_count": 4,
    "user_count": 10000
  }'
```

### Fix a build failure

```bash
curl -X POST http://localhost:9001/api/v1/autofix/build-failure \
  -H "Content-Type: application/json" \
  -d '{
    "error_log": "ModuleNotFoundError: No module named '\''httpx'\''",
    "source_files": {"requirements.txt": "fastapi==0.115.0\nuvicorn==0.30.6"}
  }'
```

### Detect GitOps drift

```bash
curl -X POST http://localhost:9001/api/v1/gitops/drift \
  -H "Content-Type: application/json" \
  -d '{
    "argocd_apps": [
      {
        "metadata": {"name": "payment-service", "namespace": "argocd"},
        "status": {
          "sync": {"status": "OutOfSync"},
          "health": {"status": "Healthy"}
        }
      }
    ]
  }'
```

---

## Configuration Reference

All configuration is via environment variables. No config files to manage.

| Variable | Default | Description |
|----------|---------|-------------|
| `INTELLIPIPELINE_PORT` | `9001` | HTTP server port |
| `ANTHROPIC_API_KEY` | `""` | Enables LLM-powered auto-fix and analysis |
| `LLM_MODEL` | `claude-sonnet-4-6` | Claude model for LLM features |
| `GITHUB_TOKEN` | `""` | GitHub API token for PR creation and status checks |
| `GITHUB_WEBHOOK_SECRET` | `""` | Webhook HMAC verification secret |
| `GITLAB_TOKEN` | `""` | GitLab API token |
| `JENKINS_URL` | `http://jenkins:8080` | Jenkins base URL |
| `JENKINS_TOKEN` | `""` | Jenkins API token |
| `ARGOCD_URL` | `http://argocd-server:80` | ArgoCD server URL |
| `ARGOCD_TOKEN` | `""` | ArgoCD API token |
| `FLUX_NAMESPACE` | `flux-system` | Flux system namespace |
| `RISK_AUTO_DEPLOY_MAX` | `30` | Max risk score for auto-approve |
| `RISK_CANARY_MAX` | `65` | Max risk score for canary |
| `RISK_BLOCK_MIN` | `85` | Min risk score for hard block |
| `VULN_CRITICAL_BLOCK` | `true` | Block pipeline on CRITICAL vulnerabilities |
| `SECRET_SCAN_ENABLED` | `true` | Enable secret scanning stage |
| `DEPENDENCY_SCAN_ENABLED` | `true` | Enable dependency CVE scanning |
| `OPTIMIZER_MIN_SAVINGS_SECONDS` | `30` | Minimum saving threshold for optimizer suggestions |

---

## Running Tests

```bash
# Run all tests
pytest

# Run with verbose output
pytest -v

# Run a specific module
pytest tests/test_risk_scorer.py -v

# Run with coverage
pytest --cov=. --cov-report=term-missing
```

Test coverage spans all intelligence modules:

| Test File | What it covers |
|-----------|---------------|
| `test_vulnerability_scanner.py` | SQL injection, command injection, XSS, path traversal, hardcoded secrets |
| `test_secret_scanner.py` | GitHub tokens, AWS keys, private keys, entropy detection, false-positive resistance |
| `test_config_auditor.py` | Dockerfile `FROM latest`, `USER root`, exposed secrets, Helm Ingress, GitHub Actions pinning |
| `test_dependency_scanner.py` | Known CVEs, clean packages, `requirements.txt` parsing |
| `test_risk_scorer.py` | All 5 dimensions, prod vs feature branch, decision thresholds |
| `test_orchestrator.py` | Full pipeline runs, secret-triggered block, stage sequencing |
| `test_gitops_drift.py` | ArgoCD OutOfSync, Flux suspended, raw YAML diff |
| `test_k8s_health.py` | CrashLoopBackOff, OOMKill, Pending pods, predictive scaling |

---

## Docker

### Build the image

```bash
docker build -t intellipipeline:1.0.0 .
```

### Run the container

```bash
docker run -p 9001:9001 \
  -e ANTHROPIC_API_KEY=your_key \
  -e GITHUB_TOKEN=your_token \
  intellipipeline:1.0.0
```

The container runs as a non-root user (`appuser`, UID 1000) with a built-in healthcheck:

```bash
docker inspect intellipipeline --format='{{.State.Health.Status}}'
# healthy
```

---

## Kubernetes Deployment (Helm)

### Prerequisites

- Kubernetes cluster (1.25+)
- Helm 3.x installed
- `kubectl` configured

### Install

```bash
# From the repo root
helm install intellipipeline ./helm \
  --set secrets.anthropicApiKey=your_key \
  --set secrets.githubToken=your_token
```

### Install with custom values

```bash
helm install intellipipeline ./helm \
  --set replicaCount=3 \
  --set autoscaling.enabled=true \
  --set autoscaling.maxReplicas=10 \
  --set config.riskAutoDeployMax=25 \
  --set secrets.anthropicApiKey=your_key
```

### Upgrade

```bash
helm upgrade intellipipeline ./helm --reuse-values \
  --set image.tag=1.1.0
```

### Uninstall

```bash
helm uninstall intellipipeline
```

### Helm values reference

| Value | Default | Description |
|-------|---------|-------------|
| `replicaCount` | `2` | Number of pod replicas |
| `image.repository` | `intellipipeline` | Container image name |
| `image.tag` | `1.0.0` | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.port` | `9001` | Service port |
| `ingress.enabled` | `false` | Enable Ingress |
| `ingress.className` | `nginx` | Ingress class |
| `ingress.host` | `intellipipeline.example.com` | Ingress hostname |
| `autoscaling.enabled` | `true` | Enable HPA |
| `autoscaling.minReplicas` | `2` | HPA minimum replicas |
| `autoscaling.maxReplicas` | `8` | HPA maximum replicas |
| `autoscaling.targetCPUUtilizationPercentage` | `70` | HPA CPU target |
| `config.riskAutoDeployMax` | `"30"` | Risk threshold for auto-approve |
| `config.riskCanaryMax` | `"65"` | Risk threshold for canary |
| `config.riskBlockMin` | `"85"` | Risk threshold for block |
| `resources.requests.cpu` | `250m` | Pod CPU request |
| `resources.requests.memory` | `256Mi` | Pod memory request |
| `resources.limits.cpu` | `1000m` | Pod CPU limit |
| `resources.limits.memory` | `512Mi` | Pod memory limit |
| `secrets.anthropicApiKey` | `""` | Anthropic API key (stored in K8s Secret) |
| `secrets.githubToken` | `""` | GitHub token (stored in K8s Secret) |
| `secrets.githubWebhookSecret` | `""` | Webhook HMAC secret |

---

## Connecting to Your CI/CD Pipeline

### GitHub Actions

Add IntelliPipeline as a step in your workflow:

```yaml
# .github/workflows/ci.yml
- name: IntelliPipeline Analysis
  run: |
    curl -X POST http://your-intellipipeline-host:9001/api/v1/pipeline/run \
      -H "Content-Type: application/json" \
      -d "{
        \"repository\": \"${{ github.repository }}\",
        \"branch\": \"${{ github.ref_name }}\",
        \"commit_sha\": \"${{ github.sha }}\",
        \"triggered_by\": \"github_actions\"
      }" | tee result.json

    # Block merge if risk score is too high
    DECISION=$(cat result.json | python -c "import sys,json; print(json.load(sys.stdin)['deploy_decision'])")
    if [ "$DECISION" = "block" ]; then
      echo "IntelliPipeline blocked deployment — risk score critical"
      exit 1
    fi
```

### GitHub Webhooks

1. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
2. Set Payload URL to `http://your-host:9001/api/v1/webhook/github`
3. Content type: `application/json`
4. Select events: **Pushes** and **Pull requests**
5. IntelliPipeline will automatically trigger a pipeline analysis on each event

### GitLab CI

```yaml
# .gitlab-ci.yml
intellipipeline:
  stage: .pre
  script:
    - |
      curl -X POST http://your-intellipipeline-host:9001/api/v1/pipeline/run \
        -H "Content-Type: application/json" \
        -d "{\"repository\": \"$CI_PROJECT_PATH\", \"branch\": \"$CI_COMMIT_REF_NAME\"}"
```

---

## What Makes This Different

| Feature | Traditional Tools | IntelliPipeline |
|---------|-----------------|-----------------|
| SAST scanning | File-by-file rules | Context-aware OWASP analysis with fixable flag |
| Secret detection | Regex only | Regex + Shannon entropy (catches obfuscated secrets) |
| Deployment decision | Manual gates | Automated 5-dimension risk score |
| Failed build fix | Developer Google search | LLM-generated patch with rule-based fallback |
| GitOps drift | Alert fatigue | Prioritised reconciliation plan |
| K8s scaling | Reactive HPA | Predictive with time-of-day seasonality |
| Integration | Tool-specific plugins | Single REST API, any CI/CD system |
| Setup | Container per feature | One `python dev_server.py` |

---

## Roadmap

| Feature | Status |
|---------|--------|
| Core scanning (SAST, secrets, config, deps) | **Complete** |
| 5-dimension risk scorer | **Complete** |
| LLM auto-fix + rule-based fallback | **Complete** |
| Pipeline YAML optimizer | **Complete** |
| GitOps drift detection (ArgoCD + Flux) | **Complete** |
| Kubernetes health + predictive scaler | **Complete** |
| GitHub / GitLab / Jenkins integrations | **Complete** |
| Helm chart | **Complete** |
| Persistent storage (PostgreSQL) | Planned |
| Real-time WebSocket notifications | Planned |
| Policy-as-code engine | Planned |
| SBOM generation | Planned |
| Multi-tenant workspace isolation | Planned |

---

## License

Apache 2.0 — see [LICENSE](LICENSE)
