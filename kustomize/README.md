# Kustomize Basics

## 1. What is Kustomize?

Kustomize is a tool used to **manage and customize Kubernetes YAML files**.

In Kubernetes, we normally create YAML files such as:

```text
deployment.yaml
service.yaml
configmap.yaml
```

Sometimes we need the **same application with small changes**.

For example:

### Development

```text
1 replica
nginx:latest
```

### Production

```text
5 replicas
nginx:1.27
```

We do not want to create completely different YAML files for every environment.

Kustomize helps us **reuse the same YAML files and make small changes**.

### Simple definition

> **Kustomize allows us to customize Kubernetes YAML files without changing the original YAML files.**

---

# 2. Why Do We Need Kustomize?

Imagine we have this Deployment:

```yaml
replicas: 2
```

We want:

```text
Development → 1 replica
Testing     → 2 replicas
Production  → 5 replicas
```

Without Kustomize, we might create:

```text
deployment-dev.yaml
deployment-test.yaml
deployment-prod.yaml
```

This creates duplicate YAML files.

With Kustomize:

```text
                 Base
                  |
          deployment.yaml
                  |
       ┌──────────┴──────────┐
       ↓                     ↓
     Dev                    Prod
   1 replica              5 replicas
```

We keep **one original YAML** and customize it.

---

# 3. Does Kustomize Need Installation?

For basic usage, Kustomize can be used through `kubectl`.

Check your Kubernetes client:

```bash
kubectl version --client
```

Check Kustomize:

```bash
kubectl kustomize version
```

If the command works, Kustomize is available through `kubectl`.

---

# 4. Important Kustomize Terms

There are two important words we need to understand.

## Base

**Base means the original/common Kubernetes configuration.**

Example:

```text
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

The base contains the common configuration.

---

## Overlay

**Overlay means the changes we want to apply to the base.**

Example:

```text
base
  |
  ├── development
  |
  └── production
```

Development can have:

```text
1 replica
```

Production can have:

```text
5 replicas
```

The base does not change.

---

# 5. What is `kustomization.yaml`?

This is the most important Kustomize file.

The filename is:

```text
kustomization.yaml
```

It tells Kustomize:

> "These are my Kubernetes resources, and these are the changes I want."

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

Here we are telling Kustomize:

```text
Use deployment.yaml
Use service.yaml
```

---

# 6. Create Our First Kustomize Project

Create a project:

```bash
mkdir kustomize-demo
cd kustomize-demo
```

Create the following structure:

```text
kustomize-demo/
│
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
│
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    │
    └── prod/
        └── kustomization.yaml
```

---

# 7. Create the Base Deployment

Create the base directory:

```bash
mkdir -p base
cd base
```

Create the Deployment:

```bash
vim deployment.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

Our base Deployment has:

```text
Application: nginx
Replicas: 2
Image: nginx:latest
```

---

# 8. Create the Service

Create:

```bash
vim service.yaml
```

Add:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  type: NodePort

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

---

# 9. Create `kustomization.yaml`

Create:

```bash
vim kustomization.yaml
```

Add:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

Our base is now:

```text
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

---

# 10. Test the Base

Go back to the project directory:

```bash
cd ..
```

Run:

```bash
kubectl kustomize base
```

Kustomize will show the final YAML.

### Important

```bash
kubectl kustomize base
```

does **not deploy** anything.

It only shows us what Kustomize will generate.

---

# 11. Apply the Base

To deploy:

```bash
kubectl apply -k base
```

Check the Deployment:

```bash
kubectl get deployment
```

Check the Pods:

```bash
kubectl get pods
```

You should have 2 nginx Pods.

---

# Demo 1 – Change Replicas Using Kustomize

## Goal

Our base has:

```text
Base → 2 replicas
```

We want:

```text
Development → 1 replica
Production → 5 replicas
```

We will **not change `deployment.yaml`**.

---

# 12. Create Development Overlay

Create the directory:

```bash
mkdir -p overlays/dev
```

Create:

```bash
vim overlays/dev/kustomization.yaml
```

Add:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: nginx
    count: 1
```

### Understand this

This:

```yaml
resources:
  - ../../base
```

means:

> Use my base configuration.

This:

```yaml
replicas:
  - name: nginx
    count: 1
```

means:

> Change the nginx Deployment to 1 replica.

---

# 13. Test Development Configuration

Run:

```bash
kubectl kustomize overlays/dev
```

Look at the generated Deployment.

You should find:

```yaml
replicas: 1
```

Notice:

Our original file still says:

```yaml
replicas: 2
```

We did not modify it.

Kustomize created the customized configuration for us.

---

# 14. Deploy Development

Run:

```bash
kubectl apply -k overlays/dev
```

Check:

```bash
kubectl get pods
```

You should have:

```text
1 nginx Pod
```

Check the Deployment:

```bash
kubectl get deployment
```

You should see 1 available replica.

---

# 15. Create Production Overlay

Create:

```bash
mkdir -p overlays/prod
```

Create:

```bash
vim overlays/prod/kustomization.yaml
```

Add:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: nginx
    count: 5
```

Now:

```text
Base
  ↓
2 replicas

Production overlay
  ↓
5 replicas
```

---

# 16. Test Production

Run:

```bash
kubectl kustomize overlays/prod
```

You will see:

```yaml
replicas: 5
```

Deploy:

```bash
kubectl apply -k overlays/prod
```

Check:

```bash
kubectl get pods
```

You should see 5 nginx Pods.

---

# 17. Important Point

Look at our original file:

```text
base/deployment.yaml
```

It still contains:

```yaml
replicas: 2
```

We did not change it.

But:

```text
dev  → 1 replica
prod → 5 replicas
```

This is the main idea of Kustomize.

> **Base stays common. Overlay contains the changes.**

---

# Demo 2 – Change Docker Image Using Kustomize

Now let's change the container image.

Our base has:

```yaml
image: nginx:latest
```

Suppose production wants:

```text
nginx:1.27
```

We do not want to edit the base.

Kustomize can change the image for us.

---

# 18. Modify Production Kustomization

Open:

```bash
vim overlays/prod/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: nginx
    count: 5

images:
  - name: nginx
    newName: nginx
    newTag: "1.27"
```

The important part is:

```yaml
images:
  - name: nginx
    newName: nginx
    newTag: "1.27"
```

This means:

```text
Old:

nginx:latest

       ↓

Kustomize

       ↓

New:

nginx:1.27
```

---

# 19. Check the Generated YAML

Run:

```bash
kubectl kustomize overlays/prod
```

Find:

```yaml
image: nginx:1.27
```

Notice:

The original base still has:

```yaml
image: nginx:latest
```

We did not change it.

---

# 20. Apply the Image Change

Run:

```bash
kubectl apply -k overlays/prod
```

Check:

```bash
kubectl get pods
```

You can check the Deployment:

```bash
kubectl get deployment nginx -o yaml
```

You should find:

```yaml
image: nginx:1.27
```

---

# 21. What Happened?

Our structure is now:

```text
                  BASE
                   │
          ┌────────┴────────┐
          │                 │
         DEV              PROD
          │                 │
    1 replica          5 replicas
                         │
                    nginx:1.27
```

Base:

```text
replicas: 2
image: nginx:latest
```

Dev:

```text
replicas: 1
image: nginx:latest
```

Prod:

```text
replicas: 5
image: nginx:1.27
```

---

# 22. Important Commands

## Generate YAML

```bash
kubectl kustomize base
```

```bash
kubectl kustomize overlays/dev
```

```bash
kubectl kustomize overlays/prod
```

---

## Apply

```bash
kubectl apply -k base
```

```bash
kubectl apply -k overlays/dev
```

```bash
kubectl apply -k overlays/prod
```

---

## Delete

```bash
kubectl delete -k overlays/dev
```

```bash
kubectl delete -k overlays/prod
```

---

# 23. `-f` vs `-k`

This is important.

Normal Kubernetes YAML:

```bash
kubectl apply -f deployment.yaml
```

`-f` means:

> Apply this YAML file.

Kustomize:

```bash
kubectl apply -k overlays/prod
```

`-k` means:

> Use Kustomize to build the configuration and apply it.

### Easy way to remember

```text
-f → YAML file

-k → Kustomize directory
```

---

# 24. Kustomize vs Helm

A simple comparison:

| Helm | Kustomize |
|---|---|
| Uses templates | Uses existing YAML |
| Uses `values.yaml` | Uses overlays |
| Package manager | Configuration customization |
| Uses `helm install` | Uses `kubectl apply -k` |
| Powerful templating | Simple YAML customization |

### Simple explanation

Helm:

```text
Template
   +
Values
   ↓
Final YAML
```

Kustomize:

```text
Base YAML
   +
Overlay
   ↓
Final YAML
```

---

# 25. What Students Should Remember

### Kustomize

> Tool for customizing Kubernetes YAML.

### Base

> Common/original configuration.

### Overlay

> Changes made on top of the base.

### `kustomization.yaml`

> File that tells Kustomize what resources to use and what changes to make.

### `kubectl kustomize`

> Shows the final YAML.

### `kubectl apply -k`

> Applies the Kustomize configuration to Kubernetes.

---

# 26. Real-World Example

Imagine a company has:

```text
my-app
```

They have three environments:

```text
Development
Testing
Production
```

Common configuration:

```text
Deployment
Service
ConfigMap
```

They create:

```text
base/
```

Then:

```text
overlays/
├── dev/
├── test/
└── prod/
```

They can customize:

```text
Dev:
1 replica

Test:
2 replicas

Prod:
5 replicas
```

They can also use different images:

```text
Dev:
myapp:dev

Test:
myapp:test

Prod:
myapp:v1.0
```

The common YAML does not need to be copied three times.

---

# 27. Kustomize in One Picture

```text
                 KUSTOMIZE

                    BASE
                     │
          ┌──────────┼──────────┐
          │          │          │
         DEV        TEST       PROD
          │          │          │
       1 replica   2 replicas  5 replicas
          │          │          │
       dev image   test image  prod image
```

### Main idea

> **Write once, customize many times.**

---

# 28. Beginner Session Flow

Recommended teaching order:

```text
1. What is Kustomize?
        ↓
2. Why do we need Kustomize?
        ↓
3. Base and Overlay
        ↓
4. kustomization.yaml
        ↓
5. Create project
        ↓
6. Create Deployment + Service
        ↓
7. kubectl kustomize
        ↓
8. kubectl apply -k
        ↓
9. Demo 1 – Change replicas
        ↓
10. Demo 2 – Change image
        ↓
11. Review commands
```

---

# 29. Final One-Line Definition

> **Kustomize is a Kubernetes tool that lets us reuse existing YAML files and customize them for different environments without changing the original YAML files.**
