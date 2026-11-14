# EFK Stack Demo on Kubernetes Using Helm

A simple Kubernetes logging demo using **Nginx, Elasticsearch, Fluent
Bit, and Kibana**.

This guide deploys one Nginx application and uses Helm to install the
logging components. Nginx generates logs, Fluent Bit collects them,
Elasticsearch stores them, and Kibana provides a web interface to search
and view them.

> **Demo storage:** Elasticsearch uses temporary `emptyDir` storage.
> This setup does not create a PersistentVolume (PV),
> PersistentVolumeClaim (PVC), or StorageClass. Elasticsearch data can
> be lost if its pod is recreated. Use this configuration only for a
> temporary demo, not production.

------------------------------------------------------------------------

## 1. Architecture

``` text
                 Browser / curl
                       |
                       v
                Nginx Service
                       |
                       v
                  Nginx Pod
                (creates logs)
                       |
                       v
             Kubernetes container logs
                       |
                       v
              Fluent Bit DaemonSet
                (collects logs)
                       |
                       v
                Elasticsearch
                 (stores logs)
                       |
                       v
                   Kibana
              (searches and displays)
```

### Components

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  Nginx                               Demo application that generates
                                      access logs

  Fluent Bit                          Collects container logs and
                                      forwards them

  Elasticsearch                       Stores and searches log data

  Kibana                              Web interface for searching and
                                      viewing logs

  ECK Operator                        Manages Elasticsearch and Kibana
                                      resources in Kubernetes

  Helm                                Installs and manages the
                                      applications using charts
  -----------------------------------------------------------------------

## 2. Prerequisites

-   A working Kubernetes cluster.
-   `kubectl` configured to access the cluster.
-   Helm installed.
-   Worker-node network access for the Nginx NodePort.
-   AWS Security Group access configured for the required NodePorts.

Check the cluster:

``` bash
kubectl get nodes
kubectl version
helm version
```

Wait until the nodes show `Ready`.

## 3. Add Helm repositories

Add the Elastic and Fluent Bit chart repositories:

``` bash
helm repo add elastic https://helm.elastic.co
helm repo add fluent https://fluent.github.io/helm-charts

helm repo update
```

Verify:

``` bash
helm repo list
```

## 4. Create the EFK namespace

``` bash
kubectl create namespace efk
```

Verify:

``` bash
kubectl get namespaces
```

## 5. Install the ECK Operator

The ECK Operator manages Elasticsearch and Kibana resources in
Kubernetes.

Install it in its own namespace:

``` bash
helm install elastic-operator elastic/eck-operator \
  --version 3.1.0 \
  -n elastic-system \
  --create-namespace
```

Check the operator:

``` bash
kubectl get pods -n elastic-system
```

Wait until the operator pod is `Running`.

Check the Elastic custom resource definitions:

``` bash
kubectl get crds | grep elastic
```

## 6. Install Elasticsearch and Kibana

Create the Helm values file:

``` bash
nano efk-values.yaml
```

Add:

``` yaml
eck-elasticsearch:
  enabled: true

  nodeSets:
    - name: default
      count: 1

      config:
        node.store.allow_mmap: false

      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 1Gi
                  cpu: 500m
                limits:
                  memory: 2Gi
                  cpu: "1"

          volumes:
            - name: elasticsearch-data
              emptyDir: {}

      volumeClaimTemplates: []

eck-kibana:
  enabled: true

  count: 1

  http:
    service:
      spec:
        type: NodePort
```

### Important settings

  -----------------------------------------------------------------------
  Setting                             Purpose
  ----------------------------------- -----------------------------------
  `count: 1`                          Runs one Elasticsearch node

  `emptyDir: {}`                      Uses temporary pod storage

  `volumeClaimTemplates: []`          Prevents Elasticsearch PVC
                                      templates from being created

  `node.store.allow_mmap: false`      Disables Elasticsearch
                                      memory-mapped storage

  `type: NodePort`                    Exposes the Kibana service through
                                      a node port
  -----------------------------------------------------------------------

Install the ECK Stack chart:

``` bash
helm install efk elastic/eck-stack \
  --version 0.18.0 \
  -n efk \
  -f efk-values.yaml
```

Check the resources:

``` bash
kubectl get pods -n efk
kubectl get elasticsearch -n efk
kubectl get kibana -n efk
kubectl get svc -n efk
```

Check PVCs:

``` bash
kubectl get pvc -n efk
```

No PVC should be created for Elasticsearch with this demo configuration.

**Important:** Wait until Elasticsearch and Kibana are ready before
installing Fluent Bit.

## 7. Deploy Nginx

Nginx is the application that will generate the logs.

Create the manifest:

``` bash
nano nginx.yaml
```

Add:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: efk
spec:
  replicas: 1
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
  namespace: efk
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply the manifest:

``` bash
kubectl apply -f nginx.yaml
```

Check:

``` bash
kubectl get pods -n efk
kubectl get svc -n efk
```

### Generate Nginx logs

Access Nginx using a worker-node IP:

``` bash
curl http://<WORKER-NODE-IP>:30080
```

Run the command several times.

View the logs:

``` bash
kubectl logs deployment/nginx -n efk
```

Nginx should show access-log entries for the requests. At this stage,
the logs are available through Kubernetes container logging. Fluent Bit
will collect them in the next step.

## 8. Get the Elasticsearch password

ECK creates a Kubernetes Secret containing the password for the default
`elastic` user.

Run:

``` bash
kubectl get secret elasticsearch-es-elastic-user -n efk \
  -o jsonpath='{.data.elastic}' | base64 -d
```

Copy the password for the Kibana login. Do not publish the password in a
GitHub repository.

## 9. Install Fluent Bit

Fluent Bit runs as a DaemonSet. This means Kubernetes schedules a Fluent
Bit pod on each eligible node.

It reads container log files from the nodes and forwards matching logs
to Elasticsearch.

Create the values file:

``` bash
nano fluent-bit-values.yaml
```

Add:

``` yaml
kind: DaemonSet

env:
  - name: ES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elasticsearch-es-elastic-user
        key: elastic

config:
  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off

    [FILTER]
        Name grep
        Match kube.*
        Regex $kubernetes['container_name'] ^nginx$

  outputs: |
    [OUTPUT]
        Name es
        Match kube.*
        Host elasticsearch-es-http
        Port 9200
        HTTP_User elastic
        HTTP_Passwd ${ES_PASSWORD}
        Logstash_Format On
        Logstash_Prefix nginx-logs
        Suppress_Type_Name On
        tls On
        tls.verify Off
        Retry_Limit False
```

### Important Fluent Bit settings

  Setting               Purpose
  --------------------- --------------------------------------------------
  `Path`                Reads Kubernetes container log files
  `Tag`                 Assigns a tag to collected records
  `kubernetes` filter   Adds Kubernetes metadata
  `grep` filter         Keeps records from the Nginx container
  `Host`                Elasticsearch Kubernetes service name
  `Port`                Elasticsearch HTTP port
  `HTTP_User`           Elasticsearch username
  `HTTP_Passwd`         Reads the password from the environment variable
  `Logstash_Prefix`     Creates indices with the `nginx-logs` prefix
  `tls On`              Enables TLS for the Elasticsearch connection

> **TLS note:** `tls.verify Off` disables certificate verification. It
> is used here to simplify this temporary demo. Do not use it as a
> production security setting.

Install Fluent Bit:

``` bash
helm install fluent-bit fluent/fluent-bit \
  --version 0.54.0 \
  -n efk \
  -f fluent-bit-values.yaml
```

Check the pods and DaemonSet:

``` bash
kubectl get pods -n efk
kubectl get daemonset -n efk
```

Check Fluent Bit logs:

``` bash
kubectl logs -l app.kubernetes.io/name=fluent-bit -n efk --tail=50
```

If the values file is changed later, apply the changes with:

``` bash
helm upgrade fluent-bit fluent/fluent-bit \
  --version 0.54.0 \
  -n efk \
  -f fluent-bit-values.yaml
```

## 10. Access Kibana

Find the Kibana service and its NodePort:

``` bash
kubectl get svc -n efk
```

Find the service named similar to:

``` text
efk-eck-kibana-kb-http
```

Example output:

``` text
NAME                         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
efk-eck-kibana-kb-http       NodePort   10.109.163.203  <none>        5601:31118/TCP
```

In this example, the Kibana NodePort is `31118`. Your assigned NodePort
may be different.

### AWS Security Group

Allow inbound TCP traffic to the Kibana NodePort on the EC2 instance's
Security Group.

For example:

  Type         Protocol      Port Source
  ------------ ---------- ------- ----------------
  Custom TCP   TCP          31118 Your public IP

Restrict the source to your own public IP rather than opening it to
everyone.

### Open Kibana

ECK enables HTTPS for Kibana. Use **HTTPS**, not HTTP:

``` text
https://<EC2-PUBLIC-IP>:<KIBANA-NODEPORT>
```

Example:

``` text
https://13.207.89.197:31118
```

The certificate may produce a browser security warning because the demo
uses the ECK-generated certificate. For this temporary lab, review the
warning and proceed only if you are connecting to your own lab instance.

### Login

Get the password again if required:

``` bash
kubectl get secret elasticsearch-es-elastic-user -n efk \
  -o jsonpath='{.data.elastic}' | base64 -d
```

Use:

``` text
Username: elastic
Password: <password-from-the-secret>
```

## 11. View Nginx logs in Kibana

Create a Data View in Kibana:

1.  Open **Stack Management**.
2.  Select **Data Views**.
3.  Click **Create data view**.
4.  Enter the following:

``` text
Name: Nginx Logs
Index pattern: nginx-logs-*
Timestamp field: @timestamp
```

5.  Save the Data View.
6.  Open **Analytics → Discover**.
7.  Select the `Nginx Logs` Data View.

You should now be able to search and view the Nginx logs collected by
Fluent Bit and stored in Elasticsearch.

## 12. Generate more logs

Run this from a terminal that has access to the Nginx NodePort:

``` bash
for i in {1..20}; do
  curl -s http://<WORKER-NODE-IP>:30080 > /dev/null
done
```

Return to Kibana Discover and refresh the data.

The new Nginx requests should appear after Fluent Bit has collected and
forwarded them.

## 13. Complete log flow

1.  A user accesses the Nginx website.
2.  Nginx generates an access log.
3.  Kubernetes makes the container log available on the node.
4.  Fluent Bit reads the container log.
5.  Fluent Bit forwards the log to Elasticsearch.
6.  Elasticsearch stores the log in an index beginning with
    `nginx-logs-`.
7.  Kibana searches and displays the log.

## 14. Verification commands

Check all pods:

``` bash
kubectl get pods -n efk
```

Check deployments:

``` bash
kubectl get deployments -n efk
```

Check DaemonSets:

``` bash
kubectl get daemonsets -n efk
```

Check services:

``` bash
kubectl get svc -n efk
```

Check Elasticsearch:

``` bash
kubectl get elasticsearch -n efk
```

Check Kibana:

``` bash
kubectl get kibana -n efk
```

Confirm that no PVCs were created in the namespace:

``` bash
kubectl get pvc -n efk
```

## 15. Troubleshooting

  -----------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------
  Elasticsearch pod is not starting   Pod events, available memory, and
                                      Elasticsearch logs

  Kibana is not ready                 Elasticsearch readiness and Kibana
                                      pod logs

  Fluent Bit is not running           DaemonSet status and Fluent Bit pod
                                      logs

  Logs are not appearing              Fluent Bit input, filters, output,
                                      and Elasticsearch connectivity

  Elasticsearch connection error      Service name, credentials, and TLS
                                      settings

  Kibana is not opening               Use HTTPS, check the NodePort, and
                                      check the AWS Security Group

  Kibana shows no records             Data View pattern and selected time
                                      range
  -----------------------------------------------------------------------

Useful commands:

``` bash
kubectl describe pod <POD-NAME> -n efk
```

``` bash
kubectl logs <POD-NAME> -n efk
```

``` bash
kubectl get events -n efk --sort-by=.metadata.creationTimestamp
```

## 16. Cleanup

Remove the Helm releases and demo resources when the lab is finished.

``` bash
helm uninstall fluent-bit -n efk
helm uninstall efk -n efk
kubectl delete -f nginx.yaml
```

If the ECK Operator was installed only for this demo and is not used by
other Elastic resources, remove it too:

``` bash
helm uninstall elastic-operator -n elastic-system
kubectl delete namespace elastic-system
```

Finally, remove the EFK namespace if it still exists:

``` bash
kubectl delete namespace efk
```

> Deleting the namespace removes the demo resources and the temporary
> Elasticsearch data.

------------------------------------------------------------------------

## Notes

-   This is a learning/demo configuration, not a production deployment.
-   Elasticsearch uses temporary `emptyDir` storage. Data is not
    persistent.
-   No PV, PVC, or StorageClass is configured for this demo.
-   Keep Elasticsearch and Kibana protected. Do not expose them publicly
    without appropriate network restrictions and security controls.
-   Do not commit Elasticsearch passwords, Kubernetes Secrets, or other
    credentials to GitHub.
