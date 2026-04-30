# 🎤 My KT Script — Log Generator Project
### 30 Minutes | Screen Share — GitHub Repo

---

> 📌 HOW TO USE THIS SCRIPT
> - Regular text = what you SAY
> - [ACTION: ...] = what you DO on screen
> - Pause naturally at every ---
> - Don't rush — speak like you're explaining to a friend

---

## OPENING
### (2 minutes)

---

[ACTION: Have your GitHub repo open. Show the repo home page.]

---

"Good morning everyone.

So, this week my focus was on Docker and Kubernetes — and as part of that, I built this small project called the Log Generator.

I'll walk you through what I built, how it works, and why I did things the way I did.

Before I open any file, let me just quickly tell you what this project actually does — because once you understand that, the code will make much more sense.

So basically — the idea is simple. We have a shell script that generates some log lines — like what a real application would log when it runs. Things like, application started, connected to database, processed a transaction, got an error, retried, shut down.

Now the script itself is not the main point. The main point is — how do we take that script, package it properly, automate it, and deploy it on Kubernetes. That's the DevOps side of it.

So the flow is like this —

We write the script. We package it into a Docker image using a Dockerfile. Jenkins picks it up, builds the image, runs it, pushes it to our internal Nexus registry, and deploys it on Kubernetes.

That's the full picture. Now let me open each file and explain what's happening."

---

## BIG PICTURE FLOW
### (2 minutes)

---

[ACTION: Stay on the repo home page. Just talk through the folder structure.]

---

"So if you look at the repo, we have four main things —

generate_log.sh — this is the script, our actual application.

Dockerfile — this is how we package the script into a Docker image.

k8s/deployment.yaml — this tells Kubernetes how to run our container.

And Jenkinsfile — this is the pipeline that automates everything end to end.

Let me start with the script."

---

## FILE 1 — generate_log.sh
### (5 minutes)

---

[ACTION: Click and open generate_log.sh]

---

"So this is generate_log.sh — our shell script.

---

[ACTION: Point to line 1]

The first line is #!/bin/bash — this is called a shebang line. It just tells the system — hey, use Bash to run this script. Every shell script needs this as the first line, otherwise the system doesn't know how to execute it.

---

[ACTION: Point to the TIMESTAMP line]

Then we have TIMESTAMP=$(date) — this captures the current date and time and stores it in a variable. We do this once at the top and reuse it in every log line. The reason is — all these log lines belong to one single run, so they should all show the same time. If we called date separately for each line, the times would be slightly different, which would be confusing.

---

[ACTION: Point to the cat EOF block]

Then we have this cat << EOF block — this is called a heredoc. It's basically a shortcut to print multiple lines at once without writing echo for every single line. Everything between EOF and the closing EOF gets printed as output. It just keeps the code clean and readable.

---

[ACTION: Point to the Build Number line inside the heredoc]

Inside the heredoc, you'll notice $BUILD_NUMBER. Now — I didn't define this anywhere in the script. This is actually a Jenkins built-in variable. When Jenkins runs this script inside a pipeline, it automatically injects the build number into the environment. So every time this script runs, it prints which Jenkins build triggered it. This is useful for traceability — if something goes wrong, you can match the log back to the exact build.

---

[ACTION: Slowly scroll through the log lines]

The log lines themselves — INFO, WARNING, ERROR — these are just simulating what a real application would log during its lifecycle.

We have INFO for normal things like startup and database connection.

We have WARNING for something like high memory usage — not broken yet, but needs attention.

We have ERROR for a transaction timeout — in banking, this is serious and needs to be investigated.

Then the application retries — which is a standard pattern, you don't just give up on the first failure.

And finally it shuts down gracefully — meaning it finished its work properly before stopping.

So even though this is a simulated script, it's designed to look like real application logs."

---

## FILE 2 — Dockerfile
### (5 minutes)

---

[ACTION: Navigate to and open the Dockerfile]

---

"Okay so now we have our script. But the problem is — if I just run this script directly, it might work on my machine but behave differently on someone else's machine, or on the Jenkins server, or on Kubernetes. Different OS, different Bash version, different environment variables — things can break.

Docker solves this. We package the script into a Docker image, and that image runs exactly the same way everywhere. That's the whole point.

This Dockerfile is basically the instructions for building that image.

---

[ACTION: Point to the FROM line]

The first line is FROM — and this is where we pick our base image. Think of it like choosing which operating system to start with inside the container. We're using Ubuntu 20.04.

Now — one important thing here. We are not pulling this from Docker Hub. We're pulling it from our internal Nexus3 registry. This is a company policy at HSBC — we can't use public registries on company infrastructure. All base images have to come from Nexus3 because those images are security-scanned and approved by the organisation. So whatever image we need, it has to be available there first.

We picked Ubuntu 20.04 specifically because it's an LTS version — Long Term Support — which means it gets security updates for 5 years. It's stable, it's widely used, and it already has Bash installed which is all we need.

---

[ACTION: Point to WORKDIR]

Next is WORKDIR /app. This sets the working directory inside the container. So instead of everything landing in the root folder of the container, we keep our application files neatly in /app. It's just good practice — you don't want to mix application files with the OS files in the root directory.

---

[ACTION: Point to COPY]

Then COPY generate_log.sh . — this copies our script from the local machine into the container's /app folder. Docker containers are completely isolated — they can't see the host machine's files. So we have to explicitly copy the script in. The dot at the end just means copy it to the current working directory, which is /app.

---

[ACTION: Point to RUN chmod]

Then RUN chmod +x generate_log.sh — this gives the script execute permission. When you copy a file into a container, it might not automatically be executable. chmod +x adds that permission. Without this, when the container tries to run the script, it would get a permission denied error.

---

[ACTION: Point to CMD]

And the last line is CMD — this is the command that runs when the container starts. We're telling Docker — when this container starts, run generate_log.sh.

We're using the exec form here — the array format with square brackets. The reason is that with this format, the script runs directly as the main process, not wrapped inside a shell. This means it handles signals properly — like if Kubernetes needs to stop the container, the script receives that signal directly and can shut down cleanly."

---

## FILE 3 — k8s/deployment.yaml
### (5 minutes)

---

[ACTION: Navigate to k8s/deployment.yaml and open it]

---

"Now — we have our Docker image built and pushed to Nexus3. Next step is running it on Kubernetes.

This deployment.yaml file is how we tell Kubernetes what to run and how to run it. It's called a manifest file — you describe what you want, and Kubernetes makes it happen.

---

[ACTION: Point to apiVersion and kind]

The first two lines are apiVersion and kind.

apiVersion: apps/v1 — this tells Kubernetes which API version to use for this resource. apps/v1 is the stable, production-ready version for managing application workloads.

kind: Deployment — this tells Kubernetes what type of resource we're creating. A Deployment is the standard way to run an application on Kubernetes. The reason we use a Deployment and not just a raw Pod is — if a Pod crashes, it stays dead. A Deployment has a controller watching over it, so if the Pod dies, Kubernetes automatically creates a new one.

---

[ACTION: Point to metadata section]

Then the metadata section. We give the deployment a name — log-generator-deployment. And we specify the namespace — dev.

Namespaces are like separate sections within a Kubernetes cluster. We use namespaces to separate environments — dev, test, prod — all on the same cluster without interfering with each other. We're deploying to dev because this is a development and testing setup.

The labels — app: log-generator — are just tags we attach to the resource. Kubernetes uses these labels to identify and connect related resources together.

---

[ACTION: Point to replicas]

replicas: 1 — this tells Kubernetes to run exactly one instance of our container. For a simple log generator script that runs and exits, one replica is enough. If this was a real API handling lots of traffic, we'd set this to 3 or more for high availability.

---

[ACTION: Point to selector and matchLabels]

The selector section with matchLabels — this is how the Deployment knows which Pods are its responsibility. The label here must exactly match the label in the Pod template below. Kubernetes uses this to link the Deployment to its Pods.

---

[ACTION: Point to imagePullSecrets]

imagePullSecrets — name: nexus3-cred. This is important. Our Docker image is sitting in the internal Nexus3 registry which requires authentication. Kubernetes can't just pull the image without credentials. So we store those credentials as a Kubernetes Secret called nexus3-cred, and we reference it here. Kubernetes uses this secret to authenticate with Nexus3 before pulling the image.

We never put credentials directly in this YAML file — that would be a security risk because this file lives in Git and anyone with repo access could see it. Secrets keep credentials safe.

---

[ACTION: Point to image and imagePullPolicy]

image — this is the full Nexus3 URL of our Docker image. Kubernetes pulls this image when creating the Pod.

imagePullPolicy: Always — this tells Kubernetes to always pull the latest image from the registry, even if it already has a cached copy. We use this because during development we keep pushing new versions tagged as latest. If Kubernetes used a cached old version, it might run outdated code.

---

[ACTION: Point to resources section]

And finally — the resources section. This is one of the most important parts.

We have two things here — requests and limits.

requests is the minimum resources Kubernetes guarantees for this container before it schedules it on a node. We're asking for 128Mi of memory and 250 millicores of CPU — which is quarter of one CPU core.

limits is the maximum the container is allowed to use. 256Mi memory and 500 millicores CPU.

Why do we need this? Because in a shared Kubernetes cluster, multiple teams and applications are running on the same nodes. Without limits, one runaway container could eat up all the resources and starve everyone else. Requests and limits are how we make sure everyone gets a fair share.

If the container exceeds the memory limit, Kubernetes will kill it immediately. If it exceeds the CPU limit, it gets throttled — slowed down but not killed."

---

## FILE 4 — Jenkinsfile
### (10 minutes)

---

[ACTION: Navigate to and open the Jenkinsfile]

---

"Okay — this is the Jenkinsfile. This is the brain of the whole project.

Everything we just talked about — building the Docker image, running the container, pushing to Nexus3, deploying to Kubernetes — all of that is automated here. We don't do any of it manually. Jenkins does it all when a build is triggered.

This is called a Declarative Pipeline — it's the modern way to write Jenkins pipelines. You define everything as code, it lives in your repo, and every team member gets the same pipeline.

---

[ACTION: Point to agent block]

The first thing is the agent block. agent label: cm-linux-gce. This tells Jenkins which machine to run the pipeline on. cm-linux-gce is the label for our Linux build agents running on Google Compute Engine within HSBC's infrastructure. We specify this label so Jenkins always picks a Linux agent — because our pipeline uses Docker and shell commands which need Linux.

---

[ACTION: Point to environment block]

Then the environment block. This is where we define all the variables that are used across the pipeline.

IMAGE_NAME is log-generator — the name of our Docker image.

IMAGE_TAG is set to $BUILD_NUMBER — Jenkins automatically gives every build a unique number. By using it as the image tag, every build produces a uniquely tagged image. So if something breaks, we can trace it back to the exact build that caused it.

NEXUS_REPO, NEXUS3_URL, NEXUS3_APP_PATH — these are our internal Nexus3 registry details. I won't read them out because they're internal, but they point to where our image gets stored.

MANIFEST_PATH points to our deployment.yaml file — k8s/deployment.yaml. We define it here so if the file ever moves, we only update it in one place.

The rest of the variables — idpapi, kubectlcli, apiserver, apicert and so on — these are all for the OIDC login stage which I'll explain shortly.

---

[ACTION: Point to Checkout stage]

Stage 1 — Checkout.

This is straightforward. checkout scm just pulls the latest code from our GitHub repo onto the Jenkins agent. scm stands for Source Control Management — Jenkins already knows the repo URL from the job configuration, so we don't have to hardcode it here. This keeps the Jenkinsfile reusable across any branch.

---

[ACTION: Point to Docker Build stage]

Stage 2 — Docker Build Image.

First we have withCredentials. This is how we securely handle passwords in Jenkins. Instead of writing the Nexus3 username and password directly in the file — which would be visible in Git — we store them in the Jenkins Credentials Store and fetch them at runtime. Jenkins also masks these values in the build logs so they never appear as plain text.

Then docker login — we authenticate with Nexus3 so we're allowed to pull the base image and later push our image.

Then docker build — this builds our Docker image using the Dockerfile. We tag it with the image name and build number. The log-generator/ at the end is the build context — the folder containing our Dockerfile and script.

---

[ACTION: Point to Docker Run stage]

Stage 3 — Docker Run.

docker run — this starts a container from the image we just built. We pass -e BUILD_NUMBER so the container gets the Jenkins build number as an environment variable. Remember in our script, we use $BUILD_NUMBER — this is how it gets there.

docker logs log-generator — this prints the container output to the Jenkins console so we can see it in the build log.

docker logs log-generator > application.log — same output but saved to a file. This file is what we'd archive as a build artifact.

sleep 30 — we wait 30 seconds to give the container enough time to finish running before we move to the next stage.

---

[ACTION: Point to Push stage]

Stage 4 — Push Docker Image to Nexus.

Again we use withCredentials for secure login.

docker tag — we add the full Nexus3 registry path to the image name. Docker needs this full path to know which registry to push to. Think of it like adding a full address to a package before posting it.

docker push — this uploads the image to our internal Nexus3 registry. The reason we push it is — the Kubernetes cluster needs to pull this image when deploying. The cluster can only access Nexus3, not the Jenkins agent's local Docker. So the image must be in Nexus3 first.

---

[ACTION: Point to OIDC Login stage]

Stage 5 — OIDC Login.

This stage is about authenticating to HSBC's internal Kubernetes platform — which we call IKP.

Now — to connect to IKP, we don't use a simple username and password. We use OIDC — OpenID Connect. It's an enterprise authentication standard that integrates with the corporate identity system. It's more secure because tokens expire, they can be revoked, and every login is audited. This is an organisation-level decision — the IKP platform requires OIDC.

The first thing you'll notice is set +x. This disables command echoing in the shell. Without this, every command would be printed in the Jenkins log — including the ones with credentials and tokens. set +x is a security measure to prevent that.

Then we call the Identity Provider API using curl to get an OIDC token. We use jq — a JSON parsing tool — to extract the id_token and refresh_token from the response.

Then we use kubectl config commands to build a kubeconfig file — that's the configuration file kubectl needs to talk to the Kubernetes cluster. We set the cluster details, user credentials, and OIDC token. And finally use-context activates it so all subsequent kubectl commands use this configuration.

---

[ACTION: Point to the commented out stages]

Now — you'll notice two stages here that are commented out. The Checking Existing Deployments stage and the Deploy to Kubernetes stage.

The Deploy stage is what would actually apply our deployment.yaml to the cluster using kubectl apply. It's written and ready — but currently commented out. The reason is that during this sprint, we wanted to first get the full pipeline working — image build, push, OIDC authentication — and validate each piece before enabling the actual deployment. The plan is to uncomment and activate this in the next iteration.

The Checking Existing Deployments stage runs a cleanup script before deploying — to make sure there are no conflicting old deployments running. That would be enabled alongside the Deploy stage.

---

[ACTION: Point to Undeploy stage]

Stage 6 — Undeploy from Kubernetes.

kubectl delete -f reads our deployment.yaml and deletes everything defined in it from the cluster. This runs after the log is generated and captured — because the log generator is a one-time task. Once it's done, there's no reason to keep the Pod running and wasting cluster resources.

--ignore-not-found=true is a safety flag. If the deployment doesn't exist for some reason — maybe an earlier stage failed — this flag prevents kubectl delete from throwing an error. Without it, the pipeline would fail on a cleanup step even though nothing is actually wrong.

---

[ACTION: Point to the commented Archive stage]

You'll also see the Archive log stage is commented out. archiveArtifacts would save application.log as a downloadable artifact on the Jenkins build page. This is ready to be enabled — it's just commented out during this phase of development.

---

[ACTION: Point to post block]

And finally — the post block. This runs after all stages complete, no matter what happened.

On success — we print a success message. In a real pipeline this would send a notification to Slack or email.

On failure — we print a failure message. Same idea — this would alert the team.

Always — regardless of success or failure, we run docker rmi to delete the locally built image from the Jenkins agent. This is important for disk space management. If every build leaves an image behind, the agent disk fills up and breaks future builds.

The || true at the end is a safety net. If the image was never built — because an early stage failed — docker rmi would normally throw an error and fail the pipeline. || true says — if this command fails, that's okay, just move on."

---

## CLOSING
### (1 minute)

---

[ACTION: Go back to the repo home page]

---

"So that's the full project.

To quickly summarise —

We have a shell script that simulates application logs.

We packaged it into a Docker image using a Dockerfile, with the base image coming from our internal Nexus3 registry.

Jenkins automates the full lifecycle — build, run, push to Nexus3, authenticate to IKP via OIDC, and deploy and undeploy on Kubernetes.

The two things still to be activated in the next iteration are the Deploy stage and the Archive log stage — both are written and ready, just commented out.

I'm happy to take any questions."

---

## HANDLING TOUGH QUESTIONS

---

> Read these answers naturally — don't rush them.

---

**Q: Why Docker and not just run the script directly?**

"Running the script directly works on one machine, but it might behave differently on another machine — different OS, different Bash version, different environment. Docker packages everything together so it runs exactly the same way everywhere. That's the core value of containerisation."

---

**Q: Why internal Nexus3 and not Docker Hub?**

"HSBC's security policy doesn't allow pulling images from public registries on company infrastructure. All images must come from the internal Nexus3 registry because those images are security-scanned and approved by the organisation. This applies to both base images and the images we push."

---

**Q: Why OIDC for Kubernetes and not a simple service account token?**

"The IKP platform — HSBC's internal Kubernetes — uses OIDC as its authentication standard. It integrates with the corporate identity system, tokens expire automatically which is more secure, and every login is audited. This is an organisation-level decision, not something we chose ourselves."

---

**Q: Why is the Deploy stage commented out?**

"During this sprint the focus was on getting the full pipeline working piece by piece — image build, push, OIDC authentication — and validating each stage before enabling actual deployment. The Deploy stage is written and ready. It will be activated in the next iteration once the full flow is validated end to end."

---

**Q: What is set +x and why is it used?**

"set +x disables command echoing in the shell. By default, every command that runs gets printed in the Jenkins build log. In the OIDC stage we're dealing with tokens and credentials — if those got printed in the log, that's a security risk. set +x prevents that. It's a standard security practice whenever you handle sensitive data in shell scripts."

---

**Q: What happens if the memory limit is exceeded in Kubernetes?**

"Kubernetes immediately kills the container — it's called OOMKilled, Out Of Memory Killed. Then Kubernetes restarts it automatically because we have a Deployment managing it. The limits are there to protect other workloads on the same node from being starved of resources."

---

**Q: Why replicas: 1 and not more?**

"The log generator is a one-time script — it runs, generates a log, and exits. There's no need for multiple instances running at the same time. If this was a real API serving user requests, we'd set replicas to 3 or more for high availability and load distribution."

---

**Q: Something you don't know the answer to**

"That's a good question — I haven't looked into that specific area yet. Let me check and get back to you after the session."

---

> 🎯 Final tip — Speak slowly. Pause after each section. If someone asks a question mid-way, answer it and say "I'll also cover this when we get to that file" if needed. You've got this! 💪

---
