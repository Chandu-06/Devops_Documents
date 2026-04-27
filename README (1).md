# Log Generator — DevOps POC

A CI/CD pipeline project built as part of the HSBC Technology DevOps onboarding programme.

The application itself is intentionally simple — a Bash script that generates a timestamped log file. The focus of this project is the **full DevOps toolchain** wrapped around it: containerisation, private image registry, automated CI/CD, and Kubernetes deployment.

---

## What This Project Demonstrates

| Area | Tool |
|---|---|
| Source Control & Branching | Git + GitHub Enterprise (Git Flow) |
| Containerisation | Docker |
| Private Image Registry | Nexus3 (internal HSBC registry) |
| CI/CD Automation | Jenkins Declarative Pipeline |
| Container Orchestration | Kubernetes |

---

## Repository Structure

```
log-generator/
├── generate_log.sh       # Bash script — generates application.log
├── Dockerfile            # Container image definition
├── Jenkinsfile           # CI/CD pipeline (all stages)
└── k8s/
    └── deployment.yaml   # Kubernetes Deployment manifest
```

---

## How It Works

Every push to the `develop` branch triggers the Jenkins pipeline automatically via a GitHub webhook.

```
Git Push (develop)
      │
      ▼
Jenkins Pipeline
      │
      ├── Checkout          → pulls latest code from GitHub Enterprise
      ├── Docker Login      → authenticates to internal Nexus3 registry
      ├── Docker Build      → builds image tagged log-generator:<BUILD_NUMBER>
      ├── Docker Run        → runs container, generates application.log
      ├── Archive Log       → saves log as a downloadable Jenkins artifact
      ├── Push to Nexus     → pushes image to Nexus3 private registry
      └── Deploy to K8s     → updates Kubernetes deployment (rolling update)
```

---

## Branching Strategy

This project follows **Git Flow**.

```
feature/log-generator
        │
        ▼ Pull Request
     develop          ← active development branch (pipeline triggers here)
        │
        ▼
   release/v1.0
        │
        ▼
     master           ← stable, production-ready code only
```

- **Never push directly to `master`**
- All work starts on a `feature/*` branch
- Changes reach `develop` via a Pull Request

---

## Application

**`generate_log.sh`** — writes four timestamped lines to `application.log`:

```
[2026-04-21 10:35:22] INFO - Application started
[2026-04-21 10:35:22] INFO - Processing request
[2026-04-21 10:35:22] INFO - Request completed successfully
[2026-04-21 10:35:22] INFO - Application finished
```

The log file is generated inside the Docker container and copied out to the Jenkins workspace, then archived as a build artifact.

---

## Docker

The container image is built from the `Dockerfile` in this folder.

> **Note:** This project uses a company-approved base image from the internal Nexus3 registry — not a public Docker Hub image. The `FROM` line in the Dockerfile references the internal Nexus URL.

Image tag format:
```
nexus3.systems.uk.hsbc:<port>/log-generator:<BUILD_NUMBER>
```

Each Jenkins build produces a uniquely tagged image. This makes every image traceable back to a specific build and Git commit.

---

## Jenkins Pipeline

**Job name:** `log-generator-pipeline`  
**Job type:** Pipeline  
**Script path:** `log-generator/Jenkinsfile`  
**Trigger:** GitHub webhook on push to `develop`

### Pipeline Stages

| Stage | What It Does |
|---|---|
| Checkout | Pulls latest code from GitHub Enterprise |
| Docker Login | Authenticates to Nexus3 using a stored Service Account credential |
| Docker Build | Builds the Docker image tagged with Jenkins `BUILD_NUMBER` |
| Docker Run | Runs the container and copies `application.log` to the workspace |
| Archive Log | Saves the log file as a downloadable Jenkins build artifact |
| Push to Nexus | Tags and pushes the image to the Nexus3 private registry |
| Deploy to Kubernetes | Applies `k8s/deployment.yaml` and confirms rollout success |

### Credentials

Nexus3 credentials are stored securely in **Jenkins → Manage Jenkins → Credentials** using the ID `Service Account`. No passwords are stored in code.

---

## Kubernetes

The app is deployed to the `devops-poc` namespace on the company Kubernetes cluster.

**`k8s/deployment.yaml`** defines a Deployment with:
- 1 replica
- Image pulled from Nexus3 (authenticated via `nexus-pull-secret`)
- Rolling update strategy — zero downtime on every deploy

Jenkins replaces the `BUILD_NUMBER_PLACEHOLDER` in the YAML at deploy time, so the running pod always reflects the exact build number of the image it is running.

### Useful Commands

```bash
# Check deployment status
kubectl get deployments -n devops-poc

# Check running pods
kubectl get pods -n devops-poc

# View pod logs
kubectl logs <pod-name> -n devops-poc

# Check rollout status
kubectl rollout status deployment/log-generator -n devops-poc
```

---

## Getting Started (New Joiners)

### Prerequisites

Before you can work on this project, make sure you have:

- [ ] Access to the internal GitHub Enterprise (`alm-github.systems.uk.hsbc`)
- [ ] Access to the company Jenkins server
- [ ] Access to Nexus3 registry (raise an IT ticket if not)
- [ ] Docker Desktop installed on your machine
- [ ] `kubectl` configured with the cluster kubeconfig file

### Clone and Set Up

```bash
# Clone the repo
git clone <internal-GHE-URL>/devops_poc.git
cd devops_poc/log-generator

# Create your feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/your-task-name
```

### Making Changes

```bash
# After making your changes
git add .
git commit -m "Brief description of what you changed"
git push origin feature/your-task-name

# Then raise a Pull Request on GitHub Enterprise:
# base: develop  ←  compare: feature/your-task-name
```

Once the PR is merged to `develop`, Jenkins will automatically trigger and run the full pipeline.

---

## Full Documentation

Detailed step-by-step documentation for this project (with screenshot guidance, troubleshooting, and design decisions) is available on the team Confluence page:

> **Confluence:** [Add your Confluence page link here]

---

## Contact

For questions about this project or the DevOps setup, reach out to the Payment Data Programme DevOps team on the internal HSBC channels.


cd devops_poc/log-generator
# paste the README.md content here
git add README.md
git commit -m "Add README for log-generator project"
git push origin develop