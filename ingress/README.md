Basic Kubernetes Ingress Example (kubeadm)
A beginner-friendly, path-based Ingress example for a kubeadm cluster
using the community NGINX Ingress Controller.
Architecture
``` text
Browser
  |
  | http://<NODE-IP>:<NODEPORT>/nginx
  v
NGINX Ingress Controller
  |
  +--- /nginx  ---> nginx-svc  ---> NGINX Pods
  |
  +--- /apache ---> apache-svc ---> Apache Pods
```
URL path    Destination
---
`/nginx`    NGINX application
`/apache`   Apache application
1. Install NGINX Ingress Controller
If the controller is already installed, skip this step.
> This example uses the community ingress-nginx controller's bare-metal
> provider manifest.
``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/baremetal/deploy.yaml
```
Check the controller and its Service:
``` bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```
Note the HTTP NodePort shown for `ingress-nginx-controller`. The
assigned port may vary.
2. Create the applications and Services
Create a file named `apps.yaml`:
``` yaml
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
Apply and verify:
``` bash
kubectl apply -f apps.yaml
kubectl get deployments
kubectl get pods
kubectl get svc
```
3. Create the Ingress resource
Create a file named `ingress.yaml`.
The rewrite annotation strips the `/nginx` or `/apache` prefix before
forwarding the request to the application.
``` yaml
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
Apply and inspect:
``` bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress basic-ingress
```
4. Find the Ingress NodePort
``` bash
kubectl get svc -n ingress-nginx
```
Example output (your port may be different):
``` text
NAME                       TYPE       PORT(S)
ingress-nginx-controller   NodePort   80:30080/TCP
```
In this example, the HTTP NodePort is `30080`.
5. Test the Ingress
Replace `<NODE-IP>` with the IP address of a node that can receive
traffic on the NodePort, and replace `30080` if your controller uses a
different port.
``` bash
curl http://<NODE-IP>:30080/nginx
curl http://<NODE-IP>:30080/apache
```
Open the same URLs in a browser:
`http://<NODE-IP>:30080/nginx`
`http://<NODE-IP>:30080/apache`
Expected result:
`/nginx` displays the NGINX welcome page.
`/apache` displays the Apache welcome page.
6. Request flow
Example request: `/apache`
The browser sends a request to `/apache`.
The Ingress Controller matches the path and rewrites it to `/`.
The controller forwards the request to `apache-svc`.
The Service routes the request to one of the Apache Pods.
The Apache Pod returns its welcome page.
Important notes
An Ingress resource defines HTTP/HTTPS routing rules; it does not
handle traffic by itself. An Ingress Controller must be installed
and running.
The application Services are `ClusterIP` Services and do not need to
be exposed directly outside the cluster.
Ensure the relevant AWS Security Group or host firewall allows
traffic to the controller's NodePort.
The Ingress class `nginx` must match the installed controller's
IngressClass.
