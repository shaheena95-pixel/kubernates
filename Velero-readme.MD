# Velero — Kubernetes Backup & Restore on EKS

> **Velero** is an open-source tool for backing up, restoring, and migrating Kubernetes cluster resources and persistent volumes.

---

## Table of Contents

- [What is Velero?](#what-is-velero)
- [Why Velero on EKS?](#why-velero-on-eks)
- [How Velero Works](#how-velero-works)
- [Architecture](#architecture)
- [What Gets Backed Up](#what-gets-backed-up)
- [What Velero Does NOT Back Up](#what-velero-does-not-back-up)
- [Installation](#installation)
- [Taking a Backup](#taking-a-backup)
- [Restoring from Backup](#restoring-from-backup)
- [Scheduled Backups](#scheduled-backups)
- [Real-World Use Case](#real-world-use-case)
- [Key Concept](#key-concept)

---

## What is Velero?

Velero lets you:

- **Back up** Kubernetes cluster resources and persistent volumes
- **Restore** Kubernetes objects after accidental deletion or cluster failure
- **Migrate** workloads across clusters
- **Perform disaster recovery** with minimal downtime

Works with: **EKS**, **GKE**, **AKS**, and self-managed clusters.

---

## Why Velero on EKS?

In EKS, AWS fully manages the control plane — meaning:

- You **cannot access etcd** directly
- AWS does **not automatically back up** your Kubernetes resources

Velero bridges this gap by backing up:

| Resource Type | Included |
|---|---|
| Deployments & Services | ✅ |
| ConfigMaps & Secrets | ✅ |
| Namespaces | ✅ |
| CRDs (Custom Resource Definitions) | ✅ |
| RBAC policies | ✅ |
| Persistent Volume metadata | ✅ |
| PV data (via EBS snapshots) | ✅ *(if configured)* |

---

## How Velero Works

```
1. Install Velero in the cluster
2. Velero runs as a controller (Pod) inside Kubernetes
3. It watches cluster resources via the API server
4. On backup trigger → collects YAML definitions of all resources
5. Stores the backup in an S3 bucket
6. On restore → recreates all resources from stored YAMLs
```

> **Think of Velero as:** `kubectl export` + backup automation + EBS snapshot integration

---

## Architecture

```
Kubernetes Cluster
        │
   Velero Pod
        │
   AWS S3 Bucket          ← Kubernetes resource YAMLs stored here

   Persistent Volume
        │
   EBS Snapshot           ← PV data backed up here (optional)
```

**Key rule:**
> *Velero protects your workloads. AWS protects the control plane.*

---

## What Gets Backed Up

- All Kubernetes object definitions (YAMLs)
- Namespaces
- RBAC roles & bindings
- Secrets & ConfigMaps
- CRDs and CR instances
- Resource metadata
- Persistent Volume data *(via EBS snapshots, when configured)*

---

## What Velero Does NOT Back Up

- External databases (e.g., RDS)
- Container images (stored in ECR/DockerHub)
- EC2 instances
- ALB / Load Balancer infrastructure configs

Velero only backs up **Kubernetes resources** — not underlying AWS infrastructure.

---

## Installation

### Step 1 — Create an S3 Bucket

```bash
aws s3 mb s3://eks-backup-bucket
```

---

### Step 2 — Create an IAM Role for Velero

Attach a policy granting:
- `s3:*` on your backup bucket
- `ec2:*Snapshot*` for EBS volume snapshots

> **Recommended:** Use [IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) to bind the role to Velero's service account.

---

### Step 3 — Install the Velero CLI

```bash
# macOS
brew install velero

# Linux — download binary
curl -L https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz | tar xz
sudo mv velero /usr/local/bin/
```

---

### Step 4 — Install Velero in the Cluster

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket eks-backup-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1
```

This installs:
- The Velero **Deployment** in the `velero` namespace
- All required **CRDs** (`Backup`, `Restore`, `Schedule`, etc.)
- The **Backup controller** that processes backup/restore jobs

Verify installation:

```bash
kubectl get pods -n velero
velero version
```

---

## Taking a Backup

### Back up the entire cluster

```bash
velero backup create full-backup
```

### Back up a specific namespace

```bash
velero backup create prod-backup --include-namespaces production
```

### Back up multiple namespaces

```bash
velero backup create multi-ns-backup --include-namespaces production,staging
```

### Check backup status

```bash
velero backup get
velero backup describe full-backup
velero backup logs full-backup
```

---

## Restoring from Backup

### Restore a full backup

```bash
velero restore create --from-backup full-backup
```

### Restore a specific namespace backup

```bash
velero restore create --from-backup prod-backup
```

Velero will automatically:
- Recreate all Deployments, Services, ConfigMaps, Secrets
- Restore RBAC rules
- Restore Persistent Volumes (if EBS snapshots were taken)

### Check restore status

```bash
velero restore get
velero restore describe <restore-name>
```

---

## Scheduled Backups

Velero supports cron-style schedules for automated backups.

### Daily backup at 2 AM

```bash
velero schedule create daily-backup \
  --schedule "0 2 * * *"
```

### Daily backup of a specific namespace

```bash
velero schedule create daily-prod-backup \
  --schedule "0 2 * * *" \
  --include-namespaces production
```

### List all schedules

```bash
velero schedule get
```

---

## Real-World Use Case

**Scenario:** A developer accidentally runs `kubectl delete namespace production`.

| Without Velero | With Velero |
|---|---|
| Manual recreation of all resources | `velero restore create --from-backup prod-backup` |
| Hours of downtime | Restored in minutes |
| Potential permanent data loss | Full recovery |

---

## Key Concept

> Velero does **not** copy or snapshot etcd.
>
> It uses the **Kubernetes API server** to export resource definitions, stores them in S3, and uses the same API to recreate them on restore.
>
> This makes it cluster-agnostic — you can restore to a completely different cluster.

---

## References

- [Velero Official Docs](https://velero.io/docs/)
- [velero-plugin-for-aws](https://github.com/vmware-tanzu/velero-plugin-for-aws)
- [AWS IRSA Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
