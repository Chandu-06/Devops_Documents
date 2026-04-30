# 📄 File 4: `Jenkinsfile` — Line by Line Explanation

---

## 🔎 What is this file?

A `Jenkinsfile` is a **pipeline-as-code** file that defines the entire CI/CD pipeline for a project. It lives in the root of your repository and tells Jenkins exactly what to do — step by step — when a build is triggered.

Instead of clicking through Jenkins UI to configure stages, you write everything in this file. This means your pipeline is:
- **Version-controlled** (stored in Git alongside your code)
- **Reproducible** (everyone gets the same pipeline)
- **Auditable** (you can see what changed and when)

This Jenkinsfile orchestrates the full lifecycle of our log-generator:
1. Checkout code
2. Build a Docker image
3. Run the container and generate logs
4. Push the image to Nexus3
5. Login to Kubernetes (OIDC)
6. Deploy to Kubernetes
7. Undeploy from Kubernetes
8. Archive logs and clean up

---

## 📋 Full Code with Line-by-Line Explanation

---

### Block 1 — Pipeline Declaration
```groovy
pipeline {
```

**What it does:**
The `pipeline` block is the **top-level wrapper** for a Jenkins Declarative Pipeline. Everything inside this block is part of the pipeline definition.

**Why Declarative Pipeline?**
Jenkins supports two styles:
| Style | Description |
|-------|-------------|
| **Declarative** (used here) | Structured, opinionated syntax. Easier to read and write. Has built-in validation. ✅ |
| **Scripted** | Groovy-based, more flexible but harder to read. For complex custom logic. |

Declarative is the **modern standard** and what HSBC teams use. It enforces structure (agent, stages, steps) which makes pipelines consistent and easier to maintain.

---

### Block 2 — Agent
```groovy
agent {
    label 'cm-linux-gce'
}
```

**What it does:**
Tells Jenkins **where to run this pipeline** — on which Jenkins agent/worker node.

`label 'cm-linux-gce'` means: *"Run this pipeline on any Jenkins agent that has the label `cm-linux-gce`."*

**Why this label?**
`cm-linux-gce` refers to a **Linux agent running on Google Compute Engine (GCE)** within HSBC's internal infrastructure. This is a **team/organisation decision** — the label was assigned by the Jenkins admin team to identify the appropriate build agents for this type of work.

**Why not `agent any`?**
`agent any` would run the pipeline on any available agent — including Windows agents, which don't support Linux shell commands (`bash`, `docker`, etc.). By specifying the label, we ensure the pipeline always runs on a Linux agent that has Docker and the required tools installed.

**Alternative approach:**
```groovy
agent any          // Run on any agent
agent none         // Declare agents per-stage
agent {
    docker { image '...' }  // Run inside a Docker container
}
```

---

### Block 3 — Environment Variables
```groovy
environment {
    NEXUS_REPO        = "Confidential"
    IMAGE_NAME        = "log-generator"
    IMAGE_TAG         = "${BUILD_NUMBER}"
    NEXUS3_URL        = "Confidential"
    NEXUS3_APP_PATH   = "Confidential"
    ENVIRONMENT       = "dev"
    MANIFEST_PATH     = "k8s/deployment.yaml"
    KUBE_CREDS        = "Service Account"
    idpapi            = "https://oidc.ikp1<Confidential>"
    jqcli             = "/usr/lib/<Confidential>"
    kubectlcli        = "/usr/lib/kubectl<Confidential>/kubectl"
    apicert           = "LS0tLSICRUdJTIBD<Confidential>"
    idpcert           = "LS0tLSICRUDITIBDR<Confidential>"
    apiserver         = "https://userapi-ikp1<Confidential>"
    idpissuer         = "https://oidc.ikpl<Confidential>"
    api_name          = "log-generator"
}
```

**What it does:**
The `environment` block defines **pipeline-wide environment variables** — values that are available to every stage and step in this pipeline.

---

**`NEXUS_REPO = "Confidential"`**
**[CONFIDENTIAL]** — The URL/name of the internal Nexus3 Docker registry. Used to login to the registry before pushing images.

---

**`IMAGE_NAME = "log-generator"`**
The name given to the Docker image being built. Used consistently across stages (build, tag, push, run).

---

**`IMAGE_TAG = "${BUILD_NUMBER}"`**
Tags the Docker image with the Jenkins build number.

**Why use `$BUILD_NUMBER` as the image tag?**
Every Jenkins build gets a unique, auto-incrementing number (1, 2, 3...). Using it as the image tag means:
- Every build produces a **uniquely tagged image**
- You can trace any running container back to the exact Jenkins build that created it
- You avoid the risk of overwriting a good image with a bad one

This is a **best practice for CI/CD traceability**.

**Alternative approach:**
Some teams use Git commit hash: `IMAGE_TAG = "${GIT_COMMIT[0..7]}"` — this links the image directly to the code commit that built it, which is even more traceable.

---

**`NEXUS3_URL = "Confidential"`**
**[CONFIDENTIAL]** — The full URL of the Nexus3 Docker registry (hostname + port). Used for `docker login` and `docker push`.

---

**`NEXUS3_APP_PATH = "Confidential"`**
**[CONFIDENTIAL]** — The internal registry path where the image will be pushed (e.g., `com/hsbc/payments/log-generator`).

---

**`ENVIRONMENT = "dev"`**
Identifies the target environment. Used in Kubernetes deployment — the Pod lands in the `dev` namespace. This makes the pipeline environment-aware without hardcoding values in multiple places.

---

**`MANIFEST_PATH = "k8s/deployment.yaml"`**
The relative path to the Kubernetes manifest file in the repository. Used in the deploy and undeploy stages. Centralising this here means if the file moves, you only update it in one place.

---

**`KUBE_CREDS = "Service Account"`**
**[CONFIDENTIAL]** — References the Jenkins credential ID for the Kubernetes service account. Used in `withCredentials` blocks to authenticate with the cluster.

---

**`idpapi`, `jqcli`, `kubectlcli`, `apicert`, `idpcert`, `apiserver`, `idpissuer`**
**[CONFIDENTIAL]** — These are all OIDC (OpenID Connect) authentication configuration values used in the OIDC Login stage:

| Variable | Purpose |
|----------|---------|
| `idpapi` | **[CONFIDENTIAL]** — Identity Provider API URL for OIDC token retrieval |
| `jqcli` | **[CONFIDENTIAL]** — Path to `jq` binary (JSON parsing tool) on the Jenkins agent |
| `kubectlcli` | **[CONFIDENTIAL]** — Path to `kubectl` binary on the Jenkins agent |
| `apicert` | **[CONFIDENTIAL]** — Base64-encoded Kubernetes API server TLS certificate |
| `idpcert` | **[CONFIDENTIAL]** — Base64-encoded Identity Provider TLS certificate |
| `apiserver` | **[CONFIDENTIAL]** — Kubernetes API server URL |
| `idpissuer` | **[CONFIDENTIAL]** — OIDC issuer URL for token validation |

**Why define these in `environment` and not hardcode in each stage?**
Centralising them here follows the **DRY (Don't Repeat Yourself) principle**. If the server URL changes, you update it once here — not in five different shell commands. It also makes the pipeline easier to read.

---

### Block 4 — Stage: Checkout
```groovy
stage('Checkout') {
    steps {
        echo "Checking out Source Code"
        checkout scm
    }
}
```

**What it does:**
This stage pulls the latest source code from the Git repository onto the Jenkins agent's workspace.

**`checkout scm`**
`scm` stands for **Source Control Management**. This special Jenkins command checks out the exact branch/commit that triggered the build — it automatically uses the repository URL and credentials configured in the Jenkins job.

**Why this approach?**
`checkout scm` is the standard Jenkins way to clone code. You don't need to write `git clone <url>` manually. Jenkins already knows the repository URL from the job configuration — `checkout scm` uses that.

**Alternative approach:**
```groovy
git url: 'https://alm-github.systems.uk.hsbc/...', branch: 'feature/log-generator'
```
This works but hardcodes the URL and branch. `checkout scm` is more flexible — the same Jenkinsfile works across any branch (feature, develop, main).

---

### Block 5 — Stage: Docker Build Image
```groovy
stage('Docker Build Image') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'Service_Account',
            passwordVariable: '<Confidential>',
            usernameVariable: '<Confidential>'
        )]) {
            sh 'docker login -u $(<Confidential>) -p $(<Confidential>) ${NEXUS_REPO}'
            sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} log-generator/'
        }
    }
}
```

**`withCredentials([...])`**

**What it does:**
Securely injects credentials from the **Jenkins Credentials Store** into the build environment as temporary environment variables.

**Why withCredentials?**
You should **never hardcode usernames and passwords** in a Jenkinsfile — it would be visible in Git history and logs. `withCredentials` fetches credentials at runtime, injects them as masked variables, and destroys them after the block completes. Jenkins also **masks these values in build logs** — they appear as `****`.

**`credentialsId: 'Service_Account'`**
**[CONFIDENTIAL]** — The ID of the credential stored in Jenkins Credentials Store. A Jenkins admin created this entry with the Nexus3 username and password.

**`passwordVariable` / `usernameVariable`**
**[CONFIDENTIAL]** — The names of the temporary environment variables that will hold the username and password during this block.

---

**`sh 'docker login -u $(...) -p $(...) ${NEXUS_REPO}'`**

**What it does:**
Authenticates with the Nexus3 Docker registry so that subsequent `docker push` commands are authorised.

**Why login before build?**
The base image in the Dockerfile comes from Nexus3 (private registry). Docker needs to be authenticated to pull that base image during the build. Without login, `docker build` would fail immediately when trying to fetch `FROM nexus3.systems.uk.hsbc:...`.

---

**`sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} log-generator/'`**

**What it does:**
Builds a Docker image from the `log-generator/` directory (which contains the Dockerfile and `generate_log.sh`).

Breaking it down:
| Part | Meaning |
|------|---------|
| `docker build` | Build a Docker image |
| `-t ${IMAGE_NAME}:${IMAGE_TAG}` | Tag it as `log-generator:5` (if build #5) |
| `log-generator/` | Build context — the directory containing the Dockerfile |

**Why tag with `IMAGE_TAG` ($BUILD_NUMBER)?**
Creates a unique, traceable image for every build. You can always identify which Jenkins run produced which image.

---

### Block 6 — Stage: Docker Run
```groovy
stage('Docker Run') {
    steps {
        sh 'docker run --name log-generator -e BUILD_NUMBER=${BUILD_NUMBER} ${IMAGE_NAME}:${IMAGE_TAG}'
        sh 'docker logs log-generator'
        sh 'docker logs log-generator > application.log'
        sh 'sleep 30'
    }
}
```

**`docker run --name log-generator -e BUILD_NUMBER=${BUILD_NUMBER} ${IMAGE_NAME}:${IMAGE_TAG}`**

**What it does:** Starts a container from our built image.

| Part | Meaning |
|------|---------|
| `--name log-generator` | Gives the container a fixed name so we can reference it in subsequent commands |
| `-e BUILD_NUMBER=${BUILD_NUMBER}` | Injects the Jenkins build number as an environment variable inside the container — this is how `generate_log.sh` accesses `$BUILD_NUMBER` |
| `${IMAGE_NAME}:${IMAGE_TAG}` | The image to run |

**Why `-e BUILD_NUMBER`?**
The `generate_log.sh` script uses `$BUILD_NUMBER` to print it in the log header. Environment variables don't automatically pass into Docker containers — you must explicitly inject them with `-e`.

---

**`docker logs log-generator`**
Prints the container's stdout (the log output from our script) to the Jenkins console. This lets you see the log in the Jenkins build log immediately.

**`docker logs log-generator > application.log`**
Redirects the same output into a file called `application.log`. This file will later be archived as a build artifact.

**`sleep 30`**
Waits 30 seconds.

**Why sleep?**
The script likely exits quickly, but `sleep 30` gives the container time to finish writing logs before we try to read them. This is a **safety buffer** — a simple but crude approach. A better approach would be to wait until the container exits with `docker wait log-generator`, but `sleep` is used here for simplicity.

---

### Block 7 — Stage: Push Docker Image to Nexus
```groovy
stage('Push Docker Image Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: '<Confidential>',
            passwordVariable: '<Confidential>',
            usernameVariable: '<Confidential>'
        )]) {
            sh 'docker login -u ${<Confidential>} -p ${<Confidential>} ${NEXUS3_URL}'
            sh 'docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS3_URL}/${NEXUS3_APP_PATH}:latest'
            sh 'docker push ${NEXUS3_URL}/${NEXUS3_APP_PATH}:latest'
        }
    }
}
```

**`docker login`** — Re-authenticates specifically to Nexus3 for pushing (same reason as before).

---

**`docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS3_URL}/${NEXUS3_APP_PATH}:latest`**

**What it does:** Creates a new tag for the existing image that includes the full Nexus3 registry path.

**Why tag?**
Docker uses the image name to determine **where to push**. The image must be tagged with the full registry URL before pushing, otherwise Docker doesn't know which registry to send it to.

Think of it like adding a full postal address before mailing a package.

Before tag: `log-generator:5`
After tag: `nexus3.systems.uk.hsbc:<port>/com/hsbc/.../log-generator:latest`

---

**`docker push ${NEXUS3_URL}/${NEXUS3_APP_PATH}:latest`**

**What it does:** Uploads the tagged image to the Nexus3 registry.

**Why push to Nexus?**
The Kubernetes cluster needs to pull the image from somewhere. Since the cluster can only access the internal Nexus3 registry (not the Jenkins agent's local Docker daemon), the image must be pushed to Nexus3 first so Kubernetes can pull it during deployment.

---

### Block 8 — Stage: OIDC Login
```groovy
stage('OIDC Login') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'Service_Account',
            passwordVariable: '<Confidential>',
            usernameVariable: '<Confidential>'
        )]) {
            sh """
            set +x;
            oidc_temp=$(curl -sku ${env.USER}:${env.PASSWORD} -X GET ${idpapi})
            && oidc_id_token=$(${jqcli} -r '.token."id_token"' <<< "$oidc_temp")
            && oidc_refresh_token=$(${jqcli} -r '.token."refresh_token"' <<< "$oidc_temp")
            && base64 -d <<< ${apicert} > ./tmp_cert
            && ${kubectlcli} config --kubeconfig=./kubeconfigfile set-cluster kubernetes --server=${apiserver} --certificate-authority=./tmp_cert --embed-certs=true
            && ${kubectlcli} config --kubeconfig=./kubeconfigfile set-context kubernetes --cluster=kubernetes --user=${env.USER}
            && ${kubectlcli} config --kubeconfig=./kubeconfigfile set-credentials ${env.USER} --auth-provider=oidc ...
            && ${kubectlcli} config --kubeconfig=./kubeconfigfile use-context kubernetes
            """
        }
        echo "IKP OIDC login successful."
    }
}
```

**What is OIDC?**
OIDC stands for **OpenID Connect** — it is an authentication protocol built on top of OAuth 2.0. In HSBC's Kubernetes environment (IKP — Internal Kubernetes Platform), instead of using a static username/password to access the cluster, users authenticate via OIDC tokens. This is more secure and supports token expiry and rotation.

---

**`set +x`**
Disables command echoing in the shell. Without this, every command (including those containing credentials) would be printed in the Jenkins log. `set +x` **prevents credentials from being leaked in logs**. This is a **security best practice**.

---

**`curl -sku ${env.USER}:${env.PASSWORD} -X GET ${idpapi}`**
Calls the **[CONFIDENTIAL]** Identity Provider API to retrieve an OIDC token.
- `-s` — Silent mode (no progress output)
- `-k` — Skip SSL certificate verification (used for internal servers with self-signed certs)
- `-u user:password` — Basic authentication
- `-X GET` — HTTP GET request

The response contains a JSON object with the OIDC tokens.

---

**`jq -r '.token."id_token"'`**
**[CONFIDENTIAL]** `jq` is a command-line JSON parser. This extracts the `id_token` value from the JSON response. The `id_token` is a JWT (JSON Web Token) that proves your identity to Kubernetes.

---

**`base64 -d <<< ${apicert} > ./tmp_cert`**
Decodes the **[CONFIDENTIAL]** base64-encoded Kubernetes API server certificate and saves it to a temp file. This certificate is needed to establish a trusted TLS connection to the API server.

---

**`kubectl config set-cluster / set-context / set-credentials / use-context`**
These four commands build a `kubeconfig` file — the configuration file `kubectl` uses to communicate with a Kubernetes cluster:

| Command | What it does |
|---------|-------------|
| `set-cluster` | Registers the **[CONFIDENTIAL]** Kubernetes API server URL and TLS certificate |
| `set-context` | Links a cluster + user into a named "context" |
| `set-credentials` | Stores the OIDC token and Identity Provider details for authentication |
| `use-context` | Activates the context so subsequent `kubectl` commands use it |

**Why OIDC instead of a simple service account token?**
OIDC is the **enterprise-grade authentication method** for HSBC's Kubernetes platform (IKP). It integrates with the corporate identity system, supports token expiry for security, and provides audit trails. Simple service account tokens are less secure and harder to rotate. This is a **team/organisation decision**.

---

### Block 9 — Commented Stages (Checking existing deployments & Deploy to Kubernetes)
```groovy
// stage ("Checking existing deployments") {
//     steps {
//         sh chmod 777 PdpMessageApiMWS/manifest_file_payments/cleanscript_nonprod.sh
//         ./PdpMessageApiMWS/manifest_file_payments/cleanscript_nonprod.sh ${ENVIRONMENT}
//     }
// }

// stage ("Deploy to Kubernetes") {
//     steps {
//         sh """set +x; ${kubectlcli} --kubeconfig=./kubeconfigfile apply -f log-generator/${MANIFEST_PATH}"""
//         echo "Kube operation successful: apply"
//         sh """sleep 30s"""
//     }
// }
```

**Why are these stages commented out?**
These stages exist in the file but are disabled with `//` comments. This is common during development — the stages are written and ready, but not yet activated for the current sprint/demo.

---

**"Checking existing deployments" stage:**
This stage runs a `cleanscript_nonprod.sh` script that cleans up any existing deployments before applying a new one. It is commented out here, but in a real pipeline it would prevent conflicts from old deployments.

**`chmod 777`** — Gives full read/write/execute permissions to the clean script before running it.

> You mentioned: *"I believe this part is irrelevant here but please explain"*
> It is relevant in a full pipeline — it ensures you don't have conflicting old deployments running when you deploy a new version. For this demo/learning project, it's commented out because we are starting fresh each time.

---

**"Deploy to Kubernetes" stage:**
```groovy
${kubectlcli} --kubeconfig=./kubeconfigfile apply -f log-generator/${MANIFEST_PATH}
```
`kubectl apply -f` reads the `deployment.yaml` file and **creates or updates** the Kubernetes deployment. The `--kubeconfig` flag points to the kubeconfig file built in the OIDC Login stage.

`sleep 30s` — Waits 30 seconds after deployment for the Pod to start up before the next stage runs.

---

### Block 10 — Stage: Undeploy from Kubernetes
```groovy
stage('Undeploy from Kubernetes') {
    steps {
        sh """set +x; ${kubectlcli} --kubeconfig=./kubeconfigfile delete -f log-generator/${MANIFEST_PATH} --ignore-not-found=true"""
        echo "Kube operation successful: delete"
    }
}
```

**What it does:** Deletes the Kubernetes deployment after the log has been generated and captured.

**`kubectl delete -f`** — Reads the same `deployment.yaml` and deletes all resources defined in it.

**`--ignore-not-found=true`**
If the deployment doesn't exist (perhaps it was never created due to an earlier failure), this flag prevents the command from failing. Without it, `kubectl delete` returns an error if the resource is not found, which would fail the pipeline even though there's nothing wrong.

**Why undeploy after running?**
This is a log-generator — a one-time task. Once the log is captured, the Pod has no reason to continue running. Leaving it running would waste cluster resources. In enterprise Kubernetes clusters, resources are shared and limited — clean up after yourself.

---

### Block 11 — Commented Archive Stage
```groovy
// stage('Archive log') {
//     steps {
//         archiveArtifacts artifacts: 'application.log', allowEmptyArchive: false
//     }
// }
```

**What it does (when enabled):**
`archiveArtifacts` saves the `application.log` file as a **Jenkins build artifact** — it becomes downloadable from the Jenkins build page and is retained for historical review.

**`allowEmptyArchive: false`** — Fails the build if `application.log` doesn't exist or is empty. This is a quality gate — if no logs were generated, something went wrong and the build should not be marked as successful.

**Why commented out?**
Likely commented out during development while the log capture approach was being finalised. In the current pipeline, `docker logs > application.log` captures the log in the Docker Run stage — the archive stage would be the next step to enable.

---

### Block 12 — Post Actions
```groovy
post {
    success {
        echo "Pipeline succeeded"
    }
    failure {
        echo "Pipeline failed"
    }
    always {
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
    }
}
```

**What it does:**
The `post` block defines actions that run **after all stages complete**, regardless of whether the pipeline passed or failed.

---

**`success { echo "Pipeline succeeded" }`**
Runs only if all stages passed. In a real pipeline, this would typically send a success notification to Slack or email.

---

**`failure { echo "Pipeline failed" }`**
Runs only if any stage failed. In a real pipeline, this would alert the team about the failure.

---

**`always { sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true" }`**

**What it does:** Always runs — deletes the locally built Docker image from the Jenkins agent after the pipeline finishes.

**Why clean up the image?**
Jenkins agents have limited disk space. If every build leaves a Docker image behind, the agent disk fills up quickly and breaks future builds. This cleanup ensures the agent stays healthy.

**`|| true`**
If `docker rmi` fails (e.g., the image was never built because an early stage failed), the `|| true` prevents this cleanup command from failing the pipeline. It says: *"Try to delete, but if it fails, that's okay — continue anyway."*

This is a **defensive programming pattern** — handle the failure gracefully.

---

## 🔐 Confidential Summary for this Jenkinsfile

| Variable / Value | Confidentiality |
|-----------------|----------------|
| `NEXUS_REPO` | **CONFIDENTIAL** — Internal Nexus3 registry URL |
| `NEXUS3_URL` | **CONFIDENTIAL** — Internal Nexus3 URL with port |
| `NEXUS3_APP_PATH` | **CONFIDENTIAL** — Internal image path in Nexus3 |
| `idpapi` | **CONFIDENTIAL** — Internal OIDC Identity Provider API URL |
| `jqcli` | **CONFIDENTIAL** — Internal tool path on Jenkins agent |
| `kubectlcli` | **CONFIDENTIAL** — Internal kubectl binary path |
| `apicert` | **CONFIDENTIAL** — Base64-encoded Kubernetes API TLS certificate |
| `idpcert` | **CONFIDENTIAL** — Base64-encoded Identity Provider TLS certificate |
| `apiserver` | **CONFIDENTIAL** — Kubernetes API server URL |
| `idpissuer` | **CONFIDENTIAL** — OIDC issuer URL |
| `credentialsId` values | **CONFIDENTIAL** — Jenkins credential store IDs |
| `usernameVariable` / `passwordVariable` | **CONFIDENTIAL** — Variable names holding credentials at runtime |

---

## 🗺️ Full Pipeline Flow Summary

```
[Trigger: Git push or manual]
        ↓
1. Checkout      — Clone source code from internal GitHub
        ↓
2. Docker Build  — Login to Nexus3, build image, tag with build number
        ↓
3. Docker Run    — Run container, generate logs, save to application.log
        ↓
4. Push to Nexus — Tag image with full Nexus path, push to registry
        ↓
5. OIDC Login    — Authenticate to IKP Kubernetes cluster via OIDC tokens
        ↓
6. Deploy (commented) — Apply deployment.yaml to Kubernetes dev namespace
        ↓
7. Undeploy      — Delete Kubernetes deployment after log capture
        ↓
8. Post Always   — Delete local Docker image, free up disk space
```

---

## ❓ Common Peer Questions & Answers

| Question | Answer |
|----------|--------|
| Why store credentials in Jenkins and not in the Jenkinsfile? | Hardcoding credentials in a file that lives in Git is a critical security risk — anyone with repo access can see them. Jenkins Credentials Store is encrypted, access-controlled, and the values are masked in logs. |
| Why do we need OIDC? Why not just use kubectl with a service account token? | HSBC's internal Kubernetes platform (IKP) uses OIDC as its authentication standard. It integrates with the corporate identity system, supports token expiry for security, and provides audit trails. Service account tokens are simpler but less secure. This is an organisation-level decision. |
| Why is the Deploy stage commented out? | It appears to be in progress — the OIDC login and Undeploy stages are active, but the Deploy stage is commented out, possibly because the Kubernetes access is still being set up or this is a partial implementation for the demo. |
| What is `set +x` and why is it important? | It disables shell command echoing. Without it, every shell command (including those containing credentials and tokens) would be printed to the Jenkins build log — a serious security leak. Always use `set +x` before any command that handles sensitive data. |
| Why `sleep` instead of waiting for container exit? | `sleep` is simple and predictable. A better approach would be `docker wait log-generator` which waits until the container exits naturally. The `sleep` is used here for simplicity in a learning/demo context. |
| What happens if the Push stage fails? | The Undeploy stage would still run (stages execute sequentially) but since the image was never pushed, the Deploy stage (if enabled) would fail because Kubernetes can't pull the image. The `post { failure }` block would then run and alert the team. |
| What is `jq`? | `jq` is a lightweight command-line JSON processor for Linux. It's used here to extract specific fields (`id_token`, `refresh_token`) from the JSON response returned by the OIDC API. Think of it as `grep` but for JSON. |





4. OIDC Process and Environment Variables — Explained Simply
"Let me explain OIDC from scratch — no jargon.
The problem first:
We need Jenkins to talk to HSBC's Kubernetes cluster — IKP. But IKP is a secure internal system. You can't just connect to it with a username and password. It uses a proper enterprise authentication system called OIDC.
What is OIDC?
OIDC stands for OpenID Connect. Think of it like this — you know how some websites let you log in with your Google account instead of creating a new account? That's the same concept. You prove your identity to a trusted third party — called an Identity Provider — and they give you a token. You then use that token to access the system you want. The system trusts the token because it trusts the Identity Provider.
In our case:

The Identity Provider is HSBC's internal corporate identity system
The token is a JWT — JSON Web Token — a string that proves who you are
The system we want to access is the IKP Kubernetes cluster

The OIDC flow step by step:
Step 1: Jenkins calls the Identity Provider API with our service account credentials
        → idpapi is the URL of that Identity Provider API

Step 2: The Identity Provider verifies our credentials and returns a JSON response
        containing two tokens — an id_token and a refresh_token
        → jqcli (jq tool) extracts these tokens from the JSON response

Step 3: We decode the API server certificate
        → apicert is the base64-encoded TLS certificate of the Kubernetes API server
        → We decode it so kubectl can trust the connection

Step 4: We build a kubeconfig file using kubectl config commands
        → This file tells kubectl: here is the cluster, here is the user, here is the token
        → apiserver is the URL of the Kubernetes API server
        → idpissuer is the URL that Kubernetes uses to verify the token is genuine
        → idpcert is the Identity Provider's certificate so Kubernetes can trust it

Step 5: kubectl use-context activates this config
        → Now all kubectl commands go to the right cluster with the right credentials
Now each variable explained:
VariableWhat it isSimple explanationidpapiIdentity Provider API URLThe address we call to get our OIDC tokenjqcliPath to jq tooljq is a JSON parser — we use it to pull the token out of the API responsekubectlcliPath to kubectl binaryThe Kubernetes command line tool — we use a specific internal versionapicertKubernetes API certificateBase64 encoded TLS cert — proves we're talking to the real Kubernetes serveridpcertIdentity Provider certificateKubernetes uses this to verify our token is from a trusted sourceapiserverKubernetes API server URLThe address of the IKP Kubernetes clusteridpissuerOIDC issuer URLKubernetes checks this to confirm the token was issued by our trusted Identity Provider
Why OIDC and not a simple token?
A simple static token never expires — if someone steals it, they have permanent access. OIDC tokens expire — usually in a few hours. Even if stolen, they become useless quickly. Plus every login is logged and audited. For a bank like HSBC, that level of security is mandatory.
Why set +x before all of this?
Because without it, every command gets printed in the Jenkins log — including the ones containing our tokens and credentials. set +x turns off that printing. It's a security measure."