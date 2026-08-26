# PostgreSQL-K8s

## Introduction

This repository contains the implementation of a home assignment for deploying PostgreSQL on a local Kubernetes environment using the Percona PostgreSQL Operator.

The objective was to deploy a functional PostgreSQL cluster, verify database connectivity, and document both the deployment process and the implementation decisions.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Minikube Installation](#minikube-installation)
4. [Architecture](#architecture)
5. [Deployment](#deployment)
6. [Verification](#verification)
7. [Design Decisions](#design-decisions)
8. [Limitations](#limitations)
9. [Future Improvements](#future-improvements)
10. [Summary](#summary)

---

# Overview

This project demonstrates how to deploy a PostgreSQL cluster on a local Kubernetes environment using the **Percona PostgreSQL Operator**.

The deployment includes:

- Installing a local Kubernetes cluster with Minikube
- Installing the Percona PostgreSQL Operator
- Deploying a PostgreSQL cluster
- Verifying cluster health
- Connecting to PostgreSQL
- Executing a SQL query to validate the deployment

### Tested with

- Kubernetes v1.30
- Minikube v1.34
- Helm v3
- PostgreSQL 18.4

---

# Prerequisites

## Required Software

- kubectl 1.24+
- Helm 3+
- Minikube
- A supported Minikube driver (for example Docker)
- PostgreSQL client (`psql`)

## Verify Installation

```bash
kubectl version --short
helm version
minikube version
```

All commands should complete successfully.

---

## macOS Installation

```bash
brew install kubectl helm minikube docker
```

Install PostgreSQL client:

```bash
brew install libpq
brew link --force libpq
```

---

## Ubuntu / Debian Installation

```bash
sudo apt-get update
sudo apt-get install -y curl

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Minikube
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Docker
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER

# PostgreSQL client
sudo apt-get install -y postgresql-client
```

---

# Minikube Installation

## Recommended Local Environment

The following resources were allocated to Minikube during this assignment:

```bash
minikube start \
  --cpus=4 \
  --memory=6144 \
  --disk-size=20g \
  --kubernetes-version=v1.30.0
```

Verify the cluster:

```bash
minikube status
```

Expected output:

```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Verify Kubernetes connectivity:

```bash
kubectl cluster-info
kubectl get nodes
```

The node should appear in the **Ready** state.

---

# Architecture

The following diagram illustrates the deployed architecture.

![PostgreSQL Kubernetes Architecture](./postgresql/architecture.png)

The deployment consists of:

- **Percona PostgreSQL Operator**
  - Manages the PostgreSQL cluster lifecycle
  - Automates operational tasks

- **PostgreSQL Cluster**
  - Three PostgreSQL instances
  - Primary / Replica topology managed automatically

- **pgBouncer**
  - Connection pooling

- **Persistent Volumes**
  - Persistent database storage

- **Backup Repository**
  - Local repository used by the Operator

- **Kubernetes Services**
  - Internal access to PostgreSQL

- **TLS/SSL**
  - Enabled by the default Percona configuration

> **Note**
>
> This deployment runs on a **single-node Minikube cluster**.
> Although multiple PostgreSQL instances are created, this does **not** provide infrastructure-level high availability because all workloads run on the same Kubernetes node.

---

# Deployment

## 1. Add Percona Helm Repository

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update
```

---

## 2. Create Namespace

```bash
kubectl create namespace postgresql
```

The dedicated namespace isolates PostgreSQL resources from the rest of the cluster.

---

## 3. Install the Percona PostgreSQL Operator

```bash
helm install pg-operator percona/pg-operator -n postgresql
```

Verify the installation:

```bash
kubectl get pods -n postgresql
kubectl get deployments -n postgresql
kubectl get crds | grep -i percona
```

The CRDs confirm that the Operator extended Kubernetes with the custom resources required to manage PostgreSQL clusters.

---

## 4. Deploy the PostgreSQL Cluster

```bash
helm install postgres-cluster percona/pg-db -n postgresql
```

Wait until all Pods reach the **Running** state.

```bash
kubectl get pods -n postgresql -w
```

Press **Ctrl+C** once every Pod is running.

---

## 5. Verify the Deployment

```bash
kubectl get pods -n postgresql
kubectl get svc -n postgresql
kubectl get pvc -n postgresql
```

Verify that:

- PostgreSQL Pods are running
- pgBouncer Pods are running
- Services were created successfully
- PersistentVolumeClaims are in the **Bound** state

---

# Verification

## Retrieve Database Credentials

Username

```bash
kubectl get secret postgres-cluster-pg-d-pguser-postgres-cluster-pg-d \
-n postgresql \
-o jsonpath='{.data.user}' | base64 -d
```

Password

```bash
kubectl get secret postgres-cluster-pg-d-pguser-postgres-cluster-pg-d \
-n postgresql \
-o jsonpath='{.data.password}' | base64 -d
```

---

## Port Forward

```bash
kubectl port-forward \
-n postgresql \
svc/postgres-cluster-pg-d-pgbouncer \
5432:5432
```

---

## Connect with psql

```bash
PGSSLMODE=require \
psql \
-h localhost \
-U postgres-cluster-pg-d \
-d postgres
```

Enter the password when prompted.

---

## Execute Test Query

```sql
SELECT version();
```

Successful execution confirms that:

- PostgreSQL is reachable
- Database authentication succeeds
- SQL queries execute successfully
- SSL/TLS connectivity is active

Exit:

```sql
\q
```

---

# Design Decisions

### Minikube

Minikube was selected because the assignment requires a local Kubernetes environment.
It provides a lightweight Kubernetes cluster that is simple to install and suitable for local testing.

### Helm

The official Percona Helm charts were used to install both the PostgreSQL Operator and the PostgreSQL cluster.

This approach simplifies installation, improves reproducibility, and reduces manual resource management.

### Dedicated Namespace

A dedicated namespace (`postgresql`) was created to isolate database resources from other Kubernetes workloads.

Since this is a one-time local deployment, the namespace was created imperatively using:

```bash
kubectl create namespace postgresql
```

### Default Helm Values

The deployment intentionally uses the default Helm chart configuration.

The purpose of the assignment is to demonstrate a successful deployment rather than production tuning.

### Scope

Additional tooling such as:

- Terraform
- GitOps
- Custom Helm charts
- CI/CD automation

was intentionally excluded to keep the implementation aligned with the assignment scope.

---

# Limitations

- Single-node Kubernetes cluster
- Intended for local testing only
- No off-cluster backup storage
- Default Helm configuration
- No production resource tuning
- No monitoring stack
- No NetworkPolicies

---

# Future Improvements

Possible enhancements include:

- Custom `values.yaml`
- Resource requests and limits
- External S3-compatible backups
- Prometheus & Grafana monitoring
- Multi-node Kubernetes deployment
- NetworkPolicies
- Automated deployment using GitOps or CI/CD pipelines

---

# Summary

This project demonstrates the deployment of a PostgreSQL cluster managed by the Percona PostgreSQL Operator on a local Kubernetes environment.

The implementation includes:

- Kubernetes environment setup
- PostgreSQL Operator installation
- PostgreSQL cluster deployment
- Database connectivity verification
- Documentation of the implementation decisions

The solution intentionally remains simple, reproducible, and aligned with the scope of the assignment.