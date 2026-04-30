# 📄 File 3: `k8s/deployment.yaml` — Line by Line Explanation

---

## 🔎 What is this file?

`deployment.yaml` is a **Kubernetes manifest file**. It is written in YAML format and tells Kubernetes exactly how to deploy and manage our application (the log-generator container) on a Kubernetes cluster.

Think of it as an **instruction sheet** you hand to Kubernetes saying:
- *"Here's the application I want to run"*
- *"Run exactly 1 copy of it"*
- *"Use this Docker image"*
- *"Give it these resources"*
- *"Pull the image using these credentials"*

Kubernetes reads this file and makes it happen — creating Pods, managing restarts, and ensuring the defined state is maintained.

> ⚠️ **Note:** The file you shared appears to have the Dockerfile content pasted at the top by mistake (likely a Google Lens OCR issue). The actual `deployment.yaml` content starts from `apiVersion:` onwards. The explanation below covers only the YAML content.

---

## 📋 Full Code with Line-by-Line Explanation

---

### Block 1 — API Version & Kind
```yaml
apiVersion: apps/v1
kind: Deployment
```

**`apiVersion: apps/v1`**

**What it does:** Tells Kubernetes which **API group and version** to use for this resource.

Kubernetes has many API groups — `apps/v1` is the stable, production-ready API for managing workloads like Deployments. `v1` means it is version 1 — the stable, non-experimental version.

**Why `apps/v1` and not just `v1`?**
Core resources like Pods and Services use just `v1`. But Deployments are part of the `apps` extension group, so they need `apps/v1`. Using the wrong API version would cause Kubernetes to reject the manifest.

---

**`kind: Deployment`**

**What it does:** Tells Kubernetes what **type of resource** you are creating.

A **Deployment** is a Kubernetes object that:
- Manages a set of identical Pods (containers)
- Ensures the desired number of replicas are always running
- Handles rolling updates and rollbacks automatically

**Why Deployment and not just a Pod?**
You could create a raw Pod, but if it crashes, it stays dead — nobody restarts it. A Deployment has a **ReplicaSet controller** watching over it — if a Pod dies, it automatically creates a new one. This is why Deployments are the standard way to run applications in Kubernetes.

**Alternative approach:**
| Kind | When to use |
|------|------------|
| `Pod` | Testing only — no auto-restart |
| `Deployment` | Stateless apps — standard choice ✅ |
| `StatefulSet` | Stateful apps like databases |
| `DaemonSet` | Run one copy on every node |
| `Job` | One-time task that runs to completion |

For our log-generator (a stateless script), `Deployment` is the right choice.

---

### Block 2 — Metadata
```yaml
metadata:
  name: log-generator-deployment
  namespace: dev
  labels:
    app: log-generator
```

**`metadata:`**
The metadata block gives identity information about this Kubernetes resource.

---

**`name: log-generator-deployment`**

**What it does:** The unique name of this Deployment within the namespace. Kubernetes uses this name to identify, update, or delete the deployment.

**Why this name?**
It's descriptive — `log-generator` tells you what the app is, and `-deployment` tells you what kind of Kubernetes object it is. This makes it easy to identify in `kubectl get deployments`.

---

**`namespace: dev`**

**What it does:** Places this deployment inside the `dev` namespace.

Kubernetes **namespaces** are like virtual clusters within a physical cluster. They are used to:
- Separate environments (dev, test, prod) on the same cluster
- Apply different access controls per namespace
- Avoid naming conflicts between teams

**Why `dev`?**
This is a development/testing deployment — it belongs in the `dev` namespace, not `prod`. This is a team convention — **taken based on team suggestion** for environment separation.

**Peer Question you may face:**
> *"What happens if you don't specify a namespace?"*

Answer: Kubernetes places the resource in the `default` namespace. In enterprise environments, using `default` is discouraged — it has no isolation and everyone can see everything in it.

---

**`labels: app: log-generator`**

**What it does:** Attaches a **label** (key-value pair) to this Deployment.

Labels are metadata tags used for:
- **Identifying** resources (`kubectl get pods -l app=log-generator`)
- **Linking** resources — the Service uses labels to find Pods to route traffic to
- **Organizing** resources in dashboards

**Why this specific label?**
`app: log-generator` is the **most common Kubernetes labelling convention**. The `app` key identifies the application name. This same label is used in `selector` and `template` below to connect the Deployment to its Pods.

---

### Block 3 — Spec (Deployment Specification)
```yaml
spec:
  replicas: 1
```

**`spec:`**
The `spec` block defines **what the Deployment should do** — its desired state.

---

**`replicas: 1`**

**What it does:** Tells Kubernetes to run **exactly 1 Pod** (container instance) of this application.

**Why only 1 replica?**
This is a log-generator script — it runs, generates a log, and exits. There's no need for multiple instances running simultaneously. For a real banking API handling thousands of requests, you'd set `replicas: 3` or more for high availability.

**Alternative approach:**
For a production API you might use:
```yaml
replicas: 3
```
This runs 3 identical Pods. If one crashes, the other 2 keep serving traffic while Kubernetes replaces the crashed one.

---

### Block 4 — Selector
```yaml
  selector:
    # Identifies which Pods this Deployment manages
    matchLabels:
      app: log-generator
```

**What it does:**
The `selector` tells the Deployment **which Pods it is responsible for managing**.

Kubernetes uses this to link the Deployment → ReplicaSet → Pods. It says: *"Manage any Pod that has the label `app: log-generator`."*

**Why must it match the template labels?**
The `selector.matchLabels` must exactly match the `template.metadata.labels` below. If they don't match, Kubernetes will reject the manifest with a validation error.

**Peer Question you may face:**
> *"Why do we need a selector? Can't Kubernetes just manage all Pods automatically?"*

Answer: A Kubernetes cluster can run hundreds of different applications. The selector is how the Deployment knows **which specific Pods are its responsibility**. Without it, there would be no way to link the Deployment controller to the correct Pods.

---

### Block 5 — Pod Template
```yaml
  template:
    # Pod template used when creating new Pods
    metadata:
      labels:
        app: log-generator
```

**`template:`**
The template defines the **blueprint for each Pod** that the Deployment creates. Every Pod created by this Deployment will look exactly like this template.

---

**`metadata.labels: app: log-generator`**
This must match the `selector.matchLabels` above. These labels are applied to every Pod created from this template.

---

### Block 6 — Pod Spec: imagePullSecrets
```yaml
    spec:
      imagePullSecrets:
        - name: nexus3-cred
```

**What it does:**
Tells Kubernetes to use a **stored secret** called `nexus3-cred` when pulling the Docker image from the Nexus3 registry.

**Why it is used:**
Our Docker image is stored in the **internal Nexus3 private registry** which requires authentication (username + password). Kubernetes can't pull the image without credentials. The `imagePullSecrets` field points to a Kubernetes Secret that stores those credentials securely.

**Why store credentials as a Kubernetes Secret?**
You never hardcode credentials (username/password) in YAML files — that's a serious security risk. Kubernetes Secrets store sensitive data in an encoded form and inject them at runtime. Only authorised service accounts can access them.

**How was `nexus3-cred` created?**
A cluster administrator would have created it using:
```bash
kubectl create secret docker-registry nexus3-cred \
  --docker-server=nexus3.systems.uk.hsbc:<port> \
  --docker-username=<CONFIDENTIAL> \
  --docker-password=<CONFIDENTIAL> \
  --namespace=dev
```

**Peer Question you may face:**
> *"What happens if `imagePullSecrets` is missing or wrong?"*

Answer: Kubernetes will fail to pull the image and the Pod will go into `ImagePullBackOff` status — it keeps retrying but can never start the container.

---

### Block 7 — Container Definition
```yaml
      containers:
        - name: log-generator
          image: nexus_image_url
          imagePullPolicy: Always
```

**`containers:`**
A Pod can run one or more containers. Here we define exactly one container.

---

**`name: log-generator`**
The name of this container within the Pod. Used for identification in `kubectl logs` and `kubectl exec` commands.

---

**`image: nexus_image_url`**
**[CONFIDENTIAL]** — This is a placeholder for the actual Nexus3 image URL. In the real file it would look like:
```
nexus3.systems.uk.hsbc:<port>/com/hsbc/group/log-generator:latest
```
This tells Kubernetes which Docker image to use when creating the container.

---

**`imagePullPolicy: Always`**

**What it does:** Tells Kubernetes to **always pull the latest image** from the registry every time it creates a Pod — even if the image already exists locally on the node.

**Why `Always`?**
During development and CI/CD, we frequently push new versions of the image tagged as `latest`. If Kubernetes used a cached version, it might run outdated code. `Always` ensures the freshest image is used on every deployment.

**Available options:**
| Policy | Behaviour |
|--------|-----------|
| `Always` | Always pull from registry ✅ (used here) |
| `IfNotPresent` | Pull only if not cached locally |
| `Never` | Never pull — use local cache only |

**When would you use `IfNotPresent`?**
In production with specific version tags (e.g., `:v1.2.3`). Since the tag never changes, there's no need to re-pull. It also reduces registry load.

---

### Block 8 — Resource Requests and Limits
```yaml
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

This is one of the most important blocks in the deployment spec. It tells Kubernetes how much compute resources this container needs.

---

**`requests:`**
```yaml
memory: "128Mi"
cpu: "250m"
```

**What it does:** The **minimum guaranteed resources** Kubernetes must reserve for this container before scheduling it on a node.

- `memory: "128Mi"` — Reserve 128 Mebibytes of RAM
- `cpu: "250m"` — Reserve 250 millicores of CPU (0.25 of 1 CPU core)

**Why requests are important:**
Kubernetes uses requests to decide **which node to place the Pod on**. It checks which node has at least 128Mi memory and 250m CPU free, and schedules the Pod there. Without requests, Kubernetes schedules blindly and nodes can become overloaded.

---

**`limits:`**
```yaml
memory: "256Mi"
cpu: "500m"
```

**What it does:** The **maximum resources** this container is allowed to use.

- `memory: "256Mi"` — Never use more than 256Mi RAM. If exceeded, the container is **killed (OOMKilled)**
- `cpu: "500m"` — Never use more than 0.5 CPU core. If exceeded, the container is **throttled** (slowed down, not killed)

**Why limits are important:**
Without limits, a buggy or runaway container could consume all resources on a node and starve other applications. In a shared cluster (like HSBC's), this is a serious issue. Limits act as a **safety cap**.

**Why these specific values?**
For a simple bash script that generates a text log:
- 128Mi memory is more than enough (the script uses practically nothing)
- 250m CPU is generous for a shell script

These values are likely set conservatively based on **team convention** for all non-production workloads.

**The relationship between requests and limits:**
```
requests ≤ limits (always)
128Mi    ≤ 256Mi  ✅
250m     ≤ 500m   ✅
```

**Peer Question you may face:**
> *"What happens if the container exceeds the memory limit?"*

Answer: Kubernetes immediately kills the container with an `OOMKilled` (Out Of Memory Killed) error and restarts it. This protects other Pods on the same node.

> *"What's the difference between `Mi` and `MB`?"*

Answer: `Mi` is Mebibytes (1 Mi = 1,048,576 bytes). `MB` is Megabytes (1 MB = 1,000,000 bytes). Kubernetes uses binary units (Mi, Gi) to be precise about memory allocation.

> *"What is `250m` CPU?"*

Answer: CPU in Kubernetes is measured in **millicores**. 1000m = 1 full CPU core. So 250m = 0.25 of one CPU core, and 500m = 0.5 of one CPU core.

---

## 🔐 Confidential Markings

| Item | Confidentiality |
|------|----------------|
| `nexus_image_url` (actual value) | **CONFIDENTIAL** — Internal Nexus3 image URL |
| `nexus3-cred` secret contents | **CONFIDENTIAL** — Registry credentials stored in Kubernetes Secret |
| `namespace: dev` | **INTERNAL** — Reveals environment structure |

---

## 🗺️ How this file fits in the pipeline

```
Jenkins builds Docker image → pushes to Nexus3
        ↓
Jenkins applies this deployment.yaml to Kubernetes
        ↓
Kubernetes reads the file → creates a Pod in the dev namespace
        ↓
Kubernetes pulls image from Nexus3 using nexus3-cred secret
        ↓
Container starts → generate_log.sh runs → logs generated
        ↓
Jenkins deletes the deployment (Undeploy stage)
```

---

## ❓ Common Peer Questions & Answers

| Question | Answer |
|----------|--------|
| Why YAML and not JSON? | Both work in Kubernetes. YAML is preferred because it's more human-readable, supports comments, and is less verbose than JSON. |
| Why do we delete the deployment after running? | This is a log-generator — a one-time task. Once the log is generated and captured, there's no reason to keep the Pod running. The Undeploy stage cleans up resources. |
| Could we use a Kubernetes `Job` instead of a `Deployment`? | Yes, actually! A `Job` is designed for one-off tasks that run to completion. It would be a better fit for a script that runs once and exits. A `Deployment` is for long-running services. This is likely a learning/demo choice. |
| What is a Pod exactly? | A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers and provides them with shared networking and storage. |
| What is a ReplicaSet? | A ReplicaSet is automatically created by the Deployment. It watches the Pods and ensures the specified number of replicas (`replicas: 1`) is always running. |
