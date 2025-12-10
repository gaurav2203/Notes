# INTERVIEW-READY CHEAT SHEET for CI/CD

## Topic 1: GitLab Pipelines

**(Artifacts, Dependencies, Needs, Stages, Jobs, Runners, Rules)**

> **Memorize this.**
> **Say this in interviews.**
> *This will immediately set you apart from junior candidates.*

-----

## 🚀 GitLab CI/CD — INTERVIEW CHEAT SHEET

*(For DevOps Roles Working With GitLab Pipelines)*

### 🔵 1. Pipeline Structure

A GitLab pipeline is built from:

  * **Stages**
      * Define execution order: `build` → `test` → `deploy`
  * **Jobs**
      * Atomic execution units inside stages.
      * Jobs in the same stage run in **parallel**.
      * The next stage runs only when **all jobs** in the previous stage succeed.
  * **Runners**
      * Machines/agents that pick up jobs and run them using executors (Docker, Shell, Kubernetes, Custom runners).
      * *Note:* Jobs do not push themselves to runners → runners **pull** jobs from the queue.

### 🔵 2. Artifacts

**Artifacts** = files produced by a job and passed to later jobs.

  * Stored in GitLab artifact storage.
  * Persist only for the configured duration.
  * Must be explicitly fetched via `dependencies:` or `needs:`.

**Example:** Build a JAR in `compile` → `deploy` job downloads it.

```yaml
artifacts:
  paths:
    - target/app.jar
```

### 🔵 3. dependencies:

Controls which artifacts will be downloaded into a job's workspace.

  * Does **NOT** affect pipeline order.
  * Must reference jobs from **previous stages only**.
  * Used when normal stage-by-stage flow is acceptable.

**Example:**

```yaml
run_tests:
  dependencies:
    - compile
```

### 🔵 4. needs: (DAG Execution)

The most advanced CI/CD concept — **heavily tested**.
`needs:` enables a **Directed Acyclic Graph (DAG)** pipeline.

  * **Concept:** Jobs run based on *dependencies* rather than strict *stage order*.
  * **What this means:**
      * Jobs can start early, even if other jobs in earlier stages are still running.
      * Dramatically faster pipelines.
      * Optional artifact downloading.

**Example DAG:**

```yaml
test_job:
  stage: test
  needs:
    - job: compile
      artifacts: true
```

**This allows:**

1.  `test_job` starts **immediately** after `compile` finishes.
2.  It doesn't wait for the entire `build` stage to finish.

### 🔵 5. When to Use What

| Requirement | Use |
| :--- | :--- |
| Share build outputs (JAR, reports) | `artifacts` |
| Download artifacts from earlier jobs | `dependencies:` |
| Faster pipeline / out-of-order execution | `needs:` |
| Download artifacts + DAG execution | `needs` + `artifacts: true` |

> **🔥 Critical Rule:**
> If the interview question includes:
>
>   * “start early”
>   * “parallel”
>   * “speed up pipeline”
>   * “run without waiting for full stage”
>
> → The answer is **ALWAYS** `needs:`.

### 🔵 6. rules vs only/except

  * **rules** (Modern & Flexible)
      * Conditional logic based on: branch, variables, pipeline source, file changes.
      * Supports `when:` (manual, on\_failure, delayed).
  * **only/except** (Legacy)
      * Only filters by branch or tag.
      * *Advice:* Use `rules` for all modern pipelines.

### 🔵 7. Basic Pipeline Template (Interview-Grade)

```yaml
stages:
  - build
  - test
  - deploy

compile:
  stage: build
  script:
    - mvn -B package
  artifacts:
    paths:
      - target/*.jar

run_tests:
  stage: test
  dependencies:
    - compile
  script:
    - mvn test

deploy:
  stage: deploy
  needs:
    - job: run_tests
      artifacts: true
  script:
    - echo "Deploying..."
```

### 🔵 8. Explain GitLab Pipeline Execution Flow (1 Sentence)

> "Runners pull jobs from GitLab, execute them according to **stages** or **needs** relationships, pass **artifacts** through **dependencies** or **needs**, and enforce conditional logic through **rules**."


-----



# INTERVIEW CORRECTION: Rolling vs. Canary Updates

### ❌ Why the "Rolling Update" Answer Failed

  * **Late Detection:** Rolling updates replace pods gradually, meaning half the cluster runs the broken version before humans usually notice.
  * **No Isolation:** Rolling does not provide a "safe slice" of traffic for testing.
  * **Missing Metrics:** Senior answers must mention automated monitoring (KPIs/logs), not user testing.

### ✅ MODEL ANSWER (Interview Ready)

> “Rolling deployments replace pods gradually, so the failure wasn’t detected until half the cluster was already on the new version. Rolling provides no early-warning mechanism or traffic segmentation.
>
> Going forward, we should use a **Canary deployment**, which sends a small % of real production traffic to the new version and blocks further rollout if error rate, latency, or logs regress. This gives us early detection and near-instant rollback without impacting the majority of users.”

-----

# 🚀 TOPIC 2: GitLab Environments & Deployment Strategies

## 🔵 1. GitLab Environments

**Definition:** A named deployment target (dev/stage/prod) tracked by GitLab with history, rollbacks, URLs, and protection rules.

### Why define `environment:` in a job?

  * Tracks exactly which commit was deployed.
  * Enables one-click rollbacks.
  * Displays the environment URL in the UI.
  * Enforces protection rules (who can deploy).
  * Logically separates dev, test, stage, and prod.

### Environment Types

  * **Static:** Standard long-lived environments (e.g., `dev`, `staging`, `production`).
  * **Dynamic:** Ephemeral environments for testing branches (e.g., Review Apps).
      * Path: `review/$CI_COMMIT_REF_NAME`

### Manual Deployments

Used for production gates.

```yaml
deploy_prod:
  when: manual
  environment: production
```

**Use Cases:**

1.  Preventing accidental production releases.
2.  Change management / Approval gates.
3.  Enterprise compliance requirements.

-----

## 🔵 2. Deployment Strategies

### A. Rolling Deployment (Default)

  * **Mechanism:** Gradually replaces old pods with new pods.
  * **Pros:** No downtime, Simple, Cheap (low resource overhead).
  * **Cons:** Slow rollback, detection often happens too late.
  * **Use For:** SaaS, internal tools, low-risk systems.

### B. Blue/Green Deployment

  * **Mechanism:** Maintains two identical environments. Traffic switches instantly from Blue (current) to Green (new).
  * **Pros:** Zero downtime, Instant rollback.
  * **Cons:** Double the infrastructure cost.
  * **Use For:** Critical systems needing instant rollback capabilities.

### C. Canary Deployment

  * **Mechanism:** Releases new version to a small subset of users (1–5%) while monitoring metrics.
  * **Pros:** Safest strategy, real production metric validation, early failure detection.
  * **Cons:** Complex routing/Load Balancer rules required.
  * **Use For:** Banking, Fintech, Regulated systems, High-risk changes.

-----

## 🔵 3. Strategy Selection Matrix

| Use Case | Strategy | Why? |
| :--- | :--- | :--- |
| **Banking API / High-risk** | **Canary** | Detect problems early with real traffic. |
| **SaaS / Fast deployments** | **Rolling** | Fast execution + minimal cost. |
| **Internal Tools** | **Rolling** | Minimal configuration overhead. |
| **Need Instant Rollback** | **Blue/Green** | Traffic switch is immediate; old env remains. |

-----



# **📘 MULTI-LANGUAGE PIPELINE INTERVIEW CHEAT SHEET**

### **1. Each language has its own Docker image**

* Java → Maven/Gradle
* Python → python:3.x
* React → node:18
* .NET → dotnet SDK

Reason: Different compilers, dependency managers, runtimes.

---

### **2. Each language has its own cache path**

* Maven → `.m2/repository`
* Python → `.cache/pip`
* React → `node_modules/`
* .NET → `.nuget/packages`

Reason: Prevent cross-language corruption.

---

### **3. Each language produces different artifacts**

* Java → JAR
* Python → wheel (`.whl`)
* React → build folder
* .NET → publish output

Artifacts must be passed to test/deploy jobs.

---

### **4. Selective execution using `rules: changes:`**

Example:

```yaml
rules:
  - changes:
      - java-service/**/*
```

Prevents unnecessary builds → critical for monorepos.

---

### **5. Shared library logic**

Use:

* a separate build job for the shared library
* `rules: changes:` on shared_lib/**/*
* `needs:` from dependent services → ensures dependent rebuilds when library changes

---

### **6. Deploy only when Java changes**

Use:

```yaml
rules:
  - changes:
      - java-service/**/*
```

NEVER `needs:` for conditional execution.

---

### **7. Parallel builds**

All four languages can build in parallel in the `build` stage.

---



# ⭐ **TOPIC 4 — FULL INTERVIEW CHEAT SHEET (GitLab Variables, Secrets & CI Security)**

This is the exact sheet senior DevOps engineers would prepare for interviews.

---

# 🔵 **1. Variable Types (What They Do)**

### **Regular Variables**

* General configuration
* Visible in logs unless masked

### **Masked Variables**

* Hidden in logs (`***`)
* Cannot appear in artifacts
* Must match strict regex
* Protects *visibility*, not *access*

### **Protected Variables**

* Only available to pipelines on **protected branches/tags**
* Prevents leaks through MR pipelines or forks
* Protects *access*, not *visibility*

### **Environment-Scoped Variables**

* Different values per environment (dev/stage/prod)
* GitLab automatically selects correct value

### **File Variables**

* Used for certs, kubeconfigs, JSON keys
* Available as temporary files during the job

---

# 🔵 **2. Security Rules You MUST Know**

### **Masked != secure storage**

Secrets remain in plain text in job environment.

### **Protected = access control**

Secrets never reach untrusted branches or fork pipelines.

### **Mask + Protect together for production**

* Mask → hide in logs
* Protect → restrict access

### **NEVER store secrets in:**

* `.gitlab-ci.yml`
* commit history
* Dockerfiles
* environment files in repo

### **GitLab prevents leaking by:**

* hiding masked vars in logs
* blocking protected vars in untrusted pipelines
* rejecting masked vars that don’t match regex
* stripping secrets from artifacts

---

# 🔵 **3. Attack Scenarios You Must Mention**

* A malicious MR could echo secrets, send them over HTTP, write them to artifacts
* Masking does NOT prevent exfiltration
* Protected variables prevent this attack by blocking secret access entirely

---

# 🔵 **4. Deployment Patterns (Security Focus)**

### **Safe Production Deployment**

```yaml
deploy_prod:
  stage: deploy
  when: manual
  environment: production
  only:
    - main
```

Production secrets:

* masked
* protected
* environment-specific

---

# 🔵 **5. Interview One-Liners (Use These)**

### **Masked vs Protected**

> “Masked hides the value in logs. Protected restricts access to trusted branches.
> Masking protects visibility; protecting controls access.”

### **Why prod secrets must be masked + protected**

> “Masked so they never appear in logs, protected so only pipelines on `main` can use them.”

### **Why masked secrets are still vulnerable**

> “Masking hides secrets in logs but not in the environment; malicious code can still exfiltrate them.”

---
