StatefulSet with PV, PVC and StorageClass
Kubernetes Setup
We are using:
1 Control Plane
2 Worker Nodes
kubeadm cluster
Manual/Static storage
No dynamic provisioner
---
1. What are we going to create?
We will create:
```text
StorageClass
      ↓
Persistent Volumes (PV)
      ↓
PersistentVolumeClaims (PVC)
      ↓
StatefulSet
      ↓
MySQL Pods
```
Final setup:
```text
Worker 1
   |
   └── PV-0
         |
         └── mysql-sts-0


Worker 2
   |
   └── PV-1
         |
         └── mysql-sts-1
```
Each MySQL Pod gets its own storage.
---
2. Check the Nodes
Run this command from the Control Plane:
```bash
kubectl get nodes -o wide
```
Example:
```text
NAME       STATUS   ROLES           INTERNAL-IP
master     Ready    control-plane   192.168.1.10
worker1    Ready    <none>          192.168.1.11
worker2    Ready    <none>          192.168.1.12
```
Remember your actual worker node names.
> Replace `worker1` and `worker2` with your actual node names.
---
3. Create Storage Directories
We are using local directories as storage.
On Worker 1
SSH into Worker 1:
```bash
ssh user@worker1
```
Create the directory:
```bash
sudo mkdir -p /mnt/mysql-0
```
Give permission:
```bash
sudo chmod 777 /mnt/mysql-0
```
Check:
```bash
ls -ld /mnt/mysql-0
```
---
On Worker 2
SSH into Worker 2:
```bash
ssh user@worker2
```
Create the directory:
```bash
sudo mkdir -p /mnt/mysql-1
```
Give permission:
```bash
sudo chmod 777 /mnt/mysql-1
```
Check:
```bash
ls -ld /mnt/mysql-1
```
---
4. Create a Namespace
Go back to the Control Plane.
Create a namespace:
```bash
kubectl create namespace stateful-demo
```
Check:
```bash
kubectl get namespaces
```
---
5. Create StorageClass
Create a file:
```bash
vim storageclass.yaml
```
Add:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
Save the file.
Apply it:
```bash
kubectl apply -f storageclass.yaml
```
Check:
```bash
kubectl get storageclass
```
You should see:
```text
NAME     PROVISIONER
manual   kubernetes.io/no-provisioner
```
What does this mean?
Normally, a StorageClass can automatically create PVs.
Here we are using:
```yaml
provisioner: kubernetes.io/no-provisioner
```
This means:
> Kubernetes will NOT create PVs automatically.
We will create the PVs manually.
---
6. Create PV for Worker 1
Create:
```bash
vim pv-0.yaml
```
Add:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-0

spec:
  capacity:
    storage: 1Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: manual

  local:
    path: /mnt/mysql-0

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker1
```
> Change `worker1` to your actual Worker 1 node name.
Apply:
```bash
kubectl apply -f pv-0.yaml
```
Check:
```bash
kubectl get pv
```
You should see:
```text
mysql-pv-0   1Gi   RWO   Retain   Available   manual
```
---
7. Create PV for Worker 2
Create:
```bash
vim pv-1.yaml
```
Add:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-1

spec:
  capacity:
    storage: 1Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: manual

  local:
    path: /mnt/mysql-1

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - key: kubernetes.io/hostname
          operator: In
          values:
            - worker2
```
> Change `worker2` to your actual Worker 2 node name.
Apply:
```bash
kubectl apply -f pv-1.yaml
```
Check:
```bash
kubectl get pv
```
Expected:
```text
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
mysql-pv-0    1Gi        RWO            Retain           Available   manual
mysql-pv-1    1Gi        RWO            Retain           Available   manual
```
---
8. Create Headless Service
A StatefulSet normally uses a Headless Service.
Create:
```bash
vim mysql-service.yaml
```
Add:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: stateful-demo

spec:
  clusterIP: None

  selector:
    app: mysql

  ports:
    - port: 3306
      targetPort: 3306
```
Apply:
```bash
kubectl apply -f mysql-service.yaml
```
Check:
```bash
kubectl get svc -n stateful-demo
```
You should see:
```text
NAME    TYPE        CLUSTER-IP   PORT(S)
mysql   ClusterIP   None         3306/TCP
```
---
9. Create StatefulSet
Create:
```bash
vim mysql-statefulset.yaml
```
Add:
```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: mysql-sts
  namespace: stateful-demo

spec:
  serviceName: mysql

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
              value: root123

          ports:
            - containerPort: 3306

          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql

  volumeClaimTemplates:

    - metadata:
        name: mysql-storage

      spec:
        storageClassName: manual

        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 1Gi
```
Apply:
```bash
kubectl apply -f mysql-statefulset.yaml
```
---
10. Check the StatefulSet
Run:
```bash
kubectl get statefulset -n stateful-demo
```
Expected:
```text
NAME        READY
mysql-sts   2/2
```
---
11. Check the Pods
Run:
```bash
kubectl get pods -n stateful-demo -o wide
```
Expected:
```text
NAME          READY   STATUS    NODE
mysql-sts-0   1/1     Running   worker1
mysql-sts-1   1/1     Running   worker2
```
The exact worker assignment depends on scheduling and the PV node affinity.
---
12. Check the PVCs
Run:
```bash
kubectl get pvc -n stateful-demo
```
Expected:
```text
NAME                         STATUS   VOLUME
mysql-storage-mysql-sts-0   Bound    mysql-pv-0
mysql-storage-mysql-sts-1   Bound    mysql-pv-1
```
What happened?
The StatefulSet used:
```yaml
volumeClaimTemplates:
```
Kubernetes automatically created:
```text
mysql-storage-mysql-sts-0
mysql-storage-mysql-sts-1
```
These are PVCs.
---
13. Check the PVs
Run:
```bash
kubectl get pv
```
Expected:
```text
NAME          STATUS   CLAIM
mysql-pv-0    Bound    stateful-demo/mysql-storage-mysql-sts-0
mysql-pv-1    Bound    stateful-demo/mysql-storage-mysql-sts-1
```
The relationship is:
```text
mysql-sts-0
     |
     ↓
PVC: mysql-storage-mysql-sts-0
     |
     ↓
PV: mysql-pv-0
     |
     ↓
/mnt/mysql-0
     |
     ↓
Worker 1
```
And:
```text
mysql-sts-1
     |
     ↓
PVC: mysql-storage-mysql-sts-1
     |
     ↓
PV: mysql-pv-1
     |
     ↓
/mnt/mysql-1
     |
     ↓
Worker 2
```
---
14. Test MySQL
Check the Pods:
```bash
kubectl get pods -n stateful-demo
```
Enter the first Pod:
```bash
kubectl exec -it mysql-sts-0 -n stateful-demo -- bash
```
Login to MySQL:
```bash
mysql -u root -p
```
Password:
```text
root123
```
Create a database:
```sql
CREATE DATABASE testdb;
```
Check:
```sql
SHOW DATABASES;
```
Exit MySQL:
```sql
exit
```
Exit the container:
```bash
exit
```
---
15. Delete the Pod
This is an important StatefulSet test.
Delete:
```bash
kubectl delete pod mysql-sts-0 -n stateful-demo
```
Check:
```bash
kubectl get pods -n stateful-demo -w
```
Kubernetes will create the Pod again:
```text
mysql-sts-0
```
Notice the name.
It is still:
```text
mysql-sts-0
```
It does not get a random name.
---
16. Check the PVC
Run:
```bash
kubectl get pvc -n stateful-demo
```
The PVC should still exist:
```text
mysql-storage-mysql-sts-0
```
The storage was not removed when the Pod was deleted.
---
17. Check the Data
After the new Pod becomes `Running`:
```bash
kubectl exec -it mysql-sts-0 -n stateful-demo -- bash
```
Login:
```bash
mysql -u root -p
```
Password:
```text
root123
```
Run:
```sql
SHOW DATABASES;
```
You should still see:
```text
testdb
```
This shows that the data is stored in persistent storage.
---
18. Important StatefulSet Features
Stable Pod Names
Deployment:
```text
app-7d8f9c-abc12
app-7d8f9c-def34
```
StatefulSet:
```text
mysql-sts-0
mysql-sts-1
```
StatefulSet gives Pods stable names.
---
Stable Storage
Each Pod gets its own PVC.
```text
mysql-sts-0 → PVC-0 → PV-0

mysql-sts-1 → PVC-1 → PV-1
```
---
Stable Network Identity
Our Headless Service is:
```text
mysql
```
The Pods can have stable DNS names such as:
```text
mysql-sts-0.mysql
mysql-sts-1.mysql
```
---
19. Deployment vs StatefulSet
Deployment
Usually used for:
Web applications
APIs
Frontend applications
Stateless applications
Example:
```text
Deployment
    ↓
Pod
    ↓
Pod
    ↓
Pod
```
Pods don't need individual identities.
---
StatefulSet
Used when Pods need:
Stable names
Stable storage
Stable network identity
Examples:
Databases
Distributed systems
Stateful applications
Example:
```text
StatefulSet
     |
     ├── mysql-sts-0
     |       |
     |       └── PVC-0
     |
     └── mysql-sts-1
             |
             └── PVC-1
```
---
20. Important Terms
Term	Simple meaning
StorageClass	Defines the storage type/rules
PV	Actual storage
PVC	Request for storage
StatefulSet	Manages stateful Pods
Headless Service	Helps provide stable network identity
volumeClaimTemplates	Creates a PVC for each StatefulSet Pod
`no-provisioner`	PVs must be created manually
`ReadWriteOnce`	Storage can be mounted for read/write by one node
---
21. Useful Commands
Check all resources:
```bash
kubectl get all -n stateful-demo
```
Check Pods:
```bash
kubectl get pods -n stateful-demo -o wide
```
Check StatefulSet:
```bash
kubectl get statefulset -n stateful-demo
```
Check PVCs:
```bash
kubectl get pvc -n stateful-demo
```
Check PVs:
```bash
kubectl get pv
```
Check StorageClass:
```bash
kubectl get storageclass
```
Check Services:
```bash
kubectl get svc -n stateful-demo
```
Describe a Pod:
```bash
kubectl describe pod mysql-sts-0 -n stateful-demo
```
Check logs:
```bash
kubectl logs mysql-sts-0 -n stateful-demo
```
---
22. Cleanup
Delete the StatefulSet:
```bash
kubectl delete statefulset mysql-sts -n stateful-demo
```
Delete the Service:
```bash
kubectl delete service mysql -n stateful-demo
```
Delete the PVCs:
```bash
kubectl delete pvc -n stateful-demo --all
```
Delete the StorageClass:
```bash
kubectl delete storageclass manual
```
Delete the PVs:
```bash
kubectl delete pv mysql-pv-0 mysql-pv-1
```
Delete the namespace:
```bash
kubectl delete namespace stateful-demo
```
---
23. Final Concept
Remember this simple flow:
```text
StorageClass
     ↓
PV
     ↓
PVC
     ↓
StatefulSet
     ↓
Pod
     ↓
Application
```
For our example:
```text
StorageClass
     ↓
Static PV
     ↓
PVC
     ↓
mysql-sts-0
     ↓
MySQL
```
And:
```text
StorageClass
     ↓
Static PV
     ↓
PVC
     ↓
mysql-sts-1
     ↓
MySQL
```
Main idea
> **PV = Storage**
> **PVC = Request for storage**
> **StorageClass = Storage type/rules**
> **StatefulSet = Manages Pods that need stable identity and storage**
> **Headless Service = Helps give Pods stable network identity**
---
Important Note About Local Storage
We are using local storage:
```text
Worker 1 → /mnt/mysql-0
Worker 2 → /mnt/mysql-1
```
The data physically exists on those workers.
If Worker 1 goes down, Kubernetes cannot automatically move the data from:
```text
Worker 1
   ↓
/mnt/mysql-0
```
to Worker 2.
So this setup is excellent for learning PV, PVC, StorageClass and StatefulSet, but it is not a complete production high-availability storage solution.
