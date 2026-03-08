# EKS Platform — Karpenter, VPA, Live Migration & Datadog

A production-grade AWS EKS platform demonstrating intelligent autoscaling, zero-downtime live pod migration, and full observability with Datadog.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐     ┌─────────────────────────┐
│              AWS EKS Cluster (v1.35)                    │     │        Datadog          │
│                                                         │     │                         │
│  ┌─────────────┐   ┌──────────────────┐                 │────▶│  CI Visibility          │
│  │  Karpenter  │   │  GitHub Actions  │                 │     │  Infrastructure         │
│  │  (nodes)    │   │  ARC Runners     │                 │────▶│  Autoscaling (DPA)      │
│  └─────────────┘   └──────────────────┘                 │     │  Logs                   │
│                                                         │────▶│                         │
│  ┌─────────────┐   ┌──────────────────┐                 │     └─────────────────────────┘
│  │    VPA      │   │ Live Migration   │                 │
│  │ (in-place)  │   │ CRIU open-source │                 │
│  └─────────────┘   └──────────────────┘                 │
│                                                         │
│              Pods / Workloads                           │
└─────────────────────────────────────────────────────────┘
```

### Two User Workflows

| | Option A — Manual Scaling | Option B — CI/CD Pipeline |
|---|---|---|
| **Trigger** | User adjusts replicas / resources | User submits GitHub Actions pipeline |
| **Autoscaling** | VPA right-sizes based on 2-week history | N/A |
| **Restart?** | No — K8s 1.35 in-place resize | N/A |
| **Observability** | Datadog Infra + Autoscaling | Datadog CI Visibility + Logs |

---

## Topics

### 1. Karpenter — Node Provisioning
[`karpenter/`](./karpenter)

Karpenter automatically provisions and deprovisions EC2 nodes based on workload demand. Supports Spot and On-Demand instances with automatic consolidation.

- `nodepool.yaml` — NodePool with Spot + On-Demand, auto-consolidation
- `ec2nodeclass.yaml` — EC2NodeClass for AL2023 nodes

### 2. VPA — In-Place Resource Resize
[`vpa/`](./vpa)

Vertical Pod Autoscaler recommends and applies CPU/memory adjustments based on historical usage (2-week window). On Kubernetes 1.35, VPA uses **in-place pod resize** — no pod restarts required.

- `vpa-load-test.yaml` — VPA for load-test workload
- `vpa-stock-backend.yaml` — VPA for stock-backend with min/max bounds

> **Key behaviour**: `resizePolicy: NotRequired` on pod spec = VPA applies changes live without restarting the pod.

### 3. Live Migration (CRIU)
[`live-migration/`](./live-migration)

Zero-downtime pod migration across nodes using **CRIU (Checkpoint/Restore In Userspace)** — fully open-source.

When a Spot instance termination is detected, the affected pod is:
1. **Checkpointed** — full RAM state saved to disk (~29MB image)
2. **Transferred** — checkpoint image moved to target node
3. **Restored** — pod continues execution on new node, exactly where it left off

**What is preserved:**
- Full RAM state (process memory, open file descriptors)
- Pod IP address (via Kube-OVN CNI annotation `ovn.kubernetes.io/ip_address`)
- Application state (counter app: 5127 → migrate → 5134, no gap)

**Files:**
- `01-namespace.yaml` — live-migration namespace
- `02-criu-node-setup-ds.yaml` — installs CRIU v3.17.1 on all nodes via DaemonSet
- `03-counter-app.yaml` — demo Flask counter app (increments every 1s)
- `04-migration-agent-ds.yaml` — privileged HTTP agent on every node (port 9090)
- `05-migration-orchestrator.yaml` — orchestrates the full checkpoint/restore workflow
- `08-stock-app.yaml` — Stock Trading App (migratable)
- `09-stock-migrate.yaml` — migration job for stock-backend
- `simulate-spot-kill.sh` — simulates AWS Spot kill and triggers live migration

### 4. GitHub Actions — CI/CD Pipelines
[`.github/workflows/`](./.github/workflows)

Self-hosted GitHub Actions runners run inside the EKS cluster via Actions Runner Controller (ARC).

**Workflows:**
- `load-test.yml` — deploys 10 parallel load-test jobs to K8s, monitors VPA in-place resize
- `stock-app-cicd.yml` — full 5-stage Stock Trading App pipeline:
  - Stage 1: Deploy & verify stock-app on EKS
  - Stage 2: Health + smoke tests (HTTP endpoints)
  - Stage 3: 5 parallel trader simulations (AAPL, GOOGL, TSLA HFT, market feed, portfolio)
  - Stage 4: VPA + Karpenter validation (NodeClaims, restart counts)
  - Stage 5: SRE report + email notification via AWS SES

### 5. Datadog Observability
[`datadog/`](./datadog)

| Product | What it monitors |
|---|---|
| **Infrastructure** | All 8 EKS nodes, pods, resource metrics |
| **Logs** | K8s node logs + GitHub Actions pipeline logs via Observability Pipeline Worker |
| **Autoscaling (DPA)** | ML-based scaling recommendations for stock-backend and load-test |
| **CI Visibility** | Full traces of every pipeline run, job, and step |

**Files:**
- `datadog-pod-autoscaler.yaml` — DatadogPodAutoscaler for stock-backend
- `ci-visibility-env.yaml` — CI Visibility ConfigMap for ARC runners
- `12-flex-logs-pipeline.yaml` — Observability Pipeline Worker (Vector → Flex Logs)
- `13-datadog-agent-opw-patch.yaml` — Agent patch to route logs through pipeline

---

## Cluster Details

| Property | Value |
|---|---|
| Cluster | `karpenter-demo` |
| Region | `us-east-1` |
| EKS Version | `1.35` |
| Nodes | 8x (Spot + On-Demand) |
| CNI | Kube-OVN v1.15.4 (IP preservation) |
| Karpenter | v1 (NodePool + EC2NodeClass) |
| VPA | cowboysysop helm chart |
| CRIU | v3.17.1 (open-source) |

---

## Prerequisites

- AWS CLI configured with EKS access
- `kubectl` connected to `karpenter-demo` cluster
- Datadog API key (set as `DD_API_KEY` secret in GitHub repo)
- AWS credentials (set as `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` secrets)

---

## Quick Start

```bash
# 1. Apply Karpenter node configuration
kubectl apply -f karpenter/

# 2. Deploy VPA resources
kubectl apply -f vpa/

# 3. Set up live migration
kubectl apply -f live-migration/01-namespace.yaml
kubectl apply -f live-migration/02-criu-node-setup-ds.yaml
kubectl apply -f live-migration/03-counter-app.yaml
kubectl apply -f live-migration/04-migration-agent-ds.yaml

# 4. Apply Datadog configuration
kubectl apply -f datadog/

# 5. Trigger CI pipeline
gh workflow run load-test.yml
```

---

## Live Migration Demo

```bash
# Simulate a Spot kill and watch live migration happen
cd live-migration
chmod +x simulate-spot-kill.sh
./simulate-spot-kill.sh
```

Watch the counter app continue counting with zero interruption across nodes.
