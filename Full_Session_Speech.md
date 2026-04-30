# 🎤 Full Session Speech — End to End
### Virtual Session | 30 Minutes | Presenter: Chandu

---

> 📌 HOW TO USE THIS
> - Read exactly as written — it's written the way you naturally speak
> - [ACTION] = what you do on screen
> - ... = natural pause, take a breath
> - Don't rush — speak slower than you think you need to
> - Keep this open on your phone or second monitor

---

## BEFORE THE SESSION STARTS

---

[ACTION: Join the call 5 minutes early. Keep your PPT minimised and GitHub repo open in another tab. Test your screen share before everyone joins.]

When people start joining, just say casually:

"Hey, good morning... yeah just setting up my screen share, give me a second."

Once most people are on:

"Okay I think everyone's joined... let me just start sharing my screen."

[ACTION: Share your screen. Open the PPT. Go to Slide 1.]

---

## OPENING
### Slide 1 — Title Slide

---

"Alright, good morning everyone. Hope everyone's doing well.

So... my name is Chandu, I joined the Payment Data Programme team recently as part of my fresher onboarding batch.

As part of my onboarding, I've been spending each week learning one DevOps tool in depth — and today I want to share what I've been working on this week.

This session is going to be around 30 minutes... I'll cover the tools I learned, how they all connect together, and then I'll do a quick live demo to show everything actually working.

Feel free to stop me anytime if something's not clear or if you have a question. I'll also keep some time at the end for questions.

Okay... let me get started."

---

## SLIDE 2 — My Onboarding Sprint — Tools I Learned

---

[ACTION: Click to Slide 2]

---

"So before I jump into the project, let me just quickly give you a picture of what my onboarding sprint looks like.

The idea is — one tool per week, learn it properly, and build something with it.

So I started with Git and GitHub... version control, branching strategies, how we manage code in a team. This was the first thing because honestly, everything else depends on Git.

Then Jenkins... this is where I learned how to build CI/CD pipelines. How to automate builds, tests, and deployments so nobody has to do it manually.

Then Docker... this is containerisation. How to package an application so it runs the same way on any machine — whether it's my laptop, a Jenkins server, or a Kubernetes cluster.

Then Kubernetes... this is how we actually run those containers at scale. We use HSBC's internal Kubernetes platform called IKP.

And Ansible... configuration management and automation. This is how we manage deployments and configurations in a clean, reusable way using playbooks.

So these five tools together form a complete DevOps pipeline — from writing code all the way to running it in production.

Now let me talk a bit about each tool."

---

## SLIDE 3 — Tools — What They Are and Why They Matter

---

[ACTION: Click to Slide 3]

---

"Okay so this slide gives a quick background on each tool — what it is and why we use it specifically.

Starting with Git...

Git is our version control system. Every file in this project lives in a Git repository on HSBC's internal GitHub — alm-github.systems.uk.hsbc. And the way we use Git is through something called GitFlow branching strategy.

So the idea with GitFlow is — you never work directly on the master branch. Master is always production-ready. You create a feature branch for your work, develop there, and then raise a Pull Request to merge it back. This way, no one's half-finished work ever affects anyone else, and everything goes through a review before it merges.

For this project I created a branch called feature/log-generator... all my work happened there.

---

Moving to Jenkins...

Jenkins is our automation server. Without Jenkins, every step would be manual — someone would have to build the Docker image, push it to the registry, deploy it to Kubernetes, every single time. Jenkins does all of that automatically the moment you push code to GitHub.

And the pipeline definition itself — the Jenkinsfile — it lives right there in the Git repository alongside the code. So the pipeline is version controlled just like everything else.

---

Docker...

Docker solves the classic problem — 'it works on my machine' but breaks on the server. Different machines have different environments, different software versions... things break.

Docker packages the application and everything it needs into a container image. That image runs exactly the same way everywhere.

One important thing for HSBC specifically — we can't use Docker Hub or any public registry. All images must come from our internal Nexus3 registry. This is a security policy — the images there are vetted and approved by the organisation. So all our Dockerfiles point to Nexus3, not Docker Hub.

---

Kubernetes...

Kubernetes is where our containers actually run — on HSBC's IKP platform. The main value of Kubernetes is that it manages the container lifecycle automatically. If a container crashes, Kubernetes restarts it without anyone doing anything. And resource limits make sure one application doesn't consume everything on the shared cluster and affect other teams.

---

And Ansible...

Ansible is configuration management and automation. We use playbooks — YAML files that describe what you want to deploy or configure. In this pipeline, Jenkins calls an Ansible playbook for deployment instead of running kubectl commands directly. This makes the deployment logic reusable across multiple projects and environments.

Okay... so that's the background on the tools. Now let me show you how they all connect."

---

## SLIDE 4 — How All Tools Connect — Full Pipeline Flow

---

[ACTION: Click to Slide 4]

---

"So this is the most important slide in the whole presentation... because this shows how everything connects together.

What you're seeing here is the complete pipeline flow — from writing code all the way to running on Kubernetes. Let me walk through it step by step.

---

It all starts with Git.

When I push my code to the feature/log-generator branch on GitHub... Jenkins is watching. It detects the push and automatically triggers the pipeline. So Git is the starting point — nothing else happens without it.

---

Then Jenkins takes over.

Jenkins picks up the code and starts running through the Jenkinsfile stage by stage — checkout, build, run, push, deploy. All automatic. Nobody is sitting there running commands manually.

---

Then Docker Build.

Jenkins runs docker build, which reads our Dockerfile and builds a container image. The base image comes from Nexus3. The build number from Jenkins is used as the image tag — so every build produces a uniquely tagged image. This is important because if something breaks, you can trace exactly which build caused it.

---

Then Nexus3.

Once the image is built, Jenkins pushes it to our internal Nexus3 registry. The reason it goes to Nexus3 is — the Kubernetes cluster can only pull images from the internal registry. So the image has to be in Nexus3 before Kubernetes can use it.

---

And finally Kubernetes.

Jenkins applies our deployment YAML to the Kubernetes cluster. Kubernetes creates the Pod in the dev namespace, pulls the image from Nexus3, and runs the container.

---

You'll also see the note at the bottom — the whole pipeline is triggered by a single Git push to the feature branch. That's the power of CI/CD. One action... everything else is automatic.

---

And below that you can see Ansible. Jenkins calls an Ansible playbook for the deployment instead of running kubectl commands directly. This keeps the deployment logic separate, reusable, and easy to maintain across different projects.

So that's the full picture. Git feeds Jenkins, Jenkins drives Docker, Docker produces images for Nexus3, Nexus3 supplies the images to Kubernetes, and Ansible handles the deployment automation. Each tool has a specific role and together they form a complete automated pipeline."

---

## SLIDE 5 — Log Generator — Step by Step Pipeline Flow

---

[ACTION: Click to Slide 5]

---

"Okay now let me get more specific... this slide shows exactly what happens step by step when this pipeline runs.

---

Step 1 — Checkout.

Jenkins pulls the latest code from our internal GitHub onto the Jenkins agent's workspace. All four files — the script, Dockerfile, deployment YAML, and Jenkinsfile — are now available for the pipeline to work with.

---

Step 2 — Docker Build.

Jenkins builds the Docker image using the Dockerfile. The Dockerfile pulls the Ubuntu 20.04 base image from Nexus3, copies our generate_log.sh script into the container, gives it execute permission, and sets it as the startup command. The result is a Docker image tagged with the Jenkins build number.

---

Step 3 — Docker Run.

Jenkins starts the container from the image we just built. The container runs, generate_log.sh executes inside it, and the log lines get printed as output. Jenkins captures those logs and saves them to a file called application.log.

---

Step 4 — Push to Nexus3.

Jenkins tags the image with the full Nexus3 registry path and pushes it. This step is necessary because the Kubernetes cluster doesn't have access to the Jenkins agent's local Docker. So the image has to be in Nexus3 first.

---

Step 5 — Deploy and Undeploy.

Jenkins applies the deployment YAML to the Kubernetes dev namespace. Kubernetes creates the Pod, pulls the image from Nexus3, and runs the container. Then after the log is captured, Jenkins deletes the deployment.

The reason we undeploy after running is — this is a one-time log generator script. Once it's done its job, there's no reason to keep it running and using cluster resources. In a shared enterprise cluster, you clean up after yourself.

---

So that's the complete flow of the project end to end."

---

## SLIDE 6 — Why These Tools — Key Benefits to the Team

---

[ACTION: Click to Slide 6]

---

"Now I want to quickly cover why we use these specific tools... because a common question is, why not something else?

---

Git and GitFlow — the benefit is discipline and safety. Everyone's work is isolated in feature branches. Nobody's incomplete work breaks anyone else's. Pull Requests ensure code is reviewed before it merges. And since the Jenkinsfile also lives in Git, the pipeline itself is version controlled.

---

Jenkins — zero manual steps. One Git push triggers everything. Same steps, same order, every single time. No human error. And if something fails, you have the full build log to investigate exactly what went wrong.

---

Docker — it solves environment differences. I build the image on the Jenkins agent, push it to Nexus3, and Kubernetes runs it exactly the same way. No setup differences, no missing dependencies, no surprises.

---

Nexus3 — this is about security and compliance. HSBC's policy is that all Docker images must come from the internal registry. Images in Nexus3 are security-scanned and approved by the organisation. We can't use Docker Hub on company infrastructure — and honestly that's the right call for a bank.

---

Kubernetes — resilience and resource management. If a container crashes, Kubernetes restarts it automatically. Resource limits prevent one application from consuming everything on the shared cluster. And scaling is as simple as changing the replica count in the YAML.

---

And Ansible — reusability. The deployment logic is in a separate playbook that can be used across multiple projects. One change in the playbook applies everywhere. Much easier to maintain than having kubectl commands spread across different Jenkinsfiles."

---

## SLIDE 7 — Thank You

---

[ACTION: Click to Slide 7]

---

"So... to quickly wrap up what I covered today.

I walked you through the five tools in my sprint — Git, Jenkins, Docker, Kubernetes, and Ansible — what each one does and why it matters.

I showed you how they all connect in a complete automated pipeline — from a single Git push all the way to a running container on Kubernetes.

And I walked through the Log Generator project step by step — each stage of the pipeline and what it does.

Before I open it for questions... let me switch to my screen and quickly show you the actual project running. Because I think seeing it is much better than just talking about it."

---

## LIVE DEMO — Screen Share
### After Slide 7 | ~10 Minutes

---

[ACTION: Minimise the PPT. Open your browser. Have GitHub repo ready on feature/log-generator branch.]

---

"Okay let me switch to the GitHub repo now."

---

### PART 1 — GitHub Repo
### (3 minutes)

---

[ACTION: Open the devops_poc repo on the feature/log-generator branch]

---

"So this is our internal GitHub — alm-github.systems.uk.hsbc. This is the devops_poc repository and I'm on the feature/log-generator branch.

You can see the four files that make up this entire project.

generate_log.sh — this is the script, our application.
Dockerfile — how we package it into a Docker image.
The k8s folder with deployment.yaml — how Kubernetes runs it.
And Jenkinsfile — the pipeline that automates everything.

Let me quickly open each one."

---

[ACTION: Open generate_log.sh]

---

"This is the script. You can see it captures a timestamp at the top and then prints log lines — INFO, WARNING, ERROR — simulating what a real application would log. It also uses $BUILD_NUMBER which Jenkins automatically injects so every log is linked to its build.

Simple script... but the pipeline around it is what makes it interesting."

---

[ACTION: Go back and open Dockerfile]

---

"This is the Dockerfile. The FROM line points to our internal Nexus3 registry — not Docker Hub. Then we set the working directory, copy the script in, give it execute permission, and set it as the startup command."

---

[ACTION: Go back and open k8s/deployment.yaml]

---

"This is the Kubernetes manifest. Namespace is dev, replicas is 1, imagePullSecrets references the Nexus3 credentials stored as a Kubernetes secret, and resource limits are defined so the container doesn't consume more than it should on the shared cluster."

---

[ACTION: Go back and open Jenkinsfile]

---

"And this is the Jenkinsfile. You can see all the stages — Checkout, Docker Build, Docker Run, Push to Nexus3, OIDC Login, and Undeploy. Each stage does exactly what I described in the slides.

Okay... let me show you Jenkins now."

---

### PART 2 — Jenkins Pipeline
### (4 minutes)

---

[ACTION: Open Jenkins in browser. Navigate to the log-generator pipeline job.]

---

"So this is our internal Jenkins. This is the log-generator pipeline.

You can see the build history here — each build has a number, timestamp, and status. Green is passed, red is failed.

Let me open the last successful build."

---

[ACTION: Click the last successful build]

---

"So you can see all the stages across the top — Checkout, Docker Build, Docker Run, Push to Nexus3, OIDC Login, Undeploy — all green, all passed.

Let me open the console output so you can see what actually happened."

---

[ACTION: Click Console Output]

---

"This is the full build log. Let me scroll through it...

Here you can see Jenkins checking out the code from GitHub.

Here it's logging into Nexus3 and building the Docker image — pulling the Ubuntu base image from our internal registry.

Here it's running the container — and you can actually see our log output right here. The timestamp, the build number, the INFO and WARNING and ERROR lines — that's exactly what generate_log.sh produces.

Here it's pushing the image to Nexus3.

And here at the bottom — Undeploy — the deployment is cleaned up from the cluster.

The whole thing ran automatically from a single Git push. Nobody ran any of these commands manually."

---

[ACTION: Go back. Point to the build number.]

---

"One small thing I want to show... see this build number here? If you look at the log output, it says Build Number followed by this exact number. That's the $BUILD_NUMBER Jenkins variable being injected into the container at runtime. Every log file is linked back to the exact build that produced it."

---

### PART 3 — The Generated Log
### (2 minutes)

---

[ACTION: Open application.log from Jenkins artifacts or workspace if accessible]

---

"And this is the actual log that was generated — application.log.

You can see the header — HSBC Log Generator, the timestamp, the build number.

Then the log entries — INFO for normal things like startup and database connection, WARNING for high memory usage, ERROR for a transaction timeout, and then the retry.

This is simulating what a real banking application would log during its lifecycle. We kept the application simple on purpose — the goal of this project was to learn the DevOps pipeline, not build a complex application."

---

### CLOSING THE DEMO
### (1 minute)

---

[ACTION: Go back to GitHub repo home page]

---

"So that's the full project — live.

You saw the four files in GitHub, the pipeline running in Jenkins with every stage passing, and the actual log the container produced.

Everything I showed in the slides is real and working.

I'm happy to take any questions now... on the tools, the project, the pipeline, anything at all."

---

## HANDLING QUESTIONS

---

> Stay calm. Take a breath before answering. If you need a second just say "that's a good question, let me think about that for a second."

---

**Q: Why GitFlow and not something simpler like just using one branch?**

"GitFlow gives us better control in a team environment. Feature branches mean no one's work affects anyone else until it's reviewed and merged. In a bank like HSBC where stability and code quality matter a lot, that discipline is important. Simpler strategies like Trunk Based Development work well for smaller teams but GitFlow is a better fit for enterprise environments."

---

**Q: Why Ubuntu as the base image?**

"Ubuntu 20.04 is an LTS release — Long Term Support — so it gets security updates for 5 years. It's stable, widely used, and has Bash pre-installed which is exactly what our script needs. Alpine would be lighter and actually better for production, but our script uses Bash-specific features and setting that up on Alpine needs extra work. For this project Ubuntu was the simpler and safer choice."

---

**Q: Why Docker at all? Can't you just run the script directly?**

"You can run the script directly but then it depends on the host machine's environment. Different machines might have different Bash versions, different settings... Docker removes all of that. The container runs exactly the same way on any machine. And it also enables us to deploy on Kubernetes which wouldn't be possible with a plain script."

---

**Q: Why do you undeploy after running?**

"The log generator is a one-time task — it runs, produces a log, and exits. There's no reason to keep the Pod running after that. In a shared Kubernetes cluster, resources are limited and shared across teams. Cleaning up after yourself is just good practice."

---

**Q: Why Nexus3 and not Docker Hub?**

"HSBC's security policy doesn't allow pulling images from public registries on company infrastructure. All images must come from the internal Nexus3 registry because those images are security-scanned and approved by the organisation. It's a company policy, not a technical choice."

---

**Q: Can this project be extended?**

"Yes, quite a bit actually. The immediate things are — enabling the Kubernetes deploy stage which is already written and just commented out, and archiving the logs as a Jenkins build artifact. Further down, we could add SonarQube for code quality scanning, Splunk integration for log monitoring, and multi-environment deployment where the pipeline automatically deploys to dev, test, or prod based on which branch triggered it."

---

**Q: You used AI to make the PPT didn't you?**

"Yes, I used Claude to help me structure and design the presentation. But the project, the pipeline, the code, and everything I explained today — that's all my own work from this sprint. Using AI tools to work smarter is something I think is important to learn early."

---

**Q: Something you don't know the answer to**

"That's a good question honestly... I haven't looked into that specific part yet. Let me note it down and come back to you with a proper answer."

---

## FINAL CLOSE

---

If the session is wrapping up and no more questions:

"Alright... I think that covers everything. Thank you all for your time and for listening. It was a good learning experience putting this together and I'm looking forward to the next sprint.

Thanks everyone."

---

> 💪 You've got this Chandu.
> Speak slowly. Pause between slides.
> The demo is your strongest part — let them see it working.
> All the best for tomorrow!

---
