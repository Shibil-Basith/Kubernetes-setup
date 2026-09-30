# Helm Essentials: Beginner Session Guide

Welcome to the **Helm Essentials** session. This guide explains Helm in simple English, sticks to the core basics, and includes step-by-step practical demos and student exercises.

---

## 📌 Table of Contents
1. [What is Helm?](#1-what-is-helm)
2. [Core Concepts](#2-core-concepts)
3. [Architecture Overview](#3-architecture-overview)
4. [Demo 1: Deploying a Public Nginx Web Server](#4-demo-1-deploying-a-public-nginx-web-server)
5. [Demo 2: Configuring and Upgrading a Redis Release](#5-demo-2-configuring-and-upgrading-a-redis-release)
6. [Demo 3: Creating a Custom Helm Chart](#6-demo-3-creating-a-custom-helm-chart)
7. [Student Hands-On Tasks 🏋️](#7-student-hands-on-tasks-)
8. [Essential Commands Reference](#8-essential-commands-reference)

---

## 1. What is Helm?

Helm is a **package manager** for Kubernetes. It simplifies installing, updating, and removing applications from a Kubernetes cluster.

Without Helm, you must manually write and apply multiple Kubernetes configuration files (`Deployment`, `Service`, `ConfigMap`, `Ingress`) using `kubectl`. Helm groups these files into a single reusable package called a **Chart**.

---

## 2. Core Concepts

Understanding Helm requires knowing three key terms:

* **Chart:** A directory containing template YAML files that describe a set of Kubernetes resources.
* **Values (Config):** Configuration variables used to customize chart templates (such as port numbers, container image names, or passwords).
* **Release:** A running instance of a Chart in a Kubernetes cluster with specific values applied.

> **Key Rule:**  
> **Chart** (Application Files) + **Values** (Your Settings) = **Release** (Running Application)

---

## 3. Architecture Overview

The Helm workflow consists of three main parts:

1. **Helm CLI:** The command-line tool on your computer that accepts commands (e.g., `helm install`).
2. **Chart Repository:** An online or local server that stores and serves published Helm charts.
3. **Kubernetes Cluster:** The target environment where Helm creates and manages resources via standard Kubernetes API calls.

---

## 4. Demo 1: Deploying a Public Nginx Web Server

This example demonstrates how to find and install a pre-made community chart from a public repository.

### Step 1: Add a public chart repository
Repositories store charts online. Here we add the popular Bitnami repository:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Step 2: Install the application
Run the install command. The syntax is `helm install [RELEASE_NAME] [CHART_NAME]`:
```bash
helm install my-webserver bitnami/nginx
```

### Step 3: Verify the deployment
List all active Helm releases and check the Kubernetes pods created:
```bash
helm list
kubectl get pods
```

### Step 4: Clean up the release
Remove the application and all associated Kubernetes resources with a single command:
```bash
helm uninstall my-webserver
```

---

## 5. Demo 2: Configuring and Upgrading a Redis Release

This example shows how to pass custom values during installation and upgrade an active release without uninstalling it.

### Step 1: Install Redis with custom settings
We use the `--set` flag to override default values in the chart. Here, we disable password authentication for quick local testing:
```bash
helm install my-redis bitnami/redis --set auth.enabled=false
```
*Explanation:* The value `auth.enabled=false` instructs the chart template not to require passwords or create secret resources.

### Step 2: Upgrade the running release
To change settings on a live application (e.g., scaling replicas from 1 to 2), run `helm upgrade`:
```bash
helm upgrade my-redis bitnami/redis --set auth.enabled=false --set replica.replicaCount=2
```
*Explanation:* Helm compares the new configuration with the live cluster state and updates only the necessary components.

### Step 3: Check status and remove
```bash
kubectl get pods
helm uninstall my-redis
```

---

## 6. Demo 3: Creating a Custom Helm Chart

This example demonstrates how to build and install your own chart from scratch.

### Step 1: Create the chart directory
```bash
helm create my-app
```
This command generates a standard directory structure:
```text
my-app/
├── Chart.yaml          # Metadata about the chart
├── values.yaml         # Default configuration values
└── templates/          # Kubernetes manifest templates
    ├── deployment.yaml
    └── service.yaml
```

### Step 2: Inspect values.yaml
Open `my-app/values.yaml`. You will see default settings such as:
```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""
```

### Step 3: Install your custom chart
Install the chart from your local directory path:
```bash
helm install demo-release ./my-app
```

### Step 4: Clean up
```bash
helm uninstall demo-release
```

---

## 7. Student Hands-On Tasks 🏋️

Complete these tasks individually or in pairs to reinforce what you learned today.

### Task 1: Public Chart Search & Deployment
1. Search for the Apache chart in the Bitnami repo:
   ```bash
   helm search repo apache
   ```
2. Install Apache with the release name `site-v1`.
3. Verify the pod is running using `kubectl get pods`.
4. Uninstall the `site-v1` release.

### Task 2: Custom Values & Scaling
1. Create a chart named `custom-web` using `helm create custom-web`.
2. Edit `custom-web/values.yaml` and change `replicaCount` to `3`.
3. Install the chart with the release name `web-app`:
   ```bash
   helm install web-app ./custom-web
   ```
4. Verify that **3 pods** are created in Kubernetes.

### Task 3: Upgrade & Rollback Practice
1. Upgrade `web-app` to set `replicaCount=5` using the `--set` flag.
2. View release revision history:
   ```bash
   helm history web-app
   ```
3. Roll back to Revision 1:
   ```bash
   helm rollback web-app 1
   ```
4. Verify the replica count returns to **3 pods**.

---

## 8. Essential Commands Reference

| Operation | Command Syntax |
| :--- | :--- |
| **Add Repository** | `helm repo add [repo-name] [url]` |
| **Search Repository** | `helm search repo [keyword]` |
| **Install Chart** | `helm install [release-name] [chart]` |
| **List Releases** | `helm list` |
| **Upgrade Release** | `helm upgrade [release-name] [chart]` |
| **Rollback Release** | `helm rollback [release-name] [revision]` |
| **Uninstall Release** | `helm uninstall [release-name]` |
| **Create Chart** | `helm create [chart-name]` |
