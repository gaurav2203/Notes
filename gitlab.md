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

# 🔜 NEXT TOPIC: Multi-Language Pipelines

*(Java + Python + React + .NET)*

**Please choose your approach for Topic 3:**

1.  **Principles First:** Theory on Docker images, caching, artifacts, and patterns.
2.  **Practical:** Jump straight into writing real example pipelines.
3.  **Q\&A Mode:** Start directly with interview questions.

-----

**How would you like to proceed?**
