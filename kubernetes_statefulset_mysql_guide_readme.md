# Kubernetes StatefulSet Demo: Step-by-Step MySQL Setup for Kubeadm

[![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![MySQL](https://img.shields.io/badge/mysql-8.0-4479A1.svg?style=flat&logo=mysql&logoColor=white)](https://www.mysql.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A practical, step-by-step tutorial demonstrating how to deploy a resilient, multi-replica MySQL database on a bare-metal or custom Kubernetes cluster built with `kubeadm`.

---

## Table of Contents

- [Overview](#overview)
- [Why Use a StatefulSet for Databases?](#why-use-a-statefulset-for-databases)
- [Implementation Roadmap](#implementation-roadmap)
- [Prerequisites](#prerequisites)
- [Step-by-Step Setup](#step-by-step-setup)
  - [Step 1: Create Host Directories](#step-1-create-host-directories-on-worker-node)
  - [Step 2: Create StorageClass](#step-2-create-the-manual-storageclass)
  - [Step 3: Create PersistentVolumes (PVs)](#step-3-create-static-persistentvolumes-pvs)
  - [Step 4: Create Service & ConfigMap](#step-4-create-service--configmap)
  - [Step 5: Deploy StatefulSet](#step-5-deploy-the-mysql-statefulset)
  - [Step 6: Verify Ordered Startup & Storage Binding](#step-6-verify-ordered-startup-and-storage-binding)
  - [Step 7: Test Data Persistence](#step-7-test-data-persistence)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)

---

## Overview

In managed cloud environments (like EKS, GKE, or AKS), dynamic storage provisioners automatically provision underlying block storage when requested. On bare-metal or virtual-machine clusters initialized with `kubeadm`, dynamic storage provisioners are not available by default.

This guide illustrates how to use **Static Persistent Volumes** backed by local host directories (`hostPath`) along with a **StatefulSet** and a **Headless Service** to provide deterministic identities, stable storage bindings, and data durability for a MySQL database.

---

## Why Use a StatefulSet for Databases?

Unlike standard stateless applications managed by a `Deployment`, databases require strict operational consistency:

| Feature | Deployment | StatefulSet | Benefit for Databases |
| :--- | :--- | :--- | :--- |
| **Pod Identity** | Random hashes (`web-7d4b8f-x9kz2`) | Predictable ordinal index (`mysql-0`, `mysql-1`) | Static endpoints for primary/replica routing. |
| **Startup Order** | Concurrent/random | Sequential (`mysql-0` must be healthy before `mysql-1` starts) | Predictable cluster bootstrapping and replication setup. |
| **Storage Binding** | Shared volume or ephemeral | Dedicated volume per ordinal via `volumeClaimTemplates` | Replaced pods automatically reconnect to their original storage volume. |

---

## Implementation Roadmap

| Step | Action | Purpose |
| :---: | :--- | :--- |
| **1** | Create Host Directories | Prepares folder locations on the worker node host drive to store database files. |
| **2** | Create StorageClass | Informs Kubernetes to expect manual volume creation rather than cloud storage. |
| **3** | Create PersistentVolumes (PVs) | Defines actual static storage blocks mapped to worker node host folders. |
| **4** | Create Service & ConfigMap | Configures database initialization parameters and provides stable DNS endpoints. |
| **5** | Deploy StatefulSet | Launches the MySQL container instances. |
| **6** | Verify Setup | Confirms sequential pod startup and successful volume binding. |
| **7** | Test Data Persistence | Writes test records, terminates a pod, and confirms zero data loss upon restart. |

---

## Prerequisites

- A running Kubernetes cluster (v1.20+) created via `kubeadm`.
- `kubectl` configured with cluster administrative access.
- SSH or terminal access to the target worker node to create host directories.

---

## Step-by-Step Setup

### Step 1: Create Host Directories on Worker Node

Because standard `kubeadm` clusters do not include cloud-backed storage provisioners, create persistent storage directories directly on the worker node where the pods will run.

Log in to your worker node server terminal:

```bash
sudo mkdir -p /mnt/data/mysql-0 /mnt/data/mysql-1
sudo chmod 777 /mnt/data/mysql-0 /mnt/data/mysql-1
```

**Explanation:**
- `mkdir -p`: Generates the directories `/mnt/data/mysql-0` and `/mnt/data/mysql-1`.
- `chmod 777`: Grants full read and write permissions so the MySQL process inside the container can manage tablespaces and log files without permission issues.

---

### Step 2: Create the "manual" StorageClass

A `StorageClass` defines how storage is provisioned. Using `provisioner: kubernetes.io/no-provisioner` signals that storage will be statically registered rather than dynamically provisioned by an external storage plugin.

Create a file named `storageclass-manual.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Apply the manifest:

```bash
kubectl apply -f storageclass-manual.yaml
```

**Key Points:**
- `name: manual`: Used by PersistentVolumes and VolumeClaims to bind together.
- `volumeBindingMode: WaitForFirstConsumer`: Prevents Kubernetes from prematurely binding storage until a Pod requesting the volume is scheduled to a specific node.

---

### Step 3: Create Static PersistentVolumes (PVs)

Register the directories created in Step 1 as official cluster storage objects.

Create a file named `mysql-pvs.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-mysql-0
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data/mysql-0"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-mysql-1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data/mysql-1"
```

Apply the manifest:

```bash
kubectl apply -f mysql-pvs.yaml
```

**Key Points:**
- `capacity: storage: 2Gi`: Allocates 2 Gigabytes per volume.
- `hostPath`: Directly links `pv-mysql-0` to `/mnt/data/mysql-0` and `pv-mysql-1` to `/mnt/data/mysql-1`.
- `persistentVolumeReclaimPolicy: Retain`: Guarantees database data files will not be deleted from the host disk even if the PV or PVC objects are removed.

---

### Step 4: Create Service & ConfigMap

Databases require configuration profiles and predictable network routing.

Create a file named `mysql-services.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    default-authentication-plugin=mysql_native_password
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
```

Apply the manifest:

```bash
kubectl apply -f mysql-services.yaml
```

**Key Points:**
- **ConfigMap (`mysql-config`)**: Supplies configuration overrides to enable `mysql_native_password` authentication for broader client compatibility with MySQL 8.
- **Headless Service (`mysql-headless`)**: Setting `clusterIP: None` disables proxying and round-robin load balancing. Instead, it generates stable internal DNS records for each individual replica:
  - Pod 0: `mysql-0.mysql-headless.<namespace>.svc.cluster.local`
  - Pod 1: `mysql-1.mysql-headless.<namespace>.svc.cluster.local`

---

### Step 5: Deploy the MySQL StatefulSet

Launch the database instances managed by the StatefulSet controller.

Create a file named `mysql-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql-headless"
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "SecretPassword123"
        - name: MYSQL_DATABASE
          value: "demodb"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: manual
      resources:
        requests:
          storage: 2Gi
```

Apply the manifest:

```bash
kubectl apply -f mysql-statefulset.yaml
```

**Key Points:**
- `replicas: 2`: Deploys two database pods (`mysql-0` and `mysql-1`).
- `volumeMounts`: Mounts the persistent disk to `/var/lib/mysql`, where MySQL stores database schemas and tables.
- `volumeClaimTemplates`: Automatically provisions a unique `PersistentVolumeClaim` (PVC) per pod index:
  - `mysql-0` binds to PVC `data-mysql-0` (and `pv-mysql-0`).
  - `mysql-1` binds to PVC `data-mysql-1` (and `pv-mysql-1`).

---

### Step 6: Verify Ordered Startup and Storage Binding

1. Monitor pod initialization to verify ordered, sequential creation:

   ```bash
   kubectl get pods -l app=mysql -w
   ```

   *Notice that `mysql-1` remains in `Pending` or `ContainerCreating` until `mysql-0` enters the `Running` state.*

2. Confirm PersistentVolumeClaims are successfully bound to the host PersistentVolumes:

   ```bash
   kubectl get pvc,pv
   ```

   **Expected Output:**
   ```text
   NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   persistentvolumeclaim/data-mysql-0   Bound    pv-mysql-0   2Gi        RWO            manual         1m
   persistentvolumeclaim/data-mysql-1   Bound    pv-mysql-1   2Gi        RWO            manual         1m

   NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   AGE
   persistentvolume/pv-mysql-0   2Gi        RWO            Retain           Bound    default/data-mysql-0   manual         3m
   persistentvolume/pv-mysql-1   2Gi        RWO            Retain           Bound    default/data-mysql-1   manual         3m
   ```

---

### Step 7: Test Data Persistence

Validate that data persists across pod failures and recreations.

1. **Insert test records into `mysql-0`:**

   ```bash
   kubectl exec -it mysql-0 -- mysql -u root -pSecretPassword123 demodb -e "
   CREATE TABLE test_table (id INT AUTO_INCREMENT PRIMARY KEY, message VARCHAR(255));
   INSERT INTO test_table (message) VALUES ('StatefulSet storage test succeeded!');
   "
   ```

2. **Verify the data was committed:**

   ```bash
   kubectl exec -it mysql-0 -- mysql -u root -pSecretPassword123 demodb -e "SELECT * FROM test_table;"
   ```

3. **Delete `mysql-0` to simulate a failure or restart:**

   ```bash
   kubectl delete pod mysql-0
   ```

4. **Wait for Kubernetes to restart `mysql-0`:**

   ```bash
   kubectl get pod mysql-0 -w
   ```

5. **Query the newly recreated pod:**

   ```bash
   kubectl exec -it mysql-0 -- mysql -u root -pSecretPassword123 demodb -e "SELECT * FROM test_table;"
   ```

   **Expected Result:**
   ```text
   +----+---------------------------------------+
   | id | message                               |
   +----+---------------------------------------+
   |  1 | StatefulSet storage test succeeded!   |
   +----+---------------------------------------+
   ```
   The data remains intact because the restarted pod reattached to `/mnt/data/mysql-0` on the worker node.

---

## Cleanup

To delete all Kubernetes resources created during this demo, run:

```bash
# Delete the StatefulSet and associated pods
kubectl delete statefulset mysql

# Delete the Headless Service and ConfigMap
kubectl delete service mysql-headless
kubectl delete configmap mysql-config

# Delete the storage claims and volumes
kubectl delete pvc data-mysql-0 data-mysql-1
kubectl delete pv pv-mysql-0 pv-mysql-1

# Delete the StorageClass
kubectl delete storageclass manual
```

*(Optional)* To remove residual data files from your worker node:

```bash
sudo rm -rf /mnt/data/mysql-0 /mnt/data/mysql-1
```

---

## Troubleshooting

- **Pod stuck in `Pending`:**
  Check pod events with `kubectl describe pod mysql-0`. If the issue relates to `volumeBindingMode: WaitForFirstConsumer`, ensure your node has matching labels and sufficient available memory/CPU.
- **MySQL permission errors on data directory:**
  Ensure permissions on the worker node are properly set using `sudo chmod 777 /mnt/data/mysql-0 /mnt/data/mysql-1`.
- **Multi-Node Clusters:**
  Because `hostPath` is tied to a specific physical node, in a multi-worker cluster you may want to set `nodeSelector` or `affinity` in the Pod template to ensure pods land on the specific worker node where the directories exist.