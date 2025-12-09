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

*This sentence alone gets you respect.*

-----

-----

**Would you like me to convert the next section for you once you decide on an option?**
