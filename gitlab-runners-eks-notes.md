# GitLab Runner on Amazon EKS: Complete Setup Guide

## Overview

This guide walks through deploying GitLab CI/CD Runners on Amazon EKS using the Kubernetes executor via Helm. Unlike EC2 runners where jobs run in Docker containers on a single VM, the Kubernetes executor spins up a **dedicated ephemeral pod per CI job** — fully isolated and automatically destroyed after the job completes.

***

## Prerequisites

Install the following tools locally before starting[1]:

- `aws cli` — authenticated with appropriate IAM permissions
- `eksctl` — for EKS cluster creation
- `kubectl` — for Kubernetes resource management
- `helm` — for installing the GitLab Runner chart
- `jq` — for JSON processing (optional but useful)
- A **runner authentication token** (`glrt-` prefix) from GitLab UI[2]

### Getting the Runner Authentication Token (GitLab 16.0+)

The legacy registration token is deprecated as of GitLab 16.0 and disabled in GitLab 17.0+[2]. Use the new workflow:

1. Go to `Project > Settings > CI/CD > Runners`
2. Click **New project runner**
3. Configure tags, description, and options in the UI
4. Copy the generated token — it will be prefixed with `glrt-`[2]

***

## Step 1: Create or Connect an EKS Cluster

If you don't have an existing cluster, create one with `eksctl`[1]:

```bash
eksctl create cluster \
  --name gitlab-runners \
  --region us-east-1 \
  --nodegroup-name runners-ng \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed
```

Then update your kubeconfig:

```bash
aws eks update-kubeconfig --name gitlab-runners --region us-east-1
kubectl get nodes   # verify cluster is accessible
```

***

## Step 2: Create a Namespace & RBAC

```bash
kubectl create namespace gitlab-runners

kubectl create serviceaccount gitlab-runner -n gitlab-runners

kubectl create clusterrolebinding gitlab-runner-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=gitlab-runners:gitlab-runner
```



> **Note:** For production, scope RBAC to least-privilege instead of `cluster-admin`.

***

## Step 3: Add the GitLab Helm Repository

```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
```

***

## Step 4: Configure `values.yaml`

Create a `values.yaml` file with your runner settings[1][3]:

```yaml
gitlabUrl: "https://gitlab.com/"
runnerToken: "glrt-YOUR_AUTH_TOKEN"

runners:
  executor: kubernetes
  tags: "k8s,eks,docker"
  namespace: gitlab-runners

rbac:
  create: true

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

***

## Step 5: Install the Runner via Helm

```bash
helm install gitlab-runner gitlab/gitlab-runner \
  -n gitlab-runners \
  -f values.yaml
```

Verify the runner pod is running:

```bash
kubectl get pods -n gitlab-runners
kubectl logs -n gitlab-runners deployment/gitlab-runner
```



To upgrade an existing installation after changing `values.yaml`:

```bash
helm upgrade gitlab-runner gitlab/gitlab-runner \
  -n gitlab-runners \
  -f values.yaml
```

***

## Step 6: Test with a Pipeline

Add a `.gitlab-ci.yml` to your repository root and match the tags set during runner creation[4]:

```yaml
build-job:
  tags:
    - k8s
    - eks
  script:
    - echo "Running inside EKS pod!"
    - kubectl version --client

test-job:
  tags:
    - k8s
  image: python:3.12-alpine
  script:
    - python --version
    - echo "Each job gets its own isolated pod"
```

Push a commit — GitLab will schedule each job as a new pod in your `gitlab-runners` namespace, which is automatically cleaned up after completion[1].

***

## Optional: Spot Instances for Cost Savings

Add a Spot instance node group to reduce compute costs by up to 70%[1][3]:

```bash
eksctl create nodegroup \
  --cluster gitlab-runners \
  --name spot-runners \
  --node-type t3.medium \
  --nodes-min 0 \
  --nodes-max 10 \
  --spot
```

Configure runner pods to prefer Spot nodes in `values.yaml`:

```yaml
runners:
  nodeSelector:
    eks.amazonaws.com/capacityType: SPOT
  tolerations:
    - key: "eks.amazonaws.com/capacityType"
      operator: "Equal"
      value: "SPOT"
      effect: "NoSchedule"
```

***

## Optional: EKS Fargate (Serverless Nodes)

For a fully serverless setup with zero node management, use EKS Fargate[5]. Jobs run on AWS-managed infrastructure and you pay only for the vCPU and memory used per job. Create a Fargate profile targeting the `gitlab-runners` namespace:

```bash
eksctl create fargateprofile \
  --cluster gitlab-runners \
  --name gitlab-fargate \
  --namespace gitlab-runners
```

***

## Optional: S3 Cache Configuration

Configure distributed caching backed by S3 so cache is shared across all runner pods[1]:

1. Attach an IAM role to your EKS node group (or use IRSA) with permissions:
   - `s3:GetObject`
   - `s3:PutObject`
   - `s3:ListBucket`

2. Add cache config to `values.yaml`:

```yaml
runners:
  cache:
    cacheType: s3
    s3BucketName: "my-gitlab-runner-cache"
    s3BucketLocation: "us-east-1"
    s3CachePath: "runner/cache"
    s3CacheInsecure: false
```

***

## EC2 Runner vs. EKS Runner

| | **EC2 Runner** | **EKS Runner** |
|---|---|---|
| Job isolation | Docker container per job | Dedicated K8s pod per job |
| Scaling | Manual / ASG | Auto via Kubernetes HPA/Karpenter |
| Setup complexity | Low | Medium |
| Cost control | Fixed EC2 cost | Scale-to-zero with Fargate/Spot |
| Concurrency | Limited by instance size | Scales horizontally across nodes |
| Best for | Simple pipelines, small teams | High-concurrency, production workloads |

[5][1]

***

## Quick Reference Commands

| Task | Command |
|------|---------|
| Add GitLab Helm repo | `helm repo add gitlab https://charts.gitlab.io` |
| Install runner | `helm install gitlab-runner gitlab/gitlab-runner -n gitlab-runners -f values.yaml` |
| Upgrade runner | `helm upgrade gitlab-runner gitlab/gitlab-runner -n gitlab-runners -f values.yaml` |
| Check runner pods | `kubectl get pods -n gitlab-runners` |
| View runner logs | `kubectl logs -n gitlab-runners deployment/gitlab-runner` |
| Uninstall runner | `helm uninstall gitlab-runner -n gitlab-runners` |
| List Helm releases | `helm list -n gitlab-runners` |

***

## Best Practices

- Use **project-level runners** with specific tags to prevent jobs from running on the wrong runner[1]
- Use `glrt-` prefixed **runner authentication tokens** — the legacy registration token is deprecated and disabled in GitLab 17.0+[2]
- For production, replace `cluster-admin` RBAC with a scoped role that only allows pod creation/deletion in the `gitlab-runners` namespace[3]
- Use **IRSA (IAM Roles for Service Accounts)** for S3 cache access instead of embedding AWS credentials[1]
- Set **resource requests and limits** in `values.yaml` to prevent runaway jobs from starving other workloads in the cluster[1]
- Consider **Karpenter** for intelligent node autoscaling that right-sizes nodes to the actual resource demands of CI jobs[3]
