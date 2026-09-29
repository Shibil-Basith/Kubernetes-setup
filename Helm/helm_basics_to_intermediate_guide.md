# Helm: Basics to Intermediate Guide 🚀

Welcome to the hands-on session guide for **Helm**, the package manager for Kubernetes. This document covers fundamental concepts up through intermediate chart management, templating, and deployment strategies in a single structured session.

---

## 📌 Table of Contents

 1. [Prerequisites & Setup](#1-prerequisites--setup)
 2. [What is Helm and Why Use It?](#2-what-is-helm-and-why-use-it)
 3. [Helm Architecture & Terminology](#3-helm-architecture--terminology)
 4. [Using Public Helm Repositories](#4-using-public-helm-repositories)
 5. [Creating Your First Helm Chart](#5-creating-your-first-helm-chart)
 6. [Helm Templating Fundamentals](#6-helm-templating-fundamentals)
 7. [Control Structures & Logic](#7-control-structures--logic)
 8. [Managing Chart Dependencies](#8-managing-chart-dependencies)
 9. [Release Management & Upgrades](#9-release-management--upgrades)
10. [Debugging & Dry Runs](#10-debugging--dry-runs)
11. [Step-by-Step Practical Examples](#11-step-by-step-practical-examples)
    * [Example 1: Deploying a Web Application (Node.js/Express)](#example-1-deploying-a-web-application-nodejs-express)
    * [Example 2: Deploying a Redis Cache with Dependencies](#example-2-deploying-a-redis-cache-with-dependencies)
12. [Hands-On Student Tasks & Lab Exercises](#12-hands-on-student-tasks--lab-exercises)
13. [Summary Cheatsheet](#13-summary-cheatsheet)

---

## 1. Prerequisites & Setup

Ensure you have the following installed before starting:

* **Kubernetes Cluster**: A local cluster like `minikube`, `kind`, or a remote cluster.
* **kubectl**: Configured to communicate with your cluster.
* **Helm 3**: Installed on your local machine.

### Installation Quick Check

```bash
# Verify kubectl connection
kubectl get nodes

# Verify Helm installation
helm version
```

---

## 2. What is Helm and Why Use It?

Kubernetes manifests (`yaml` files) for complex applications can quickly become difficult to maintain manually.

**Helm provides:**

* **Package Management**: Bundle multiple Kubernetes resources into a single unit (a Chart).
* **Parameterization**: Replace static values with dynamic variables (`values.yaml`).
* **Release Management**: Easily upgrade, rollback, or delete applications.
* **Reusability**: Share charts across teams or use community-maintained charts.

---

## 3. Helm Architecture & Terminology

* **Chart**: A collection of files describing a set of Kubernetes resources.
* **Repository**: A central location where charts are stored and shared.
* **Release**: An instance of a chart running in a Kubernetes cluster.
* **Values**: Configuration settings passed to customize a chart installation.

> 💡 **Helm 3 Architecture**: Helm 3 operates entirely client-side. It interacts directly with the Kubernetes API server using your local `kubeconfig` permissions (unlike Helm 2, which used an in-cluster server component called Tiller).

---

## 4. Using Public Helm Repositories

### Step 1: Add a Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Step 2: Search for Charts

```bash
helm search repo nginx
```

### Step 3: Install a Chart

```bash
helm install my-nginx bitnami/nginx --set service.type=NodePort
```

### Step 4: List and Inspect Releases

```bash
# List active releases
helm list

# Get status of a specific release
helm status my-nginx
```

---

## 5. Creating Your First Helm Chart

Generate a standard chart scaffold:

```bash
helm create my-app
```

### Directory Structure

```text
my-app/
├── Chart.yaml          # Metadata about the chart
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
└── templates/          # Go template files for K8s manifests
    ├── NOTES.txt       # Usage text printed after installation
    ├── deployment.yaml
    ├── service.yaml
    └── _helpers.tpl    # Template helpers/partials
```

---

## 6. Helm Templating Fundamentals

Helm uses Go templates augmented with the [Sprig library](https://masterminds.github.io/sprig/) for string functions.

### Example: Referencing `values.yaml`

**`values.yaml`**

```yaml
replicaCount: 3
image:
  repository: nginx
  tag: 1.25.0
```

**`templates/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### Built-in Objects

* `.Values`: Values passed from `values.yaml` or via `--set`.
* `.Release`: Information about the release (e.g., `.Release.Name`, `.Release.Namespace`).
* `.Chart`: Metadata from `Chart.yaml`.

---

## 7. Control Structures & Logic

### Conditionals (`if/else`)

```yaml
metadata:
  labels:
    {{- if .Values.enableLabels }}
    app.kubernetes.io/managed-by: Helm
    {{- else }}
    app.kubernetes.io/managed-by: Custom
    {{- end }}
```

### Pipelines and Functions

Transform variables using pipes (`|`):

```yaml
metadata:
  name: {{ .Release.Name | trunc 63 | trimSuffix "-" }}
  namespace: {{ .Release.Namespace | default "default" | quote }}
```

### Template Partial (`_helpers.tpl`)

Define reusable code blocks:

**`templates/_helpers.tpl`**

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

**`templates/deployment.yaml`**

```yaml
metadata:
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
```

---

## 8. Managing Chart Dependencies

Declare sub-charts directly inside `Chart.yaml`:

```yaml
apiVersion: v2
name: my-app
version: 0.1.0
dependencies:
  - name: mariadb
    version: 19.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: mariadb.enabled
```

### Download Dependencies

```bash
helm dependency update my-app/
```

---

## 9. Release Management & Upgrades

### Upgrading a Release

To change a configuration or update the chart version:

```bash
helm upgrade my-nginx bitnami/nginx --set service.type=ClusterIP
```

### Viewing History

```bash
helm history my-nginx
```

### Rolling Back

To revert to a previous revision (e.g., Revision 1):

```bash
helm rollback my-nginx 1
```

### Uninstalling

```bash
helm uninstall my-nginx
```

---

## 10. Debugging & Dry Runs

Always test chart templates before applying them to a live cluster:

```bash
# Render templates locally without contacting K8s
helm template my-app ./my-app

# Perform a dry run against the live API server
helm install my-app-test ./my-app --dry-run --debug
```

---

## 11. Step-by-Step Practical Examples

### Example 1: Deploying a Web Application (Node.js/Express)

In this practical example, we will create a complete Helm chart to deploy a custom Node.js application that requires a `ConfigMap` for application configuration and a `Service` for network access.

#### Step 1: Create the Chart Directory
```bash
helm create nodejs-app
cd nodejs-app
# Clean out default generated templates for a clear learning setup
rm -rf templates/*
```

#### Step 2: Define `values.yaml`
Edit `values.yaml` to include configuration values for image, environment, and replicas:

```yaml
# values.yaml
replicaCount: 2

image:
  repository: stefanprodan/podinfo
  tag: 6.5.0
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 9898

appConfig:
  message: "Hello from Helm-managed Node.js App!"
  logLevel: "info"
```

#### Step 3: Create the ConfigMap Template (`templates/configmap.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  labels:
    app: {{ .Chart.Name }}
data:
  APP_MESSAGE: {{ .Values.appConfig.message | quote }}
  LOG_LEVEL: {{ .Values.appConfig.logLevel | quote }}
```

#### Step 4: Create the Deployment Template (`templates/deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        release: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          envFrom:
            - configMapRef:
                name: {{ .Release.Name }}-config
```

#### Step 5: Create the Service Template (`templates/service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      protocol: TCP
      name: http
  selector:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
```

#### Step 6: Test and Deploy
```bash
# 1. Test template rendering locally
helm template my-web-app .

# 2. Deploy to Kubernetes
helm install my-web-app .

# 3. Verify resources were created
kubectl get pods,svc,configmap -l app=nodejs-app
```

---

### Example 2: Deploying a Redis Cache with Dependencies

In this example, we will see how to construct a parent chart (`backend-stack`) that pulls in an official community chart (Redis) as a dependency.

#### Step 1: Create Parent Chart
```bash
helm create backend-stack
cd backend-stack
```

#### Step 2: Configure Dependencies in `Chart.yaml`
Add Bitnami's Redis chart as a dependency inside `Chart.yaml`:

```yaml
apiVersion: v2
name: backend-stack
description: A Helm chart for Backend Stack with Redis dependency
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: redis
    version: 18.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

#### Step 3: Fetch the Dependency
```bash
helm dependency update
# This downloads redis-18.x.x.tgz into the charts/ directory
```

#### Step 4: Override Dependency Values in `values.yaml`
Configure the Redis sub-chart values directly inside your parent `values.yaml`:

```yaml
# Parent chart configuration
replicaCount: 1

# Dependency configuration (matches dependency name 'redis')
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
  master:
    service:
      ports:
        redis: 6379
```

#### Step 5: Deploy the Stack
```bash
helm install prod-stack .
```

#### Step 6: Verify the Installation
```bash
# Verify both parent resources and Redis sub-chart resources are deployed
helm list
kubectl get pods
```

---

## 12. Hands-On Student Tasks & Lab Exercises

Complete these practical exercises to test your understanding.

### 🏋️ Task 1: Basic Deployment & Parameterization
**Objective:** Create and deploy a customized Nginx deployment using your own Helm chart.

1. Create a new Helm chart named `student-web`.
2. Modify `values.yaml` to set `replicaCount` to `3` and change the Nginx container port to `8080`.
3. Update `templates/deployment.yaml` so container port dynamically reads from `values.yaml`.
4. Deploy the chart using the release name `demo-web`.
5. Use `kubectl get pods` to verify 3 replicas are running.

### 🏋️ Task 2: Upgrades and Rollbacks
**Objective:** Perform zero-downtime updates and roll back changes.

1. Upgrade your `demo-web` release to change `replicaCount` from `3` to `5` using the command line `--set` flag:
   ```bash
   helm upgrade demo-web ./student-web --set replicaCount=5
   ```
2. Verify that 5 pods are created.
3. Check release history using `helm history demo-web`.
4. Perform a rollback to revision 1:
   ```bash
   helm rollback demo-web 1
   ```
5. Confirm that the pod count scales back down to 3.

### 🏋️ Task 3: Conditionals & Ingress
**Objective:** Practice template logic using `if` statements.

1. Add an `ingress.enabled` boolean setting in `values.yaml` (default: `false`).
2. Create an `ingress.yaml` template file under `templates/` wrapped in a conditional block:
   ```yaml
   {{- if .Values.ingress.enabled }}
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   ...
   {{- end }}
   ```
3. Run `helm template demo-web ./student-web` and verify that Ingress YAML is NOT generated.
4. Run `helm template demo-web ./student-web --set ingress.enabled=true` and confirm the Ingress resource IS generated.

### 🏋️️ Task 4: Multi-Environment Deployment
**Objective:** Manage multiple environments using separate values files.

1. Create a file named `values-dev.yaml` with `replicaCount: 1`.
2. Create a file named `values-prod.yaml` with `replicaCount: 4`.
3. Render the templates for production:
   ```bash
   helm template my-release ./student-web -f values-prod.yaml
   ```
4. Verify that the output manifest specifies 4 replicas.

---

## 13. Summary Cheatsheet

| Command | Description |
| :--- | :--- |
| `helm create <name>` | Scaffold a new chart directory |
| `helm repo add <name> <url>` | Add a remote Helm chart repository |
| `helm repo update` | Fetch latest chart info from repositories |
| `helm install <release> <chart>` | Deploy a chart to the cluster |
| `helm upgrade <release> <chart>` | Update an existing release |
| `helm rollback <release> <revision>` | Roll back to a previous revision |
| `helm list` | View all active releases in current namespace |
| `helm uninstall <release>` | Delete a release and its resources |
| `helm template <chart>` | Render manifests locally for debugging |
| `helm dependency update` | Download dependencies listed in `Chart.yaml` |
