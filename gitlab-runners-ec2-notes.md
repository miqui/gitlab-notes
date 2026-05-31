# GitLab Runner on EC2: Complete Setup Guide

## Overview

This guide covers how to set up an Amazon EC2 instance as a GitLab CI/CD Runner, including installation, registration, caching strategies, and best practices. A GitLab Runner is an agent that picks up CI/CD jobs from GitLab and executes them on the host machine.

***

## Part 1: Setting Up EC2 as a GitLab Runner

### Step 1: Launch & Access Your EC2 Instance

Spin up an EC2 instance (Ubuntu or Amazon Linux 2023 recommended) and SSH into it[1]. Recommended specs:

- **OS**: Ubuntu 22.04 LTS or Amazon Linux 2023
- **Instance type**: `t3.medium` or larger for Docker workloads
- **Disk**: At least 30 GB to handle Docker image layers and cache[1]

```bash
ssh -i your-key.pem ec2-user@<your-ec2-public-ip>
```

### Step 2: Install GitLab Runner

**Amazon Linux 2023:**
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
sudo yum install -y gitlab-runner
gitlab-runner --version
```

**Ubuntu:**
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner
```



### Step 3: Install Docker (Recommended Executor)

Docker is the recommended executor for clean, isolated CI jobs[2]:

```bash
# Amazon Linux 2023
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker gitlab-runner

# Ubuntu
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker gitlab-runner
```

### Step 4: Get the Registration Token

Navigate to GitLab to retrieve your runner token[2]:

- **Project-level**: `Project > Settings > CI/CD > Runners` → click **New project runner**
- **Group-level**: `Group > Settings > CI/CD > Runners`
- **Instance-wide**: `Admin Area > CI/CD > Runners`

Copy the **GitLab URL** and **registration token** displayed.

### Step 5: Register the Runner

Run this command on your EC2 instance, replacing the token and URL with your own[1]:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --token "YOUR_REGISTRATION_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "ec2-docker-runner" \
  --tag-list "docker,aws,ec2" \
  --run-untagged="true"
```

> **Tip:** For a simpler setup without Docker, use `--executor "shell"` and skip the Docker installation step[2].

### Step 6: Start & Verify

```bash
sudo gitlab-runner start
sudo gitlab-runner verify    # confirms connection to GitLab
sudo gitlab-runner list      # shows registered runners
```



### Step 7: Test with a Pipeline

Create a `.gitlab-ci.yml` at the root of your repository and match the tags used during registration[2]:

```yaml
test-job:
  tags:
    - docker
    - ec2
  script:
    - echo "Running on EC2 runner!"
    - docker --version
```

Push a commit and the pipeline will trigger on your EC2 instance.

***

## Part 2: Caching in GitLab CI/CD

### Why Caching Matters

Caching in GitLab CI/CD allows jobs to reuse previously downloaded dependencies and built assets across pipeline runs, dramatically reducing execution times[3]. A well-tuned cache can cut pipeline times by **50% or more**[4].

### When Caching Is Most Valuable

Caching is critical in these scenarios[3][4]:

- **Heavy dependency installs** — `node_modules`, Python `venv`, Maven `.m2`, Go module cache; package downloads are slow and repetitive
- **Ephemeral runners** — Docker executors and Kubernetes pods start fresh each job; without a cache, everything is re-downloaded from scratch[5]
- **Long-running pipelines** — If build + test cycles exceed ~5 minutes, caching brings compounding savings across many commits
- **Monorepos or multi-stage builds** — Multiple jobs sharing the same dependencies benefit from a shared cache in parallel stages[6]
- **Frequent CI runs on the same branch** — Feature branches that trigger pipelines on every push benefit greatly when the lockfile rarely changes between commits[7]

### Cache vs. Artifacts

| | **Cache** | **Artifacts** |
|---|---|---|
| Purpose | Dependencies, optional reuse | Build outputs passed between jobs |
| Availability | Best-effort, may be missing | Guaranteed if job succeeds |
| Scope | Shared across pipelines | Scoped to a single pipeline |
| Example use | `node_modules`, `.m2` | Compiled binaries, test reports |

[4]

### When Caching Is NOT Worth It

- **Generated files that change every build** — Caching them wastes upload time with no benefit[4]
- **Small, fast-to-install dependencies** — If `pip install` or `npm ci` takes under 30 seconds, the cache overhead may not be worth it[7]
- **Non-deterministic outputs** — Files that vary per run cause constant cache invalidation and churn

### Cache Configuration Examples

**Node.js — best practice using lockfile key:**
```yaml
cache:
  key:
    files:
      - package-lock.json    # invalidate only when deps change
  paths:
    - node_modules/
```

**Python:**
```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .pip-cache/

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"
```

**Maven (Java):**
```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository/
```



### S3-Backed Cache for EC2 Runners

For EC2-based runners, storing cache in S3 is the recommended approach for durability and sharing across runner instances[1]. Steps:

1. Attach an IAM role to your EC2 instance with the following permissions:
   - `s3:GetObject`
   - `s3:PutObject`
   - `s3:ListBucket`

2. Update `/etc/gitlab-runner/config.toml`:

```toml
[[runners]]
  [runners.cache]
    Type = "s3"
    [runners.cache.s3]
      BucketName = "my-gitlab-runner-cache"
      BucketLocation = "us-east-1"
```

This allows cache to be shared across multiple EC2 runner instances and persists between runner restarts[1].

***

## Summary of Key Commands

| Task | Command |
|------|---------|
| Install runner (Amazon Linux) | `sudo yum install -y gitlab-runner` |
| Install runner (Ubuntu) | `sudo apt install -y gitlab-runner` |
| Register runner | `sudo gitlab-runner register` |
| Start runner | `sudo gitlab-runner start` |
| Verify connection | `sudo gitlab-runner verify` |
| List runners | `sudo gitlab-runner list` |
| Check runner status | `sudo gitlab-runner status` |
| Restart runner | `sudo gitlab-runner restart` |

***

## Best Practices

- Use **project-level runners** instead of shared runners for sensitive projects to limit exposure[1]
- Use `cache:key:files` with your lockfile (e.g., `package-lock.json`, `requirements.txt`) so caches are only invalidated when dependencies actually change[7][4]
- Assign **tags** to your runner and use them in `.gitlab-ci.yml` to ensure jobs run on the correct runner[2]
- For production workloads, consider **Docker Machine autoscaling** on AWS to spin up runner VMs on demand and reduce costs[8]
- Monitor runner disk usage — Docker image layers and build caches can fill up 30 GB faster than expected; set up periodic `docker system prune` via cron
