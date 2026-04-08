# GitLab DevOps Engineering — Senior Interview Q&A Catalog

> **Sources**: GitLab official docs, Ansh Lamba / Abhishek Veeramalla YouTube series, Cloud Champ GitLab CI/CD tutorial, senior DevOps interview transcripts.
> **Architecture diagram**: `gitlab_cicd_architecture.svg` (open in browser)
> **Target**: Senior DevOps Engineer — GitLab SaaS focus
> **Scope**: CI/CD Pipeline Design, GitLab Runners, Kubernetes integration, Security Scanning, Scenario-based

---

## TABLE OF CONTENTS

1. [CI/CD Fundamentals](#1-cicd-fundamentals)
2. [GitLab Runners](#2-gitlab-runners)
3. [Pipeline Design Patterns](#3-pipeline-design-patterns)
4. [Security Scanning](#4-security-scanning)
5. [Kubernetes Integration](#5-kubernetes-integration)
6. [GitLab Flow & Branching](#6-gitlab-flow--branching)
7. [Scenario-Based Questions](#7-scenario-based-questions)

---

## 1. CI/CD Fundamentals

### Q1. Walk me through the anatomy of a `.gitlab-ci.yml` file. What are the top-level keywords?
**Answer:**

`.gitlab-ci.yml` is the single source of truth for your GitLab pipeline. It lives at the root of your repository.

**Top-level (global) keywords:**
```yaml
# ── Pipeline-level keywords ──────────────────────────────────────
stages:           # define stage order
  - build
  - test
  - scan
  - package
  - deploy

variables:        # global CI/CD variables (can be overridden per-job)
  DOCKER_IMAGE: python:3.12-slim
  APP_ENV: staging

workflow:         # control WHEN a pipeline runs at all
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

include:          # import other YAML files (templates, shared configs)
  - template: Security/SAST.gitlab-ci.yml
  - project: 'my-group/ci-templates'
    file: '/templates/docker-build.yml'

default:          # defaults applied to all jobs unless overridden
  image: alpine:3.18
  before_script:
    - echo "Pipeline starting..."
  retry:
    max: 2
    when: runner_system_failure
  tags:
    - linux
```

**Per-job keywords:**
```yaml
build-app:
  stage: build
  image: node:20-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  needs: []         # DAG: start immediately, don't wait for previous stage
  tags:
    - docker
  environment:
    name: staging
  allow_failure: false
  timeout: 30 minutes
```

---

### Q2. What is the difference between `rules`, `only/except`, and `workflow`? When do you use each?
**Answer:**

**`only/except`** (legacy):
```yaml
# Old way — limited flexibility
deploy:
  only:
    - main
    - tags
  except:
    - schedules
```

**`rules`** (modern, preferred):
- Evaluated top-to-bottom; first matching rule wins
- Can set `when`, `allow_failure`, `variables` per rule
- Supports complex conditions with `&&`, `||`, regex, file changes

```yaml
deploy:
  rules:
    - if: $CI_COMMIT_TAG                          # tag push → deploy
      when: on_success
    - if: $CI_COMMIT_BRANCH == "main"             # main branch → deploy (manual)
      when: manual
      allow_failure: false
    - if: $CI_PIPELINE_SOURCE == "schedule"       # scheduled → skip
      when: never
    - changes:                                     # only if specific files changed
        - src/**/*
        - Dockerfile
      when: on_success
    - when: never                                  # default: don't run
```

**`workflow`** — controls whether a pipeline is created at all (before jobs are evaluated):
```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH       # branch push
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"             # MR pipeline
    - if: $CI_COMMIT_TAG                                           # tag
    - when: never                                                   # skip everything else (API, web)
```

**Decision guide:**
- Use `workflow` to prevent duplicate pipelines (e.g., stop running both branch AND MR pipelines simultaneously)
- Use `rules` on individual jobs for conditional execution
- Never use `only/except` in new pipelines — it's being soft-deprecated

---

### Q3. Explain `artifacts` vs `cache` in GitLab CI. What's the difference and when do you use each?
**Answer:**

| Aspect | `artifacts` | `cache` |
|---|---|---|
| Purpose | Pass files between jobs/stages | Speed up jobs by reusing downloaded dependencies |
| Storage | GitLab server (object storage) | Runner-local or external (S3/GCS/Azure) |
| Scope | Passed downstream to dependent jobs | Shared within same runner (or across via shared cache) |
| Guarantee | Exact files available to next job | Best-effort — may be stale or missing (warm-up run) |
| Expiry | `expire_in: 1 day` (default: 30 days) | Expires based on `key` change |
| Typical use | `dist/`, test results, Docker image tar, coverage report | `node_modules/`, `.venv/`, Maven `.m2/`, Gradle cache |

```yaml
# ARTIFACTS: build → test pipeline
build:
  stage: build
  script: npm run build
  artifacts:
    paths:
      - dist/             # passed to downstream jobs
      - coverage.xml
    reports:
      junit: test-results.xml      # parsed by GitLab MR widget
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    expire_in: 2 days
    when: always          # keep artifacts even if job fails

# CACHE: dependency caching
test:
  stage: test
  cache:
    key:
      files:
        - package-lock.json     # cache key based on lockfile hash
    paths:
      - node_modules/
    policy: pull            # only pull cache (don't update it in test job)
  script:
    - npm test
```

**`cache.policy`:**
- `pull-push` (default): pull at start, push at end
- `pull`: read-only (test jobs should use this)
- `push`: write-only (used in a dedicated cache-warmup job)

---

### Q4. What are GitLab CI/CD Variables? Explain the different types and their precedence.
**Answer:**

**Variable types:**
| Type | Where defined | Who can see |
|---|---|---|
| Predefined | GitLab CI system | All jobs automatically |
| Project variables | Project → Settings → CI/CD → Variables | Jobs in that project |
| Group variables | Group → Settings → CI/CD → Variables | All projects in group |
| Instance variables | Admin → Settings → CI/CD | All projects on instance |
| `.gitlab-ci.yml` variables | `variables:` block | That pipeline/job |
| dotenv artifact | Passed from job to downstream | Consuming jobs |

**Precedence (highest to lowest):**
1. Trigger variables / pipeline variables (passed via API/trigger token)
2. Job-level `variables:` in YAML
3. Pipeline-level `variables:` in YAML
4. Project CI/CD variables
5. Group CI/CD variables
6. Instance CI/CD variables
7. Predefined variables

**Protected vs Masked:**
- **Protected**: only available in jobs running on protected branches/tags — prevents leaking secrets to feature branch pipelines
- **Masked**: value is redacted in job logs (`[MASKED]`) — requires value to match masking requirements (no newlines, ≥8 chars)
- Best practice: secrets should be **both** protected AND masked

**Key predefined variables:**
```bash
$CI_COMMIT_SHA          # full commit hash
$CI_COMMIT_REF_SLUG     # branch/tag name, URL-safe (e.g. feature-foo)
$CI_COMMIT_TAG          # tag name (only set on tag pipelines)
$CI_PROJECT_PATH        # group/project-name
$CI_REGISTRY_IMAGE      # registry.gitlab.com/group/project
$CI_PIPELINE_SOURCE     # push, merge_request_event, schedule, trigger, api
$CI_ENVIRONMENT_SLUG    # environment name, URL-safe
$CI_JOB_TOKEN          # short-lived token for registry/API auth within pipeline
```

---

### Q5. What is a DAG pipeline (`needs:`) and how does it differ from stage-based execution?
**Answer:**

**Stage-based (default):** all jobs in stage N must complete before any job in stage N+1 starts.

```
Stage:  build          test          deploy
        [compile] ──→  [unit-test]   [deploy-prod]
                       [lint]
                       [integration]  ← BLOCKS deploy even if it doesn't need compile output
```

**DAG with `needs:`**: a job starts as soon as its declared dependencies finish — ignores stage ordering.

```yaml
stages: [build, test, deploy]

compile:
  stage: build
  artifacts:
    paths: [bin/]

unit-test:
  stage: test
  needs: [compile]        # starts as soon as compile finishes

integration-test:
  stage: test
  needs: [compile]        # also starts as soon as compile finishes (parallel!)

lint:
  stage: test
  needs: []               # needs nothing — starts immediately at pipeline creation

deploy-staging:
  stage: deploy
  needs: [unit-test, integration-test]  # waits for both
```

**Result:** total pipeline time can drop significantly — `lint` runs in parallel with `compile`, `unit-test` and `integration-test` run in parallel as soon as `compile` is done.

**`needs: []`** (empty) means the job starts immediately with no dependencies — useful for jobs that don't need artifacts (lint, security scans that work on source).

**Limitations:**
- `needs:` jobs can only reference jobs earlier in the DAG (no cycles)
- Cross-stage `needs:` requires `needs:` job to be in an earlier or same stage OR use `needs: [{job, pipeline}]` for cross-project

---

### Q6. How does `parallel:matrix` work and when do you use it?
**Answer:**

`parallel:matrix` fans out a single job definition into multiple parallel jobs, each with a different set of variable values.

```yaml
test-matrix:
  stage: test
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.10", "3.11", "3.12"]
        OS: ["ubuntu", "alpine"]
      - PYTHON_VERSION: ["3.12"]
        OS: ["windows"]
  image: python:${PYTHON_VERSION}-${OS}
  script:
    - python -m pytest tests/

# Creates 7 parallel jobs:
# test-matrix: [3.10, ubuntu], [3.10, alpine], [3.11, ubuntu], [3.11, alpine],
#              [3.12, ubuntu], [3.12, alpine], [3.12, windows]
```

**Use cases:**
- Multi-platform testing (OS × language version matrix)
- Deploying to multiple regions/environments in parallel
- Database compatibility testing (MySQL 5.7, 8.0, PostgreSQL 14, 15)
- Sharding a large test suite across N parallel runners

```yaml
# Test sharding: run N instances, each handles a slice
test-sharded:
  parallel: 5
  script:
    - pytest tests/ --splits 5 --group $CI_NODE_INDEX
  # CI_NODE_INDEX: 1..5, CI_NODE_TOTAL: 5
```

---

### Q7. Explain `include` and how you build a shared CI/CD template library.
**Answer:**

`include` imports YAML from four sources:

```yaml
include:
  # 1. GitLab-provided templates (built-in library)
  - template: Security/SAST.gitlab-ci.yml
  - template: Docker.gitlab-ci.yml

  # 2. Another file in same repo
  - local: .gitlab/ci/docker-build.yml

  # 3. File from another project (must have access)
  - project: 'my-org/ci-templates'
    ref: main
    file:
      - '/templates/python-test.yml'
      - '/templates/deploy-k8s.yml'

  # 4. Remote URL (HTTPS)
  - remote: 'https://raw.githubusercontent.com/.../template.yml'
```

**Building a shared template library (enterprise pattern):**

```
my-org/ci-templates/
├── templates/
│   ├── python-test.yml      # reusable Python test job
│   ├── docker-build.yml     # standard Docker build + push
│   ├── deploy-k8s.yml       # Helm deploy template
│   └── security-scan.yml    # org-wide security config
└── .gitlab-ci.yml           # tests the templates themselves
```

**Template using `extends` (job inheritance):**
```yaml
# In templates/docker-build.yml
.docker-build-template:    # dot prefix = hidden job (not run directly)
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# In consuming project's .gitlab-ci.yml
include:
  - project: 'my-org/ci-templates'
    file: '/templates/docker-build.yml'

build-image:
  extends: .docker-build-template   # inherits all keys
  variables:
    DOCKERFILE: Dockerfile.prod      # override specific variables
```

**`!reference` tag** (more powerful than `extends`):
```yaml
# Reference a specific section from another job
test:
  script:
    - !reference [.setup-script, script]  # re-use script block
    - pytest tests/
```

---

### Q8. How do parent-child pipelines and multi-project pipelines work?
**Answer:**

**Parent-Child Pipelines** (single repository, modular CI):
```yaml
# .gitlab-ci.yml (parent)
stages: [trigger]

trigger-backend:
  stage: trigger
  trigger:
    include: backend/.gitlab-ci.yml    # child pipeline from same repo
    strategy: depend                   # parent waits for child result

trigger-frontend:
  stage: trigger
  trigger:
    include: frontend/.gitlab-ci.yml
    strategy: depend
```

**Use case**: monorepo with multiple components — each component has its own CI file, parent orchestrates.

**Multi-Project Pipelines** (cross-repository):
```yaml
# Trigger a pipeline in another project
trigger-deploy:
  stage: deploy
  trigger:
    project: my-org/infrastructure    # another GitLab project
    branch: main
    strategy: depend
  variables:
    APP_VERSION: $CI_COMMIT_SHA       # pass variables to downstream
```

**Key difference:**
| | Parent-Child | Multi-Project |
|---|---|---|
| Repo | Same | Different |
| Variables | Automatically inherited | Must pass explicitly |
| Access | No extra permissions | Needs trigger token or group access |
| Use case | Monorepo, dynamic pipelines | Deploy pipeline, infra automation |

**Dynamic child pipelines** (generate YAML at runtime):
```yaml
generate-config:
  stage: .pre
  script:
    - python generate_ci.py > generated-pipeline.yml
  artifacts:
    paths: [generated-pipeline.yml]

trigger-dynamic:
  stage: build
  trigger:
    include:
      - artifact: generated-pipeline.yml
        job: generate-config
    strategy: depend
```

---

### Q9. What is `resource_group` and why is it important for deployments?
**Answer:**

`resource_group` ensures that only **one job with a given resource group name runs at a time** across all pipeline runs. It creates a deployment mutex.

```yaml
deploy-production:
  stage: deploy
  resource_group: production       # only 1 instance of this group runs concurrently
  environment:
    name: production
  script:
    - helm upgrade --install myapp ./chart
```

**Why it matters:**
- Without it: two pipelines triggered in quick succession can run `deploy-production` simultaneously → race condition, partial deployment, Helm state corruption
- With it: second job queues, waits for first to finish, then runs

**Process modes** (GitLab 14.3+):
```yaml
deploy-production:
  resource_group: production
  # process_mode: unordered     (default) — any queued job can run next
  # process_mode: oldest_first  — FIFO queue
  # process_mode: newest_first  — LIFO (skip stale deployments)
```

`newest_first` is the most useful for CD: if 3 deployments pile up, only the newest one runs — the intermediate ones are cancelled.

---

### Q10. How do GitLab Environments work and what is the difference between `environment.name` and `environment.url`?
**Answer:**

**Environments** track where your code is deployed and give you:
- Deployment history per environment
- A "view deployment" button linking to the live app
- Rollback to any previous deployment
- Environment-specific CI/CD variables
- Required approvals before deployment (protected environments)

```yaml
deploy-staging:
  stage: deploy
  environment:
    name: staging                                   # environment name
    url: https://staging.myapp.com                  # link shown in GitLab UI
    on_stop: stop-staging                           # job to call when stopping
    auto_stop_in: 1 day                             # auto-stop after 1 day
  script:
    - helm upgrade --install myapp-staging ./chart

stop-staging:
  stage: deploy
  environment:
    name: staging
    action: stop                                    # marks environment as stopped
  when: manual
  script:
    - helm uninstall myapp-staging

# Dynamic environments (review apps)
deploy-review:
  stage: deploy
  environment:
    name: review/$CI_COMMIT_REF_SLUG               # unique per branch
    url: https://$CI_COMMIT_REF_SLUG.review.myapp.com
    on_stop: stop-review
    auto_stop_in: 3 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Protected environments**: In Project → Settings → CI/CD → Environments, mark `production` as protected:
- Only users with "Deployer" role can trigger deployment jobs
- Requires approval from N people before job runs
- Blocks job from running on certain pipeline types

---

## 2. GitLab Runners

### Q11. Explain the GitLab Runner architecture. How does a runner pick up a job?
**Answer:**

**Components:**
```
GitLab.com (server)
   ├── Job queue (per project/group)
   └── Coordinator API (/api/v4/jobs/request)

GitLab Runner (agent, runs on your infra or GitLab's infra)
   ├── Manager process (polls coordinator)
   └── N worker goroutines (execute jobs concurrently)
        └── Executor (Docker / K8s / Shell / etc.)
             └── Clone repo → run script → upload artifacts
```

**Job pickup flow (polling model):**
1. Runner polls `POST /api/v4/jobs/request` every second (short-poll, configurable)
2. GitLab matches queued jobs to runners based on: **tags**, **runner type** (shared/group/project), **protected status**
3. Runner receives job token + job definition (script, image, variables, artifacts config)
4. Runner creates executor (e.g., spins up Docker container with specified image)
5. Runner injects CI/CD variables as environment variables into the executor
6. Runs: `before_script` → `script` → `after_script`
7. Uploads artifacts to GitLab, sends logs in real-time via WebSocket
8. Reports job result (success/failure) back to coordinator

**Key: GitLab runners are pull-based**, not push. GitLab never pushes jobs to runners — runners poll GitLab. This means runners don't need inbound ports open, just outbound HTTPS to GitLab.

---

### Q12. What are the different executor types and when do you choose each?
**Answer:**

| Executor | Isolation | Use case | Notes |
|---|---|---|---|
| **Docker** | Container per job | Most common — clean, isolated, any image | Requires Docker daemon; use `docker:dind` for Docker-in-Docker |
| **Kubernetes** | Pod per job | Cloud-native, auto-scales pods | Best for Kubernetes-based infra |
| **Shell** | Process on runner host | Legacy, fast, no isolation | Avoid — pollutes runner host, security risk |
| **Docker Machine** | VM + Docker per job | GitLab SaaS shared runners (legacy) | Being deprecated; replaced by auto-scaling |
| **VirtualBox / Parallels** | Full VM per job | Windows/macOS testing | Slow; only when OS-level isolation needed |
| **SSH** | Remote machine | Run on pre-existing servers | Very limited isolation |
| **Instance (new)** | VM per job | Replacement for Docker Machine | AWS, GCP auto-scaling VMs |

**Docker executor — most important config:**
```toml
[[runners]]
  name = "docker-runner"
  url = "https://gitlab.com"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    image = "alpine:latest"          # default image if not specified in job
    privileged = false               # NEVER true unless DinD required
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
    pull_policy = ["if-not-present"] # avoid pulling image every job (speed)
```

**Kubernetes executor:**
```toml
[[runners]]
  executor = "kubernetes"
  [runners.kubernetes]
    namespace = "gitlab-runners"
    image = "alpine:latest"
    cpu_limit = "2"
    memory_limit = "2Gi"
    cpu_request = "500m"
    memory_request = "512Mi"
    service_account = "gitlab-runner-sa"
    poll_interval = 3
    poll_timeout = 180
```

---

### Q13. What is the difference between Shared, Group, and Project runners?
**Answer:**

| Type | Scope | Who configures | Typical use |
|---|---|---|---|
| **Shared runners** | All projects on GitLab.com | GitLab admins (SaaS: GitLab team) | Open source, small teams, no custom needs |
| **Group runners** | All projects in a group/subgroup | Group owners | Org-wide runners with custom config, private network access |
| **Project runners** | Single project only | Project maintainers | Highly specific needs, specialized hardware (GPU, macOS) |

**Precedence for job assignment**: Project runners are checked first, then group runners, then shared runners.

**SaaS shared runner characteristics (GitLab.com):**
- Linux x86-64: Google Cloud `n2d-standard-2` (2 vCPU, 8 GB RAM)
- macOS: Apple Silicon (M1), limited minutes
- Windows: available on Premium/Ultimate
- Ephemeral: each job gets a fresh VM (not container — full VM for security isolation)
- Free tier: 400 CI/CD minutes/month; paid tiers: 10,000+/month

**When to use group runners (self-managed):**
- Need access to private registry, internal services, VPN
- Need larger instances (16 vCPU, 64 GB RAM)
- Need specific tools pre-installed
- Need GPU for ML training
- Cost optimization (own infrastructure)

---

### Q14. How do you configure runner autoscaling for cost efficiency?
**Answer:**

**Option 1: GitLab Runner Autoscaler (new, recommended)**
Uses cloud provider APIs to spin up VMs on demand:

```toml
# config.toml
[runners.autoscaler]
  capacity_per_instance = 1
  max_use_count = 1                    # 1 job per VM (ephemeral)
  max_instances = 20
  [runners.autoscaler.plugin]
    name = "fleeting-plugin-googlecompute"   # or aws, azure
  [runners.autoscaler.plugin.config]
    project = "my-gcp-project"
    zone = "us-central1-a"
    instance_group = "gitlab-runner-mig"    # managed instance group

[[runners.autoscaler.policy]]
  idle_count = 2                   # always keep 2 warm VMs
  idle_time = "30m"                # terminate VMs idle for 30 min
  scale_factor = 1.5               # add 50% extra capacity during spikes
  scale_factor_limit = 5
```

**Option 2: Kubernetes executor autoscaling**
Leverage your K8s cluster's HPA/Karpenter:
```yaml
# Runner operator scales runner pods
# K8s cluster scales nodes based on pending pods
# Most cost-efficient: spot/preemptible nodes for runner workloads
```

**Cost optimization strategies:**
1. Use spot/preemptible VMs for workers (60-80% savings)
2. Set `max_use_count = 1` for security (each job gets fresh VM)
3. Use `idle_count = 0` during off-hours (weekend/night schedule)
4. Pre-bake large tool installations into custom VM images (faster, cheaper than init downloads)
5. Group runner tags so GPU runners only get GPU jobs (not generic ones)

---

### Q15. What are runner tags and how do they control job assignment?
**Answer:**

**Tags** are labels on runners that jobs can match against. A job with tags will only run on runners that have **all** of those tags.

```yaml
# Job requests a runner with these tags
test-gpu:
  tags:
    - gpu
    - linux
    - large-runner
  script:
    - python train.py

# Runner config.toml
[[runners]]
  name = "gpu-runner-01"
  tags = ["gpu", "linux", "large-runner", "nvidia"]
```

**Rules:**
- Job with NO tags → runs on any runner that allows untagged jobs
- Job with tags → runner must have ALL specified tags
- Runner config: `run_untagged = false` to prevent it from picking up untagged jobs

**Enterprise tagging strategy:**
```
Tags by capability:
  linux, windows, macos          ← OS
  docker, kubernetes, shell      ← executor
  large, medium, small           ← instance size
  gpu, high-memory               ← special hardware
  prod-network, dev-network      ← network zone
  protected                      ← only for protected branches (security)

Protected runners: only assigned to jobs running on protected branches/tags
  → prevents secret leakage to feature branch pipelines
```

---

### Q16. How do you secure a self-managed GitLab Runner?
**Answer:**

**1. Never run privileged unless absolutely necessary**
```toml
[runners.docker]
  privileged = false   # default and correct for most workloads
  # privileged = true  # only if you NEED Docker-in-Docker
```
Privileged containers can escape to the host — treat it as root on the host machine.

**2. Use Docker-in-Docker alternatives**
```yaml
# Instead of privileged DinD, use:
# Option A: Kaniko (rootless image builds)
build-image:
  image:
    name: gcr.io/kaniko-project/executor:v1.21.0-debug
    entrypoint: [""]
  script:
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile Dockerfile
      --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# Option B: Buildah, Podman in rootless mode
```

**3. Limit network access**
```toml
[runners.docker]
  network_mode = "none"     # no network by default; override per-job if needed
  # Or use a restricted bridge with only allowed outbound
```

**4. Rotate runner authentication tokens**
```bash
# GitLab 15.x+: runner authentication tokens replace registration tokens
# Set token rotation in runner settings
# Use short-lived tokens with automatic renewal
```

**5. Ephemeral runners**
- `max_use_count = 1` in autoscaler: VM is destroyed after 1 job
- Prevents cross-job secret leakage via filesystem/memory

**6. Least-privilege service account**
```yaml
# K8s executor: dedicated service account with minimal RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-runner-jobs
  namespace: gitlab-runners
# Only bind roles the CI jobs actually need
```

**7. Protected runners for production deployments**
- Mark runner as "protected" in GitLab
- Only jobs on protected branches/tags run on it
- Prevent feature branch code from accessing production secrets

---

### Q17. How do you debug a failing runner or stuck pipeline?
**Answer:**

**Step 1: Check job log in GitLab UI**
```
Pipeline → Job → View Log
Look for:
  - "Pulling image..." hanging → image pull issue (auth, size, rate limit)
  - "Waiting for runner..." → no runner available with matching tags
  - "ERROR: Job failed: exit code 1" → script error (look above in log)
  - "ERROR: Job failed (system failure)" → runner crashed, network issue
```

**Step 2: Check runner system logs**
```bash
# On runner host
journalctl -u gitlab-runner -f
# or
sudo gitlab-runner verify                          # check connectivity to GitLab
sudo gitlab-runner list                            # list registered runners
```

**Step 3: Add debug output to pipeline**
```yaml
debug-job:
  script:
    - set -x                          # print every command
    - env | sort | grep CI_           # dump all CI variables
    - df -h                           # check disk space
    - docker info                     # check Docker daemon (if docker executor)
    - cat /etc/resolv.conf            # DNS issues
```

**Step 4: Runner `--debug` flag**
```bash
sudo gitlab-runner --debug run
```

**Common issues and fixes:**

| Symptom | Likely Cause | Fix |
|---|---|---|
| "Waiting for runner" forever | No runner with matching tags | Check tags; ensure runner is online |
| Image pull fails | Auth expired / wrong registry | Check `$CI_REGISTRY_*` variables; `docker login` step |
| Job stuck at 0% | Runner zombie / system failure | Restart runner service |
| OOM kill | Job exceeds memory limit | Increase `memory_limit` in runner config |
| Cache restore fails | Cache key mismatch or S3 issue | Check `cache.key`; check S3 credentials |
| Flaky network timeouts | Runner behind strict firewall | Whitelist GitLab IPs; check DNS |

---

---

## 3. Pipeline Design Patterns

### Q18. How do you implement environment promotion (dev → staging → production) in GitLab CI?
**Answer:**

**Pattern: sequential promotion with manual gates**

```yaml
stages: [build, test, deploy-dev, deploy-staging, deploy-prod]

variables:
  CHART_NAME: myapp
  HELM_TIMEOUT: 5m

# ── Reusable deploy template ──────────────────────────────────────
.deploy-template:
  image: alpine/helm:3.14.0
  before_script:
    - helm repo add myrepo $HELM_REPO_URL
    - kubectl config use-context $KUBE_CONTEXT

deploy-dev:
  extends: .deploy-template
  stage: deploy-dev
  environment:
    name: development
    url: https://dev.myapp.com
  variables:
    KUBE_CONTEXT: dev-cluster
    REPLICAS: 1
  script:
    - helm upgrade --install $CHART_NAME myrepo/$CHART_NAME
        --set image.tag=$CI_COMMIT_SHA
        --set replicas=$REPLICAS
        --namespace dev --create-namespace
        --timeout $HELM_TIMEOUT --wait
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy-staging:
  extends: .deploy-template
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.myapp.com
  variables:
    KUBE_CONTEXT: staging-cluster
    REPLICAS: 2
  script:
    - helm upgrade --install $CHART_NAME myrepo/$CHART_NAME
        --set image.tag=$CI_COMMIT_SHA --set replicas=$REPLICAS
        --namespace staging --timeout $HELM_TIMEOUT --wait
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-production:
  extends: .deploy-template
  stage: deploy-prod
  environment:
    name: production
    url: https://myapp.com
  variables:
    KUBE_CONTEXT: prod-cluster
    REPLICAS: 5
  when: manual                # human approval required
  allow_failure: false        # pipeline stays blocked until approved/declined
  resource_group: production  # prevent concurrent production deploys
  script:
    - helm upgrade --install $CHART_NAME myrepo/$CHART_NAME
        --set image.tag=$CI_COMMIT_SHA --set replicas=$REPLICAS
        --namespace production --timeout $HELM_TIMEOUT --wait
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

---

### Q19. How do you implement rollback in a GitLab CI/CD pipeline?
**Answer:**

**Strategy 1: Helm rollback (most common for K8s)**
```yaml
rollback-production:
  stage: deploy-prod
  environment:
    name: production
    action: start
  when: manual
  resource_group: production
  script:
    - helm rollback $CHART_NAME --namespace production
    # Or to specific revision:
    - helm rollback $CHART_NAME 3 --namespace production
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

**Strategy 2: Re-deploy previous Docker image tag**
```yaml
# Use GitLab's Environments UI: Deployments → re-deploy any previous deployment
# This re-runs the deploy job with that commit's variables ($CI_COMMIT_SHA)
```

**Strategy 3: GitOps rollback (git revert)**
```bash
# Revert the commit → new pipeline → automatic redeploy to previous state
git revert $BAD_COMMIT_SHA
git push origin main
# GitLab pipeline runs → deploys reverted code
```

**Strategy 4: Blue-Green with instant traffic switch**
```yaml
deploy-green:
  script:
    - helm upgrade --install myapp-green ./chart --set image.tag=$CI_COMMIT_SHA
    - kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

rollback-to-blue:
  when: manual
  script:
    - kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
    # Instant: just changes where Service routes traffic
```

---

### Q20. What is the `dotenv` artifact and how do you pass dynamic values between jobs?
**Answer:**

`dotenv` reports allow a job to **export variables that downstream jobs can use** — critical when job A computes a value (e.g., a version tag) that job B needs.

```yaml
# Job A: compute and export version
set-version:
  stage: .pre
  script:
    - VERSION=$(git describe --tags --always)
    - echo "APP_VERSION=${VERSION}" >> variables.env
    - echo "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> variables.env
  artifacts:
    reports:
      dotenv: variables.env    # magic: these become CI variables for downstream jobs

# Job B: use the exported variable
build-image:
  stage: build
  needs:
    - job: set-version
      artifacts: true          # must explicitly request artifact
  script:
    - echo "Building version $APP_VERSION"     # $APP_VERSION from dotenv
    - docker build --label "version=$APP_VERSION" -t $CI_REGISTRY_IMAGE:$APP_VERSION .
    - docker push $CI_REGISTRY_IMAGE:$APP_VERSION

deploy:
  stage: deploy
  needs:
    - job: set-version
      artifacts: true
    - job: build-image
  script:
    - helm upgrade --install myapp ./chart --set image.tag=$APP_VERSION
```

**Limitation**: dotenv variables only propagate to `needs:` downstream jobs, not all downstream jobs. Must use `needs: [{job, artifacts: true}]`.

---

### Q21. How do you handle secrets in GitLab CI beyond CI/CD variables?
**Answer:**

**GitLab native: CI/CD Variables (baseline)**
```yaml
# Access in pipeline:
script:
  - echo $DB_PASSWORD     # set in Project → Settings → CI/CD → Variables (masked + protected)
```

**HashiCorp Vault integration (recommended for enterprise)**
```yaml
# .gitlab-ci.yml
deploy:
  secrets:
    DATABASE_PASSWORD:
      vault: production/db/password@secret    # path@mount
      file: false                              # true = write to tmp file instead of env var
  script:
    - echo "Connecting with $DATABASE_PASSWORD"
```
GitLab uses JWT auth with Vault — no static tokens, short-lived.

**AWS Secrets Manager / Azure Key Vault (via script)**
```yaml
fetch-secrets:
  image: amazon/aws-cli
  script:
    - export DB_PASS=$(aws secretsmanager get-secret-value
        --secret-id prod/myapp/db --query SecretString --output text)
    - echo "DB_PASS=${DB_PASS}" >> secrets.env
  artifacts:
    reports:
      dotenv: secrets.env
```

**Best practices:**
1. Never hardcode secrets in `.gitlab-ci.yml` (it's in git history)
2. Never echo/print secrets — they appear in logs even if masked
3. Use `protected + masked` for all production secrets
4. Rotate secrets regularly — GitLab variables support rotation without pipeline changes
5. Use Vault or cloud secret managers for short-lived dynamic credentials
6. Audit secret access via GitLab audit events

---

### Q22. Explain pipeline efficiency: how do you reduce overall pipeline execution time?
**Answer:**

**1. Use `needs:` to eliminate stage wait times**
```yaml
# Without needs: each stage waits for previous → serial
# With needs: jobs start as soon as their deps finish → parallelism
lint:
  needs: []          # start immediately, no deps
security-scan:
  needs: []          # in parallel with lint
unit-test:
  needs: [build]     # starts right after build
```

**2. Optimize caching**
```yaml
# Cache lockfile-keyed node_modules — only invalidated when deps change
cache:
  key:
    files: [package-lock.json]
  paths: [node_modules/]
  policy: pull          # test jobs: read-only (faster)

# Separate cache warm-up job that runs on schedule
warm-cache:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  cache:
    policy: push        # only this job writes cache
  script:
    - npm ci            # populates cache
```

**3. Parallelism with `parallel:matrix`**
```yaml
test:
  parallel:
    matrix:
      - GROUP: [1, 2, 3, 4]   # 4 parallel test runners
  script:
    - pytest tests/ --splits 4 --group $GROUP
```

**4. Use lightweight images for simple jobs**
```yaml
lint:
  image: node:20-alpine    # 180MB vs node:20 at 1GB
  # alpine images: 3-10x smaller → faster pull → faster start
```

**5. Skip unnecessary jobs with `rules: changes`**
```yaml
backend-test:
  rules:
    - changes:
        - backend/**/*
        - requirements.txt
      when: on_success
    - when: never         # skip if no backend files changed
```

**6. Use interruptible pipelines** (cancel stale MR pipelines)
```yaml
build:
  interruptible: true    # old build is cancelled when new commit pushed to same MR
```

---

## 4. Security Scanning

### Q23. What security scanning tools does GitLab provide and how are they integrated?
**Answer:**

GitLab offers a comprehensive suite of Application Security Testing (AST) tools, all available as pre-built CI templates:

| Scanner | Tool | What it scans | Template |
|---|---|---|---|
| **SAST** | Semgrep, Gosec, ESLint, Brakeman, Bandit (language-specific) | Source code for vulnerabilities | `Security/SAST.gitlab-ci.yml` |
| **DAST** | OWASP ZAP | Running application (HTTP) | `Security/DAST.gitlab-ci.yml` |
| **Container Scanning** | Trivy (default), Grype | Docker image OS packages + app deps | `Security/Container-Scanning.gitlab-ci.yml` |
| **Dependency Scanning** | Gemnasium | Library/package vulnerabilities (SCA) | `Security/Dependency-Scanning.gitlab-ci.yml` |
| **Secret Detection** | GitLeaks + custom rules | Hardcoded secrets in code/history | `Security/Secret-Detection.gitlab-ci.yml` |
| **License Compliance** | LicenseFinder | Open source license violations | `Security/License-Scanning.gitlab-ci.yml` |
| **IaC Scanning** | KICS | Terraform, K8s manifests, Dockerfiles | `Security/IaC-Scanning.gitlab-ci.yml` |
| **API Security** | GitLab DAST API | OpenAPI/REST/GraphQL APIs | `DAST-API.gitlab-ci.yml` |
| **Fuzz Testing** | libFuzzer/AFL | Code paths with unexpected input | `Coverage-Fuzzing.gitlab-ci.yml` |

**Basic integration:**
```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

# All scanners output reports to artifacts:
# gl-sast-report.json, gl-dependency-scanning-report.json, etc.
# GitLab parses these and shows findings in MR widget + Security Dashboard
```

---

### Q24. Walk through how SAST works in GitLab. How does it detect vulnerabilities without running code?
**Answer:**

**SAST (Static Application Security Testing)** analyzes source code without executing it.

**How GitLab SAST works:**
1. GitLab detects the language(s) in your repo
2. Selects the appropriate analyzer (Semgrep is the primary; language-specific analyzers for others)
3. Runs the analyzer container against your source code
4. Analyzer produces `gl-sast-report.json` as a job artifact
5. GitLab parses the report → shows in MR widget, Security Dashboard, Vulnerability Management

```yaml
# Include and customize SAST
include:
  - template: Security/SAST.gitlab-ci.yml

variables:
  SAST_EXCLUDED_PATHS: "spec,test,tests,tmp,node_modules"
  SAST_EXCLUDED_ANALYZERS: "eslint"      # disable specific analyzers
  SEMGREP_TIMEOUT: "300"
  SAST_SEVERITY_LEVEL: "medium"         # only fail on medium+ severity
```

**SAST findings example in MR:**
```
Security scanning detected 2 vulnerabilities
  ● Critical: SQL Injection in app/models/user.rb:45
    User-controlled input passed directly to SQL query
    Fix: Use parameterized queries or ORM methods
  ● High: Hardcoded credential in config/database.yml:12
    Password appears to be hardcoded
```

**Customizing SAST rules with Semgrep:**
```yaml
# .gitlab/sast-ruleset.toml
[semgrep]
  [[semgrep.passthrough]]
    type = "url"
    value = "https://semgrep.dev/c/p/gitlab"   # use GitLab-managed ruleset

# Or point to custom rules repo:
  [[semgrep.passthrough]]
    type = "git"
    value = "https://gitlab.com/my-org/sast-rules.git"
    ref = "main"
```

---

### Q25. How does DAST work and what are its operational requirements?
**Answer:**

**DAST (Dynamic Application Security Testing)** tests a **running application** by sending HTTP requests and analyzing responses — finds vulnerabilities that SAST can't (e.g., auth bypass, session issues, server misconfiguration).

**How it works:**
1. Your deploy-to-review or deploy-to-staging job creates a live app
2. DAST job runs OWASP ZAP against the live URL
3. ZAP crawls the application, sends attack payloads (SQLi, XSS, CSRF, path traversal, etc.)
4. Produces `gl-dast-report.json`

**Requirements:**
- A running instance of the app (review app or staging URL)
- `DAST_WEBSITE` variable pointing to it
- App must be accessible from the runner (same network or public)

```yaml
include:
  - template: Security/DAST.gitlab-ci.yml

stages: [build, test, deploy-review, dast]

dast:
  stage: dast
  variables:
    DAST_WEBSITE: https://$CI_ENVIRONMENT_SLUG.review.myapp.com
    DAST_FULL_SCAN_ENABLED: "true"     # passive + active scan (default: passive only)
    DAST_ZAP_USE_AJAX_SPIDER: "true"   # for JS-heavy SPAs
    DAST_BROWSER_SCAN: "true"          # browser-based scan for modern apps
  needs:
    - job: deploy-review
      artifacts: false
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: verify
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**DAST authenticated scanning:**
```yaml
variables:
  DAST_AUTH_URL: https://review.myapp.com/login
  DAST_USERNAME: dast-test-user
  DAST_PASSWORD: $DAST_TEST_PASSWORD      # CI variable
  DAST_USERNAME_FIELD: "input[name=email]"
  DAST_PASSWORD_FIELD: "input[name=password]"
  DAST_SUBMIT_FIELD: "button[type=submit]"
```

---

### Q26. What is Container Scanning and how do you reduce false positives?
**Answer:**

**Container Scanning** scans your built Docker image for CVEs in:
- OS packages (Ubuntu/Alpine/RHEL packages)
- Application dependencies (node_modules, Python packages installed in image)

```yaml
include:
  - template: Security/Container-Scanning.gitlab-ci.yml

variables:
  CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA   # image to scan
  CS_SEVERITY_THRESHOLD: MEDIUM                   # skip LOW findings
  CS_ANALYZER: trivy                              # trivy (default) or grype

# Scan runs AFTER the image is built and pushed to registry
container_scanning:
  needs: [build-and-push-image]
```

**Reducing false positives — key strategies:**

**1. Use minimal base images:**
```dockerfile
# Bad: ubuntu:22.04 → 200+ packages, many with CVEs
FROM ubuntu:22.04

# Better: distroless or Alpine
FROM gcr.io/distroless/python3-debian12
# or
FROM python:3.12-alpine
# Alpine: ~20 packages vs ~200 in Ubuntu
```

**2. Keep base images updated:**
```yaml
# Schedule weekly rebuild of base images to get OS patches
rebuild-base:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - docker build --no-cache -f Dockerfile.base -t $CI_REGISTRY_IMAGE/base:latest .
    - docker push $CI_REGISTRY_IMAGE/base:latest
```

**3. Use vulnerability exceptions (`.gitlab-ci-local-policies.yml`):**
```yaml
# Mark accepted/false-positive vulnerabilities
vulnerability_allowlist:
  - CVE-2021-12345    # accepted risk, no fix available, expires: 2025-12-31
```

**4. Use multi-stage builds to exclude dev tools:**
```dockerfile
FROM python:3.12 AS builder
RUN pip install -r requirements.txt

FROM python:3.12-slim AS runtime    # clean runtime image, no build tools
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY app/ /app/
```

---

### Q27. What are Scan Execution Policies and Merge Request Approval Policies?
**Answer:**

These are **enterprise-grade policy controls** managed by the security team, enforced across all projects in a group — developers cannot bypass them.

**Scan Execution Policies** (enforce that scans RUN):
```yaml
# .gitlab/security-policies/policy.yml
---
scan_execution_policy:
  - name: require-sast-on-all-merge-requests
    description: SAST must run on every MR
    enabled: true
    rules:
      - type: pipeline
        branches: ["*"]
    actions:
      - scan: sast
      - scan: secret_detection

  - name: container-scan-on-protected-branches
    enabled: true
    rules:
      - type: pipeline
        branches: [main, release/*]
    actions:
      - scan: container_scanning
        variables:
          CS_SEVERITY_THRESHOLD: LOW    # stricter on production branches
```

**Merge Request Approval Policies** (block merge based on findings):
```yaml
approval_policy:
  - name: block-critical-vulnerabilities
    description: Security team must approve if critical vulns found
    enabled: true
    rules:
      - type: any_merge_request
        branch_type: protected
    approval_settings:
      block_branch_modification: true
      prevent_pushing_and_force_pushing: true
    actions:
      - type: require_approval
        approvals_required: 2
        role: security_approver
        scanners: [sast, container_scanning]
        vulnerabilities_allowed: 0
        severity_levels: [critical, high]
        vulnerability_states: [newly_detected]
```

**Key point**: These policies are stored in a **security policy project** linked to the group. The security team owns this project. Developers in individual projects cannot remove or bypass these policies.

---

## 5. Kubernetes Integration

### Q28. Explain the GitLab Agent for Kubernetes (agentk/kas). How does it work and why is it better than the legacy certificate-based integration?
**Answer:**

**Legacy certificate-based (deprecated):**
- You upload a kubeconfig/certificate to GitLab
- GitLab stores your cluster credentials
- Pipeline jobs call `kubectl` with those credentials
- Problem: long-lived credentials stored in GitLab, wide blast radius if leaked, GitLab needs to reach your cluster API

**GitLab Agent for Kubernetes (current):**
```
Your Kubernetes cluster:
  └── agentk Pod (installed via Helm)
        └── Connects OUT to kas (GitLab server component)
              └── Bidirectional gRPC over WebSocket

Flow:
  agentk polls kas for changes to .gitlab/agents/<name>/config.yaml
  → agentk applies manifests from Git directly to cluster
  → No inbound ports needed on cluster
  → Short-lived, just-in-time credentials (no long-lived kubeconfig in GitLab)
```

**Install agent:**
```bash
# 1. Register agent in GitLab: Infrastructure → Kubernetes clusters → Connect cluster
# 2. Copy generated helm command:
helm repo add gitlab https://charts.gitlab.io
helm upgrade --install gitlab-agent gitlab/gitlab-agent \
  --namespace gitlab-agent \
  --create-namespace \
  --set config.token=<TOKEN> \
  --set config.kasAddress="wss://kas.gitlab.com"
```

**Agent config for GitOps (pull-based):**
```yaml
# .gitlab/agents/prod-agent/config.yaml
gitops:
  manifest_projects:
    - id: my-org/kubernetes-manifests
      default_namespace: production
      paths:
        - glob: 'environments/production/**/*.yaml'
      reconcile_timeout: 3600s
      dry_run_strategy: none        # or: client, server
      prune: true                   # delete resources removed from Git
      prune_propagation_policy: foreground
```

**Agent config for CI/CD access (pipeline kubectl):**
```yaml
# .gitlab/agents/staging-agent/config.yaml
ci_access:
  projects:
    - id: my-org/my-app             # which projects can use this agent
  groups:
    - id: my-org/platform-team      # which groups can use this agent
```

---

### Q29. How do you deploy to Kubernetes from a GitLab pipeline using the agent?
**Answer:**

**Method 1: `kubectl` in pipeline (push-based via agent)**
```yaml
deploy-to-k8s:
  image: bitnami/kubectl:latest
  stage: deploy
  environment:
    name: production
    kubernetes:
      namespace: production
  script:
    # The agent creates a kubeconfig automatically in the job
    - kubectl config use-context $KUBE_CONTEXT
    - kubectl set image deployment/myapp app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/myapp -n production --timeout=300s
  variables:
    KUBE_CONTEXT: my-org/my-app:prod-agent    # project:agent-name
```

**Method 2: Helm deploy**
```yaml
deploy-helm:
  image: alpine/helm:3.14.0
  stage: deploy
  resource_group: production
  environment:
    name: production
    url: https://myapp.com
  script:
    - kubectl config use-context $KUBE_CONTEXT
    - helm upgrade --install myapp ./helm/myapp
        --namespace production
        --set image.repository=$CI_REGISTRY_IMAGE
        --set image.tag=$CI_COMMIT_SHA
        --set replicaCount=3
        --wait --timeout 5m
        --atomic                       # auto-rollback if deploy fails
  variables:
    KUBE_CONTEXT: my-org/my-app:prod-agent
```

**Method 3: GitOps (pull-based, zero-touch)**
```yaml
# Pipeline just updates the manifest in Git
# agentk detects change and applies it automatically
update-manifest:
  stage: deploy
  script:
    - git clone https://gitlab-ci-token:$CI_JOB_TOKEN@gitlab.com/my-org/kubernetes-manifests.git
    - cd kubernetes-manifests
    - sed -i "s|image:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" environments/production/deployment.yaml
    - git config user.email "ci@gitlab.com"
    - git config user.name "GitLab CI"
    - git commit -am "Update image to $CI_COMMIT_SHA [skip ci]"
    - git push origin main
  # agentk picks up the change within seconds and applies it
```

---

### Q30. How do you implement review apps on Kubernetes?
**Answer:**

Review apps create an ephemeral deployment per MR — gives reviewers a live environment to test before merge.

```yaml
deploy-review:
  stage: deploy
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.review.myapp.com
    on_stop: stop-review
    auto_stop_in: 3 days
  script:
    - kubectl config use-context $KUBE_CONTEXT
    # Create namespace for this review app
    - kubectl create namespace review-$CI_ENVIRONMENT_SLUG --dry-run=client -o yaml | kubectl apply -f -
    # Deploy with review-specific values
    - helm upgrade --install myapp-$CI_ENVIRONMENT_SLUG ./helm/myapp
        --namespace review-$CI_ENVIRONMENT_SLUG
        --set image.tag=$CI_COMMIT_SHA
        --set ingress.host=$CI_ENVIRONMENT_SLUG.review.myapp.com
        --set replicaCount=1
        --set resources.requests.cpu=100m
        --set resources.requests.memory=128Mi
        --wait --timeout 3m
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    KUBE_CONTEXT: my-org/my-app:staging-agent

stop-review:
  stage: deploy
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
  script:
    - helm uninstall myapp-$CI_ENVIRONMENT_SLUG --namespace review-$CI_ENVIRONMENT_SLUG
    - kubectl delete namespace review-$CI_ENVIRONMENT_SLUG
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
  variables:
    KUBE_CONTEXT: my-org/my-app:staging-agent
```

**Cost control for review apps:**
- `auto_stop_in: 3 days` — GitLab automatically runs the `on_stop` job
- Use small resource requests (1 replica, minimal CPU/memory)
- Use a separate node pool for review namespaces (spot instances)
- Cleanup job in scheduled pipeline:
```yaml
# Runs nightly: delete review namespaces older than 7 days
cleanup-stale-reviews:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - kubectl get ns -l type=review --no-headers | while read ns _; do
        created=$(kubectl get ns $ns -o jsonpath='{.metadata.creationTimestamp}')
        # delete if older than 7 days
      done
```

---

## 6. GitLab Flow & Branching

### Q31. Explain GitLab Flow. How does it compare to GitFlow and GitHub Flow?
**Answer:**

**GitHub Flow** (simplest):
```
feature/* → main (production)
```
One long-lived branch. Merge = deploy. Good for SaaS with continuous deployment.

**GitFlow** (complex):
```
feature/* → develop → release/* → main (+ hotfix/*)
```
Multiple long-lived branches. Good for versioned software releases (mobile apps, libraries).

**GitLab Flow** (pragmatic middle ground):
```
feature/* → main → pre-production → production
```
Or with environment branches:
```
feature/* → main (CI tests) → staging (auto-deploy) → production (manual)
```

**GitLab Flow principles:**
1. `main` is always deployable
2. Use **environment branches** (`staging`, `production`) to track deployment state
3. Code only flows in one direction: `feature → main → staging → production`
4. Hotfixes: cherry-pick to production branch (don't merge back through full flow)
5. MR required for all merges to protected branches

**Recommended setup for a SaaS team:**
```
Branch protection:
  main       : protected, ≥1 approval, passing pipeline required
  staging    : protected, auto-deploys from main via CI
  production : protected, manual deploy, ≥2 approvals

Feature work:
  1. Create branch: feature/TICKET-123-description
  2. Push → MR pipeline runs (build, test, SAST, review app)
  3. Code review + approval
  4. Merge to main → auto-deploy to staging
  5. Staging validation → manual promote to production
```

---

### Q32. What are protected branches and protected tags? How do you configure them?
**Answer:**

**Protected branches** restrict who can push to or merge into a branch.

**Configuration** (Project → Settings → Repository → Protected branches):
```
Branch: main
  Allowed to merge:   Maintainers, Developers (or specific groups)
  Allowed to push:    No one  (force push via MR only)
  Allowed to force push: false
  Code owner approval required: true
  Require MR: true
```

**Protected tags** restrict who can create tags matching a pattern:
```
Tag pattern: v*        (e.g., v1.2.3)
  Allowed to create: Maintainers only

Tag pattern: release/* (e.g., release/2025-Q1)
  Allowed to create: Maintainers + specific group
```

**Why protected tags matter for CI/CD:**
- Jobs can be configured to run ONLY on protected tags:
  ```yaml
  publish-to-pypi:
    rules:
      - if: $CI_COMMIT_TAG && $CI_COMMIT_REF_PROTECTED == "true"
  ```
- Prevents unauthorized deployments from arbitrary tags
- Combined with protected runners: only protected pipelines run on production runners with production secrets

**Code Owners (CODEOWNERS file):**
```
# CODEOWNERS
/infrastructure/    @platform-team
/security/          @security-team
*.tf                @infrastructure-team
/src/payments/      @payments-lead @payments-team
```
When code owner approval is required, the relevant team must approve any MR touching their files.

---

## 7. Scenario-Based Questions

### Q33. Scenario: Your pipeline runs in 45 minutes. The team wants it under 10 minutes. Walk through your optimization strategy.
**Answer:**

**Step 1: Profile — find the bottlenecks**
```yaml
# Enable job timing visibility
# GitLab CI: Pipeline → View details → Each job shows timing
# Identify: which stage/job takes longest?
```

**Typical findings and solutions:**

| Finding | Solution | Expected speedup |
|---|---|---|
| `npm install` takes 8 min | Cache `node_modules/` keyed on `package-lock.json` | 8 min → 30 sec |
| Tests run serially (1 thread) | `parallel: 4` with pytest-split | 20 min → 5 min |
| Docker build from scratch | Cache Docker layers; use BuildKit `--cache-from` | 10 min → 2 min |
| Stage 3 waits for all of Stage 2 | Use `needs:` DAG — start deploy-staging as soon as build+unit-test pass | Eliminates wait |
| Full test suite on every commit | `rules: changes:` — skip backend tests if only frontend changed | 15 min → 0 min |
| Pulling large images every run | `pull_policy: if-not-present` on runner | 2 min → 10 sec |

**Optimization with `needs:` + parallelism:**
```yaml
# BEFORE (serial stages): 45 min
# build(5) → [unit(10), lint(8), security(12)] → [integration(15)] → deploy(5)
# Total: 5 + 12 + 15 + 5 = 37 min minimum

# AFTER (DAG + parallel):
lint:   needs: []        # starts at t=0
sast:   needs: []        # starts at t=0 (parallel with lint)
build:  needs: []        # starts at t=0
unit:   needs: [build]   # starts at t=5 (right after build)
integration:
  needs: [build]         # starts at t=5 (parallel with unit)
  parallel: 3            # 15 min split into 5 min across 3 runners
deploy-staging:
  needs: [unit, integration, lint, sast]  # starts when all complete
# Total: 5 (build) + 5 (parallel unit+integration) + 2 (deploy) = ~12 min
```

---

### Q34. Scenario: A critical vulnerability is found in production. Your container image has a CVE in a base OS package. Walk through your response and prevention strategy.
**Answer:**

**Immediate response:**

```bash
# 1. Identify scope: which images are affected?
# GitLab Security Dashboard → Container Scanning → filter by CVE ID
# or: trivy image --severity CRITICAL $CI_REGISTRY_IMAGE:latest

# 2. Check if exploitable
# CVE in a library your app doesn't use? → lower priority
# CVE in network-facing code? → immediate action

# 3. Emergency patch pipeline
git checkout -b hotfix/CVE-2025-XXXXX main
# Update base image in Dockerfile
sed -i 's/FROM ubuntu:22.04/FROM ubuntu:22.04-20250401/g' Dockerfile
# Or pin to patched minor version
git commit -am "fix: update base image to patch CVE-2025-XXXXX"
git push origin hotfix/CVE-2025-XXXXX
# Create MR → emergency approval → merge → pipeline rebuilds + redeploys
```

**Prevention strategy — defense in depth:**

```yaml
# 1. Scheduled weekly base image rebuild (pick up OS patches automatically)
weekly-rebuild:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - docker build --no-cache -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:latest

# 2. Container scanning on EVERY MR pipeline
include:
  - template: Security/Container-Scanning.gitlab-ci.yml
variables:
  CS_SEVERITY_THRESHOLD: HIGH     # fail pipeline on HIGH+ CVEs

# 3. Scan Execution Policy: enforce scanning even if developer removes it

# 4. MR approval policy: block merge if critical CVE found
# (configured in security policy project)

# 5. Use minimal base images → smaller attack surface
# Distroless: 0 shell, 0 package manager → dramatically fewer CVEs
FROM gcr.io/distroless/java21-debian12

# 6. Automated dependency updates
# Use GitLab's Renovate integration or Dependabot-style tool
# Creates MRs automatically when new base image versions available
```

---

### Q35. Scenario: You need to design a GitLab CI/CD pipeline for a microservices monorepo with 8 services. Each service should only build/test when its code changes. How do you design this?
**Answer:**

**Challenge**: Avoid building all 8 services on every commit — wasteful and slow.

**Solution: `rules: changes:` + parent-child pipelines**

```yaml
# .gitlab-ci.yml (root — parent pipeline)
stages: [detect-changes, trigger-services]

# Each service triggered only when its files change
trigger-user-service:
  stage: trigger-services
  trigger:
    include: services/user-service/.gitlab-ci.yml
    strategy: depend
  rules:
    - changes:
        - services/user-service/**/*
        - shared/libs/**/*          # also trigger if shared lib changes
        - .gitlab-ci.yml

trigger-order-service:
  stage: trigger-services
  trigger:
    include: services/order-service/.gitlab-ci.yml
    strategy: depend
  rules:
    - changes:
        - services/order-service/**/*
        - shared/libs/**/*
```

**Each service's CI file:**
```yaml
# services/user-service/.gitlab-ci.yml
variables:
  SERVICE: user-service
  IMAGE: $CI_REGISTRY_IMAGE/user-service

build:
  stage: build
  script:
    - docker build -t $IMAGE:$CI_COMMIT_SHA services/user-service/
    - docker push $IMAGE:$CI_COMMIT_SHA

test:
  stage: test
  script:
    - cd services/user-service && go test ./...
```

**Dynamic pipeline approach (more advanced):**
```python
# generate-pipeline.py: detects changed services and generates YAML
import subprocess, yaml, sys

changed = subprocess.check_output(
    ["git", "diff", "--name-only", "origin/main...HEAD"]
).decode().splitlines()

services_to_build = set()
for path in changed:
    if path.startswith("services/"):
        service = path.split("/")[1]
        services_to_build.add(service)
    if path.startswith("shared/"):
        services_to_build = {"all"}  # rebuild everything if shared lib changed
        break

pipeline = {}
for service in services_to_build:
    pipeline[f"build-{service}"] = {
        "stage": "build",
        "script": [f"make build-{service}"]
    }

print(yaml.dump(pipeline))
```

```yaml
# .gitlab-ci.yml
generate:
  stage: .pre
  script: python generate-pipeline.py > generated.yml
  artifacts:
    paths: [generated.yml]

trigger-generated:
  trigger:
    include:
      - artifact: generated.yml
        job: generate
```

---

### Q36. Scenario: A developer accidentally committed AWS credentials to the repo. How does GitLab help and what's your incident response?
**Answer:**

**GitLab's automatic detection:**
- **Secret Detection** scanner runs in the pipeline and scans git history (not just latest commit)
- Detects AWS access keys, API tokens, private keys, connection strings
- Creates a vulnerability finding: "Secret detected: AWS access key in src/config.py:23"

**Immediate incident response:**

```bash
# 1. INVALIDATE the credential immediately (before anything else)
# AWS: Go to IAM → User → Security credentials → Delete the access key
aws iam delete-access-key --access-key-id AKIAIOSFODNN7EXAMPLE

# 2. Check CloudTrail for unauthorized usage
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time 2025-01-01T00:00:00Z

# 3. Remove from git history (git history rewrite)
# Use git-filter-repo (preferred over BFG):
pip install git-filter-repo
git filter-repo --path src/config.py --invert-paths  # remove file entirely
# or: redact the specific line:
git filter-repo --replace-text expressions.txt

# 4. Force-push to GitLab (requires Maintainer + unprotect branch temporarily)
git push origin main --force
# Re-protect the branch

# 5. Notify security team, create incident ticket
```

**Prevention going forward:**
```yaml
# 1. Enforce Secret Detection on ALL MRs via Scan Execution Policy
# 2. Add pre-commit hook locally:
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets

# 3. Use GitLab CI/CD variables or Vault — never hardcode credentials
# 4. Configure GitLab push rules to block commits with secrets:
#    Project → Settings → Repository → Push rules:
#    "Prevent secrets" checkbox
```

---

### Q37. Scenario: How do you implement a GitOps deployment model where infrastructure changes require approval and are fully auditable?
**Answer:**

**Architecture:**
```
Developer → feature branch → MR → approval → merge to main
  → GitLab pipeline:
      1. Terraform plan (show what will change)
      2. Manual approval step (human reviews plan output)
      3. Terraform apply (deploy on approval)
  → All changes tracked in Git history = full audit trail
  → GitLab audit events log: who approved, when, what pipeline ran
```

**Pipeline implementation:**
```yaml
# .gitlab-ci.yml (infrastructure repo)
stages: [validate, plan, apply]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform/environments/$ENVIRONMENT
  TF_STATE_NAME: ${ENVIRONMENT}

.terraform-template:
  image: hashicorp/terraform:1.7
  before_script:
    - terraform -chdir="$TF_ROOT" init
        -backend-config="address=${GITLAB_TF_ADDRESS}"
        -backend-config="lock_address=${GITLAB_TF_ADDRESS}/lock"
        -backend-config="username=gitlab-ci-token"
        -backend-config="password=${CI_JOB_TOKEN}"

validate:
  extends: .terraform-template
  stage: validate
  script:
    - terraform -chdir="$TF_ROOT" validate
    - terraform -chdir="$TF_ROOT" fmt -check -recursive

plan:
  extends: .terraform-template
  stage: plan
  script:
    - terraform -chdir="$TF_ROOT" plan -out=tfplan
    - terraform -chdir="$TF_ROOT" show -no-color tfplan > plan.txt
    - cat plan.txt
  artifacts:
    paths: [terraform/environments/$ENVIRONMENT/tfplan, plan.txt]
    expire_in: 1 day
    reports:
      terraform: plan.txt     # shown in MR widget
  resource_group: terraform-$ENVIRONMENT

apply:
  extends: .terraform-template
  stage: apply
  when: manual                 # requires human approval
  allow_failure: false
  environment:
    name: infrastructure/$ENVIRONMENT
  resource_group: terraform-$ENVIRONMENT
  script:
    - terraform -chdir="$TF_ROOT" apply -auto-approve tfplan
  needs:
    - job: plan
      artifacts: true
  rules:
    - if: $CI_COMMIT_BRANCH == "main"   # only apply from main
```

**Auditability:**
- Every `apply` is tied to a specific Git commit, MR, and approver
- GitLab deployment history shows: who ran apply, when, which pipeline, which commit
- Terraform state stored in GitLab-managed backend (HTTP backend) — versioned
- GitLab audit log: MR approvals, pipeline triggers, environment deploys

---

### Q38. Scenario: How do you handle a situation where your pipeline passes but the application crashes in production (works in CI, fails in prod)?
**Answer:**

**Root causes and systematic investigation:**

**1. Environment parity issues**
```yaml
# Problem: CI uses different image/versions than production
# Solution: Pin exact versions everywhere
variables:
  PYTHON_VERSION: "3.12.2"    # exact, not "3.12" or "3"
  POSTGRES_VERSION: "16.2"

services:
  - name: postgres:$POSTGRES_VERSION
    alias: postgres
    variables:
      POSTGRES_DB: testdb
      POSTGRES_PASSWORD: testpass
```

**2. Missing environment variables in production**
```yaml
# Add env var validation to pipeline smoke test
post-deploy-validation:
  stage: deploy
  script:
    - kubectl exec deployment/myapp -- python -c "
        import os, sys
        required = ['DATABASE_URL', 'REDIS_URL', 'SECRET_KEY', 'S3_BUCKET']
        missing = [v for v in required if not os.environ.get(v)]
        if missing:
            print(f'MISSING ENV VARS: {missing}')
            sys.exit(1)
        print('All required env vars present')
      "
```

**3. Database migration not run / schema mismatch**
```yaml
migrate:
  stage: pre-deploy
  script:
    - kubectl exec deployment/myapp -- python manage.py migrate --check
    - kubectl exec deployment/myapp -- python manage.py migrate
  resource_group: production     # prevent parallel migrations
```

**4. Integration test against real dependencies**
```yaml
# CI often mocks external services; production uses real ones
# Add integration stage that hits actual staging endpoints:
integration-staging:
  stage: post-deploy
  script:
    - pytest tests/integration/ --env=staging
      --base-url=https://staging.myapp.com
  environment:
    name: staging
    action: verify
```

**5. Production observability: detect crash faster**
```yaml
# After deploy: monitor error rate for 5 minutes
smoke-test-production:
  stage: post-deploy
  script:
    - sleep 300    # wait 5 min
    - ERROR_RATE=$(curl -s https://prometheus.myapp.com/api/v1/query
        --data 'query=rate(http_requests_total{status=~"5.."}[5m])')
    - python check_error_rate.py "$ERROR_RATE" --threshold 0.01
  when: on_success
  allow_failure: true    # don't block but alert
```

---

## Quick Reference: Key GitLab CI Commands & Patterns

```bash
# Validate .gitlab-ci.yml locally
docker run --rm -v $(pwd):/src -w /src \
  gitlab/gitlab-runner:latest \
  exec docker --docker-image alpine:latest my-job-name

# Lint .gitlab-ci.yml via API
curl -X POST "https://gitlab.com/api/v4/projects/$PROJECT_ID/ci/lint" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"content\": \"$(cat .gitlab-ci.yml | base64)\"}"

# Trigger pipeline via API
curl -X POST "https://gitlab.com/api/v4/projects/$PROJECT_ID/trigger/pipeline" \
  -F "token=$TRIGGER_TOKEN" \
  -F "ref=main" \
  -F "variables[ENVIRONMENT]=production"

# Cancel all running pipelines for a branch
gitlab-ctl pipeline cancel --branch feature/my-branch
```

## Key Numbers & Defaults

| Item | Value |
|---|---|
| GitLab SaaS shared runner RAM | 8 GB (n2d-standard-2) |
| Free CI minutes/month | 400 |
| Default artifact expiry | 30 days |
| Default job timeout | 1 hour (60 min) |
| Max job timeout | 1 month |
| Max pipeline depth (parent-child) | 2 levels |
| Max `needs:` dependencies | 50 |
| Cache max size (shared runners) | 1 GB |
| DAST full scan timeout default | 1 hour |
| Runner poll interval | 3 seconds |
| Max parallel jobs per runner | `concurrent` in config.toml |

---

*End of catalog — 38 Q&A covering GitLab DevOps Engineering: CI/CD Fundamentals, Runners, Pipeline Patterns, Security Scanning, Kubernetes Integration, GitLab Flow, and Scenario-Based questions.*
