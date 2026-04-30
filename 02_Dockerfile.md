# 📄 File 2: `Dockerfile` — Line by Line Explanation

---

## 🔎 What is this file?

A `Dockerfile` is a **text file with instructions** that tells Docker how to build a container image for your application.

Think of it like a **recipe** — it defines:
- What base operating system to start from
- Where to place your application code
- What permissions to set
- What command to run when the container starts

In this project, the Dockerfile packages our `generate_log.sh` script into a Docker image, so it can be run consistently anywhere — on a developer laptop, Jenkins server, or Kubernetes cluster — without any setup differences.

---

## 📋 Full Code with Line-by-Line Explanation

---

### Line 1
```dockerfile
FROM nexus3.systems.uk.hsbc:<port>/com/hsbc/group/ubuntu/gcr-ubuntu-2004:latest
```

**What it does:**
This is the **base image instruction**. It tells Docker: *"Start building from this existing image."*

Every Docker image is built on top of another image. Here we are using an Ubuntu 20.04 image sourced from the **HSBC internal Nexus3 Docker registry**.

Breaking it down:
| Part | Meaning |
|------|---------|
| `nexus3.systems.uk.hsbc` | **[CONFIDENTIAL]** — Internal Nexus3 registry hostname |
| `<port>` | **[CONFIDENTIAL]** — The port number for the registry (e.g., 8082, 8083) |
| `com/hsbc/group/ubuntu` | **[CONFIDENTIAL]** — Internal registry path/namespace |
| `gcr-ubuntu-2004` | The image name — Ubuntu 20.04 from Google Container Registry, mirrored internally |
| `:latest` | The image tag — always pulls the most recent version of this image |

**Why it is used:**
Every container needs a base OS to run on. We need Ubuntu because our script uses standard Linux Bash commands.

**Why this specific image (internal Nexus3)?**
HSBC does not allow pulling images directly from Docker Hub or public registries on company infrastructure. All base images must come from the **internal Nexus3 registry** which is security-scanned, approved, and controlled by the organisation. This is a **team/organisation decision** — not a technical preference.

**Why Ubuntu 20.04 specifically?**
Ubuntu 20.04 (LTS - Long Term Support) is a stable, widely-used Linux distribution. It has `bash` pre-installed, which is all our script needs. LTS versions are preferred in enterprise environments because they receive security patches for 5 years.

**Alternative approach:**
In a public/personal project, you could use:
```dockerfile
FROM ubuntu:20.04          # Docker Hub
FROM alpine:3.18           # Lightweight Alpine Linux (only ~5MB)
```
Alpine would be better for production (smaller image = faster pull, less attack surface). But since we are constrained to the internal registry, we use whatever is available there.

**Peer Question you may face:**
> *"Why can't we just use `FROM ubuntu:20.04` from Docker Hub?"*

Answer: HSBC's security policy does not allow pulling images from public registries on company laptops or infrastructure. All images must come from the internal Nexus3 registry which is vetted and approved. This is a company policy, not a technical limitation.

---

### Line 2
```dockerfile
WORKDIR /app
```

**What it does:**
Sets the **working directory** inside the container to `/app`. All subsequent instructions (`COPY`, `RUN`, `CMD`) will execute relative to this directory.

If `/app` doesn't exist, Docker automatically creates it.

**Why it is used:**
Without `WORKDIR`, files would be placed in the root `/` directory of the container, which is messy and bad practice — the root directory contains critical OS files. Keeping application files in a dedicated directory like `/app` is clean and organised.

**Why `/app` specifically?**
`/app` is the **industry convention** for application code inside containers. You could use `/workspace`, `/project`, or any name — but `/app` is what most developers expect to see, making it easier for others to understand.

**Alternative approach:**
You could skip `WORKDIR` and use absolute paths everywhere:
```dockerfile
COPY generate_log.sh /app/generate_log.sh
RUN chmod +x /app/generate_log.sh
CMD ["/app/generate_log.sh"]
```
This works but is repetitive. `WORKDIR` keeps it DRY (Don't Repeat Yourself).

**Peer Question you may face:**
> *"What happens if you don't use WORKDIR?"*

Answer: Files get placed in the root `/` directory by default. This is disorganised and can conflict with OS files. It also makes the Dockerfile harder to read and maintain.

---

### Line 3
```dockerfile
COPY generate_log.sh .
```

**What it does:**
Copies the `generate_log.sh` file from your **local machine (host)** into the container's working directory (`/app`).

The `.` means "copy to the current working directory" — which is `/app` because we set it with `WORKDIR`.

So effectively this copies: `host: ./generate_log.sh` → `container: /app/generate_log.sh`

**Why it is used:**
The container image starts as a blank Ubuntu OS. It has no knowledge of our script. We need to explicitly copy our application code into the image so it's available when the container runs.

**Why `COPY` and not `ADD`?**
Docker has two similar instructions — `COPY` and `ADD`. `COPY` is straightforward: it just copies files. `ADD` has extra features like auto-extracting tar archives and downloading from URLs. **Best practice is to use `COPY` unless you specifically need `ADD`'s extra features** — it's more predictable and transparent.

**Alternative approach:**
You could use `ADD` instead:
```dockerfile
ADD generate_log.sh .
```
This would work identically here, but `COPY` is preferred for simple file copying.

**Peer Question you may face:**
> *"Why do we copy the script into the image? Why not just reference it from outside?"*

Answer: Docker containers are **isolated** — they can't access the host filesystem by default. We must copy the script into the image to make it available inside the container. This also ensures the image is **self-contained** — it has everything it needs to run, regardless of where it's deployed.

---

### Line 4
```dockerfile
RUN chmod +x generate_log.sh
```

**What it does:**
`chmod +x` changes the file permissions to make `generate_log.sh` **executable**.

By default, when you copy a file into a Docker image, it may not have execute permissions — meaning the OS will refuse to run it as a script.

`+x` adds the **execute permission** for all users (owner, group, others).

**Why it is used:**
Without this, when the container tries to run `./generate_log.sh`, it will get a **Permission denied** error. This is a mandatory step for any shell script that needs to be executed directly.

**Why `RUN` and not doing it before the COPY?**
You could `chmod` the file on your local machine before copying it. But doing it in the Dockerfile ensures it is **always correct regardless of who builds the image** — some file systems (like Windows) don't preserve Linux permissions, so the execute bit can get lost during `COPY`. Doing it inside the Docker build guarantees it.

**Alternative approach:**
You could set permissions before copying:
```bash
chmod +x generate_log.sh   # on your local machine
```
But as mentioned, this is unreliable across different host operating systems. The `RUN chmod` in the Dockerfile is the safer, more portable approach.

**Peer Question you may face:**
> *"What does `+x` mean exactly?"*

Answer: `chmod` stands for "change mode" — it controls file permissions in Linux. `+x` means "add execute permission." Without it, the OS treats the file as a plain text file, not a runnable program.

---

### Line 5
```dockerfile
CMD ["./generate_log.sh"]
```

**What it does:**
Defines the **default command** that runs when the container starts.

`CMD` tells Docker: *"When someone runs this container, execute this command automatically."*

The array format `["./generate_log.sh"]` is called the **exec form** — it runs the command directly without a shell wrapper.

**Why it is used:**
Without `CMD`, the container would start and immediately exit because there's nothing to run. `CMD` defines the container's purpose — in our case, to run the log generator script.

**Why exec form `["./generate_log.sh"]` and not shell form `CMD ./generate_log.sh`?**
There are two ways to write CMD:
- **Shell form:** `CMD ./generate_log.sh` — runs via `/bin/sh -c`, which creates an extra shell process
- **Exec form:** `CMD ["./generate_log.sh"]` — runs the command directly as process ID 1

The **exec form is preferred** because:
1. The script runs as PID 1 — meaning it receives OS signals (like SIGTERM for graceful shutdown) directly
2. No extra shell process is created — cleaner and more efficient
3. It's the Docker-recommended best practice

**Difference between CMD and ENTRYPOINT:**
| Instruction | Behaviour |
|-------------|-----------|
| `CMD` | Default command — can be overridden when running the container |
| `ENTRYPOINT` | Fixed command — cannot be overridden easily |

We use `CMD` here because it gives flexibility — a developer could run the container with a different command for debugging if needed.

**Alternative approach:**
```dockerfile
ENTRYPOINT ["./generate_log.sh"]
```
This would make the script the fixed entry point and it cannot be overridden. For a simple script like this, either works — `CMD` is slightly more flexible.

**Peer Question you may face:**
> *"What's the difference between RUN and CMD?"*

Answer: `RUN` executes during the **image build** phase — it's used to install software, set permissions, etc. `CMD` executes when the **container starts** — it defines what the container does when it runs. They serve completely different purposes.

---

## 🔐 Confidential Markings

| Item | Confidentiality |
|------|----------------|
| `nexus3.systems.uk.hsbc` | **CONFIDENTIAL** — Internal registry hostname |
| `<port>` | **CONFIDENTIAL** — Internal registry port number |
| `com/hsbc/group/ubuntu` | **CONFIDENTIAL** — Internal image path/namespace |

---

## 🗺️ How this Dockerfile fits in the pipeline

```
generate_log.sh (your script)
        ↓
Dockerfile builds an image containing the script
        ↓
Jenkins runs: docker build → creates the image
        ↓
Jenkins runs: docker run → container starts → script executes → logs generated
        ↓
Jenkins captures logs → archives as application.log
        ↓
Image pushed to Nexus3 registry for reuse
```

---

## ❓ Common Peer Questions & Answers

| Question | Answer |
|----------|--------|
| Why do we need Docker at all? Can't we just run the script directly? | Yes, technically. But Docker ensures the script runs in a **consistent, isolated environment** regardless of the host machine. No "works on my machine" problems. It also enables Kubernetes deployment. |
| What is a Docker image vs a container? | An **image** is the blueprint (like a class in OOP). A **container** is a running instance of that image (like an object). You can run multiple containers from one image. |
| Why `:latest` tag? Is that good practice? | For learning/demo purposes it's fine. In production, you should use a **specific version tag** (e.g., `gcr-ubuntu-2004:20.04.1`) so builds are reproducible. `:latest` can change unexpectedly. |
| How big is this Docker image? | It depends on the base Ubuntu image size. Ubuntu 20.04 is typically around 70-80MB. Our additions (just a shell script) add virtually nothing. |
| What happens if the script fails inside the container? | The container exits with a non-zero exit code. Jenkins detects this and marks the build as failed. |
