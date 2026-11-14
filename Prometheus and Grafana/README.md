# Prometheus and Grafana on Kubernetes Using Helm

Kubernetes monitoring demo using **Prometheus,
Grafana, and Nginx**.

This guide uses the `kube-prometheus-stack` Helm chart. It installs
Prometheus, Grafana, Prometheus Operator, Node Exporter, and Kube State
Metrics.

The demo monitors Kubernetes cluster health and resource usage,
including the Nginx pod's CPU and memory usage.

> **Demo storage:** This setup does not configure a PersistentVolume
> (PV), PersistentVolumeClaim (PVC), or StorageClass. Prometheus and
> Grafana use temporary storage. Metrics and dashboards' stored state
> may be lost when their pods are recreated. This is for a temporary
> demo, not production.

------------------------------------------------------------------------

## 1. What are we going to build?

``` text
                  Kubernetes Cluster
                          |
             +------------+------------+
             |            |            |
             v            v            v
        Node Exporter  Kube State   Nginx Pod
        (node metrics)  Metrics     (application)
             |            |            |
             +------------+------------+
                          |
                          v
                     Prometheus
                 (collects metrics)
                          |
                          v
                       Grafana
                 (dashboards and charts)
```

### Components

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  Prometheus                          Collects and stores metrics

  Grafana                             Displays metrics in dashboards

  Prometheus Operator                 Manages Prometheus resources in
                                      Kubernetes

  Node Exporter                       Collects CPU, memory, disk, and
                                      other Linux node metrics

  Kube State Metrics                  Exposes information about
                                      Kubernetes objects such as Pods and
                                      Deployments

  Nginx                               Demo application to monitor
  -----------------------------------------------------------------------

**Important:** The default Nginx image does not expose detailed Nginx
HTTP metrics such as request count or response time. This demo monitors
Kubernetes and node metrics, including Nginx pod status and resource
usage. Nginx-specific HTTP metrics require an additional Nginx
Prometheus exporter.

## 2. Prerequisites

-   A working Kubernetes cluster.
-   `kubectl` configured to access the cluster.
-   Helm 3 installed.
-   Enough CPU and memory for the monitoring components.
-   Network access to the selected NodePorts.

Check the cluster:

``` bash
kubectl get nodes
kubectl version
helm version
```

Wait until the nodes show `Ready`.

## 3. Add the Prometheus Helm repository

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update
```

Verify:

``` bash
helm repo list
```

## 4. Create a monitoring namespace

``` bash
kubectl create namespace monitoring
```

Verify:

``` bash
kubectl get namespaces
```

We will install the monitoring components and deploy Nginx in this
namespace to keep the demo simple.

## 5. Create the Helm values file

We will expose Grafana and Prometheus using NodePort services.

Create the file:

``` bash
nano monitoring-values.yaml
```

Add:

``` yaml
grafana:
  enabled: true

  service:
    type: NodePort
    nodePort: 30300

  persistence:
    enabled: false

prometheus:
  enabled: true

  service:
    type: NodePort
    nodePort: 30090

  prometheusSpec:
    retention: 2d
    storageSpec: {}
    replicas: 1

alertmanager:
  enabled: true

  alertmanagerSpec:
    storage: {}
```

### Understand the important settings

  --------------------------------------------------------------------------
  Setting                                Meaning
  -------------------------------------- -----------------------------------
  `grafana.enabled`                      Installs Grafana

  `grafana.service.type: NodePort`       Exposes Grafana through a
                                         Kubernetes node port

  `nodePort: 30300`                      Grafana access port

  `grafana.persistence.enabled: false`   Does not create Grafana persistent
                                         storage

  `prometheus.enabled`                   Installs Prometheus

  `nodePort: 30090`                      Prometheus access port

  `retention: 2d`                        Keeps metrics for two days

  `storageSpec: {}`                      Does not configure a Prometheus PVC

  `replicas: 1`                          Runs one Prometheus replica

  `alertmanager.enabled`                 Installs Alertmanager
  --------------------------------------------------------------------------

The chart also installs Node Exporter and Kube State Metrics by default.

## 6. Install Prometheus and Grafana using Helm

Install the `kube-prometheus-stack` chart:

``` bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --version 92.8.0 \
  -n monitoring \
  -f monitoring-values.yaml \
  --set grafana.adminPassword='ChangeThisPassword!'
```

**Important:** Replace `ChangeThisPassword!` with your own password. Do
not commit a real password or Kubernetes Secret to GitHub.

Check the installation:

``` bash
kubectl get pods -n monitoring
```

Wait until the pods become `Running` and ready.

Check Helm:

``` bash
helm list -n monitoring
```

Check the services:

``` bash
kubectl get svc -n monitoring
```

You should find services for Grafana and Prometheus.

Check PVCs:

``` bash
kubectl get pvc -n monitoring
```

No PVC should be created by this demo configuration.

## 7. Understand the installed components

Run:

``` bash
kubectl get pods -n monitoring
```

You should find pods with names similar to:

``` text
monitoring-grafana-...
monitoring-kube-prometheus-stack-operator-...
prometheus-monitoring-kube-prometheus-prometheus-0
monitoring-kube-state-metrics-...
monitoring-prometheus-node-exporter-...
```

Names may vary slightly.

### What does each component do?

**Node Exporter**

-   Runs as a DaemonSet.
-   Collects Linux node metrics.
-   Provides CPU, memory, disk, and network information.

Check:

``` bash
kubectl get daemonset -n monitoring
```

**Kube State Metrics**

-   Provides information about Kubernetes resources.
-   Reports Pod, Deployment, and ReplicaSet status.
-   Helps Prometheus understand the state of Kubernetes objects.

Check:

``` bash
kubectl get deployment -n monitoring
```

**Prometheus Operator**

-   Manages Prometheus and related monitoring resources.
-   Watches Kubernetes monitoring resources.
-   Helps configure metric collection.

**Prometheus**

-   Scrapes metrics from configured targets.
-   Stores the collected metrics.
-   Allows us to run PromQL queries.

**Grafana**

-   Connects to Prometheus.
-   Displays metrics using dashboards and graphs.

## 8. Deploy Nginx

Now we need an application to monitor.

Create the Nginx manifest:

``` bash
nano nginx.yaml
```

Add:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: monitoring
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
          image: nginx:1.27
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

``` bash
kubectl apply -f nginx.yaml
```

Check:

``` bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Access Nginx:

``` bash
curl http://<WORKER-NODE-IP>:30080
```

You should receive the default Nginx welcome page HTML.

## 9. Access Prometheus

Check the services:

``` bash
kubectl get svc -n monitoring
```

Find the Prometheus service. Its name should be similar to:

``` text
monitoring-kube-prometheus-prometheus
```

The NodePort is `30090` in our values file.

### AWS Security Group

Allow inbound traffic to the monitoring EC2 instance:

  Type         Protocol      Port Source
  ------------ ---------- ------- ----------------
  Custom TCP   TCP          30090 Your public IP
  Custom TCP   TCP          30300 Your public IP
  Custom TCP   TCP          30080 Your public IP

Use restricted source IP addresses for the demo instead of exposing
these services to everyone.

### Open Prometheus

``` text
http://<EC2-PUBLIC-IP>:30090
```

Open:

**Status → Targets**

You should find monitoring targets such as:

-   Node Exporter
-   Kube State Metrics
-   Kubernetes API server
-   Kubelet
-   Prometheus itself

The exact targets depend on your Kubernetes environment and the
components that expose metrics.

### Demo 1 -- Check whether Prometheus is working

Open the Prometheus expression browser and enter:

``` promql
up
```

Click **Execute**.

This query shows whether Prometheus can successfully scrape its
configured targets:

-   `1` means the target is up.
-   `0` means the target is down.

Try:

``` promql
kube_pod_info
```

This displays information about Kubernetes Pods discovered by Kube State
Metrics.

## 10. Access Grafana

Open:

``` text
http://<EC2-PUBLIC-IP>:30300
```

Login:

``` text
Username: admin
Password: <password-configured-during-installation>
```

The password is the one supplied using `grafana.adminPassword` during
Helm installation.

### Demo 2 -- Explore Grafana dashboards

After login:

1.  Open **Dashboards**.
2.  Browse the dashboards installed by the Helm chart.
3.  Open the Kubernetes-related dashboards.
4.  Explore node CPU and memory usage.
5.  Check Pod and Deployment information.

The chart automatically configures Prometheus as Grafana's data source
and provisions dashboards.

## 11. Monitor Nginx pod status

We have deployed Nginx with two replicas.

Check the Pods:

``` bash
kubectl get pods -n monitoring
```

In Grafana, open **Explore**.

Select the Prometheus data source.

Run this PromQL query:

``` promql
kube_pod_info{namespace="monitoring",pod=~"nginx-.*"}
```

This helps identify the Nginx Pods being reported by Kube State Metrics.

### Check whether Nginx Pods are running

``` promql
kube_pod_status_phase{
  namespace="monitoring",
  pod=~"nginx-.*",
  phase="Running"
}
```

A value of `1` indicates a running Pod.

## 12. Monitor Nginx CPU usage

Run this PromQL query in Grafana Explore:

``` promql
sum by (pod) (
  rate(
    container_cpu_usage_seconds_total{
      namespace="monitoring",
      pod=~"nginx-.*",
      container!="",
      image!=""
    }[5m]
  )
)
```

This query shows the CPU usage rate for the Nginx containers.

## 13. Monitor Nginx memory usage

Run:

``` promql
container_memory_working_set_bytes{
  namespace="monitoring",
  pod=~"nginx-.*",
  container!="",
  image!=""
}
```

This query displays the current working memory usage of the Nginx
containers.

## 14. Demo -- Delete an Nginx Pod

This is a useful way to demonstrate Kubernetes monitoring.

First, check the Nginx Pods:

``` bash
kubectl get pods -n monitoring
```

Delete one Nginx Pod:

``` bash
kubectl delete pod <NGINX-POD-NAME> -n monitoring
```

Now check again:

``` bash
kubectl get pods -n monitoring
```

Kubernetes creates a replacement Pod because the Deployment maintains
the desired replica count.

Go back to Grafana and refresh the relevant Pod dashboard or query.

Explain:

1.  We manually deleted an Nginx Pod.
2.  Kubernetes detected that the desired number of replicas was not
    available.
3.  The Deployment created a replacement Pod.
4.  Kube State Metrics exposed the updated Pod state.
5.  Prometheus collected the metrics.
6.  Grafana displayed the monitoring information.

Metrics are collected at intervals, so a short-lived event may not
appear in every graph.

## 15. Useful PromQL queries

Try these queries in Prometheus or Grafana Explore.

### Check all scrape targets

``` promql
up
```

### Check node CPU usage

``` promql
100 * (
  1 - avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

### Check node memory usage percentage

``` promql
100 * (
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

### Check running Pods

``` promql
kube_pod_status_phase{phase="Running"}
```

### Check Nginx Pod information

``` promql
kube_pod_info{namespace="monitoring",pod=~"nginx-.*"}
```

## 16. Final verification

Check all Pods:

``` bash
kubectl get pods -n monitoring
```

Check Deployments:

``` bash
kubectl get deployments -n monitoring
```

Check DaemonSets:

``` bash
kubectl get daemonsets -n monitoring
```

Check services:

``` bash
kubectl get svc -n monitoring
```

Check monitoring resources:

``` bash
kubectl get prometheus -n monitoring
kubectl get servicemonitors -n monitoring
```

Check PVCs:

``` bash
kubectl get pvc -n monitoring
```

No PVC should be created by this demo configuration.

## 17. Troubleshooting

  -----------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------
  Prometheus Pod is not starting      Pod events, available memory, and
                                      Prometheus logs

  Grafana is not opening              NodePort, AWS Security Group, and
                                      Grafana Pod

  Prometheus targets are down         Target status, network access, and
                                      exporter Pods

  Grafana shows no data               Prometheus data source and selected
                                      time range

  Nginx Pod metrics are missing       Kube State Metrics,
                                      kubelet/cAdvisor scraping, and
                                      query labels

  PVCs are created unexpectedly       Helm values and existing resources

  Helm installation fails             Helm output, CRDs, and Kubernetes
                                      events
  -----------------------------------------------------------------------

Useful commands:

``` bash
kubectl get events -n monitoring \
  --sort-by=.metadata.creationTimestamp
```

``` bash
kubectl describe pod <POD-NAME> -n monitoring
```

``` bash
kubectl logs <POD-NAME> -n monitoring
```

Check Helm release status:

``` bash
helm status monitoring -n monitoring
```

## 18. Cleanup

Remove the Nginx application:

``` bash
kubectl delete -f nginx.yaml
```

Uninstall the monitoring stack:

``` bash
helm uninstall monitoring -n monitoring
```

Delete the namespace:

``` bash
kubectl delete namespace monitoring
```

**CRD note:** Helm does not automatically remove the Prometheus Operator
CRDs when the chart is uninstalled. They may be used by other monitoring
releases. Do not delete them unless you have confirmed that no other
resources depend on them.

------------------------------------------------------------------------

## 19. Demo checklist

Use this checklist to follow the practical sequence:

-   [ ] Check Kubernetes cluster and Helm.
-   [ ] Add the Prometheus Community Helm repository.
-   [ ] Create the `monitoring` namespace.
-   [ ] Create `monitoring-values.yaml`.
-   [ ] Install `kube-prometheus-stack`.
-   [ ] Verify Prometheus, Grafana, Node Exporter, and Kube State
    Metrics.
-   [ ] Deploy Nginx with two replicas.
-   [ ] Access the Prometheus UI.
-   [ ] Check Prometheus targets.
-   [ ] Run the `up` PromQL query.
-   [ ] Access Grafana.
-   [ ] Explore Kubernetes dashboards.
-   [ ] Check Nginx Pod status.
-   [ ] Check Nginx CPU and memory usage.
-   [ ] Delete an Nginx Pod and observe its replacement.

## 20. Important notes

-   This is a temporary learning/demo setup, not a production monitoring
    deployment.
-   No PV, PVC, or StorageClass is configured.
-   Prometheus metrics and Grafana state are not guaranteed to survive
    pod recreation.
-   The chart installs additional components such as Alertmanager, Node
    Exporter, and Kube State Metrics.
-   Restrict access to Prometheus and Grafana using appropriate network
    rules.
-   Never commit real passwords or Kubernetes Secrets to GitHub.
-   Nginx HTTP request metrics require an additional exporter; the
    queries in this guide focus on Kubernetes pod and node metrics.

## References

-   [Prometheus Community Helm
    Charts](https://github.com/prometheus-community/helm-charts)
-   [kube-prometheus-stack Chart
    Documentation](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
-   [Prometheus
    Documentation](https://prometheus.io/docs/introduction/overview/)
-   [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
-   [Node Exporter](https://github.com/prometheus/node_exporter)
-   [Kube State
    Metrics](https://github.com/kubernetes/kube-state-metrics)
