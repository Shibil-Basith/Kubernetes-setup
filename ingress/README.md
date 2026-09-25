# Basic Kubernetes Ingress Example (kubeadm)

A beginner-friendly, path-based Ingress example for a kubeadm cluster using the NGINX Ingress Controller.

## Architecture

```text
                  Browser
                     |
                     | http://<NODE-IP>:<NODEPORT>
                     v
            NGINX Ingress Controller
                     |
           +---------+---------+
           |                   |
      /nginx                /apache
           |                   |
           v                   v
       nginx-svc           apache-svc
           |                   |
           v                   v
       NGINX Pods          Apache Pods
```

| URL Path | Destination |
|---|---|
| `/nginx` | NGINX application |
| `/apache` | Apache application |

---

## 1. Install NGINX Ingress Controller

If the controller is already installed, skip this step.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/baremetal/deploy.yaml
```

Check the controller:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Note the HTTP NodePort assigned to `ingress-nginx-controller`.

---

## 2. Create Applications and Services

Create a file named `apps.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
        - name: apache
          image: httpd:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: apache-svc
spec:
  type: ClusterIP
  selector:
    app: apache
  ports:
    - port: 80
      targetPort: 80
```

Apply the configuration:

```bash
kubectl apply -f apps.yaml
```

Verify:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

---

## 3. Create the Ingress Resource

Create a file named `ingress.yaml`.

The rewrite annotation removes the `/nginx` or `/apache` prefix before forwarding the request to the application.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /nginx(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
          - path: /apache(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: apache-svc
                port:
                  number: 80
```

Apply the configuration:

```bash
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl get ingress
kubectl describe ingress basic-ingress
```

---

## 4. Find the Ingress NodePort

```bash
kubectl get svc -n ingress-nginx
```

Example output:

```text
NAME                       TYPE       PORT(S)
ingress-nginx-controller   NodePort   80:30080/TCP
```

In this example, the HTTP NodePort is `30080`.

Your assigned port may be different.

---

## 5. Test the Ingress

Replace `<NODE-IP>` with the IP address of a node that can receive traffic on the NodePort.

### Test NGINX

```bash
curl http://<NODE-IP>:30080/nginx
```

### Test Apache

```bash
curl http://<NODE-IP>:30080/apache
```

### Test using a browser

```text
http://<NODE-IP>:30080/nginx
http://<NODE-IP>:30080/apache
```

### Expected results

- `/nginx` displays the NGINX welcome page.
- `/apache` displays the Apache welcome page.

---

## 6. Request Flow

Example: `http://<NODE-IP>:30080/apache`

1. The browser sends a request to `/apache`.
2. The Ingress Controller matches the path.
3. The controller rewrites `/apache` to `/`.
4. The request is forwarded to `apache-svc`.
5. The Service routes the request to an Apache Pod.
6. The Apache Pod returns its welcome page.

---

## Important Notes

- An Ingress resource defines HTTP/HTTPS routing rules. An Ingress Controller must be installed and running to handle the traffic.
- The application Services are `ClusterIP` Services and do not need to be exposed directly outside the cluster.
- Ensure the AWS Security Group or host firewall allows traffic to the controller's NodePort.
- The `ingressClassName: nginx` must match the IngressClass of your installed controller.
- Replace `30080` with your actual HTTP NodePort.
