---
description: >-
  Learn about how the various core concepts of Traefik work together in
  Kubernetes
---

# Kubernetes

Learn how to use Traefik's native **IngressRoute CRD** to manage traffic in a Kubernetes cluster:

1. &#x20;Deploy `whoami` protected by basic auth
2. Add `httpbin` protected by rate limiting

### Before you begin

* [Docker](https://docs.docker.com/get-docker/) installed
* [k3d](https://k3d.io/#installation) installed
* [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
* [helm](https://helm.sh/docs/intro/install/) installed
* `curl` and `htpasswd` available in your terminal

> **Install htpasswd if missing:**
>
> * Linux: `sudo apt-get install apache2-utils`
> * macOS: `brew install httpd`

### Deploy Traefik and whoami with Basic Auth

1. Create a k3d cluster

Create a cluster with port 80 exposed on your local machine:

```bash
k3d cluster create traefik-demo \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"
```

Verify the cluster is running:

```bash
kubectl get nodes
```

Output is similar to:

```
NAME                        STATUS   ROLES                  AGE
k3d-traefik-demo-server-0   Ready    control-plane,master   30s
```

Verify Traefik is running . k3d ships with Traefik by default:

```bash
kubectl get pods -n kube-system | grep traefik
```

Output is similar to:

```
traefik-xxx   1/1   Running   0   60s
```

2\. Add whoami.localhost to /etc/hosts

```bash
echo "127.0.0.1 whoami.localhost" | sudo tee -a /etc/hosts
```

3. Deploy whoami

Save the following as `01-whoami-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: default
spec:
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f 01-whoami-deployment.yaml
```

Verify the Pod is running:

```bash
kubectl get pods
```

Output is similar to:

```
NAME                      READY   STATUS    RESTARTS
whoami-xxx                1/1     Running   0
```

4. Create the Basic Auth secret

a. Generate a bcrypt hash for user `admin` with password `admin123`:

```bash
htpasswd -nb admin admin123
```

Save the output. It will look something like:

```
admin:$apr1$xyz123$hashedpassword
```

b. Save the following as `02-basic-auth-secret.yaml`.

Replace `<YOUR_HASH>` with the full output from the command above:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
  namespace: default
type: Opaque
stringData:
  users: "admin:<YOUR_HASH>"
```

Apply it:

```bash
kubectl apply -f 02-basic-auth-secret.yaml
```

Save the following as `03-basic-auth-middleware.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: default
spec:
  basicAuth:
    secret: basic-auth-secret
```

Apply it:

```bash
kubectl apply -f 03-basic-auth-middleware.yaml
```

Verify the middleware is created:

```bash
kubectl get middleware
```

Output is similar to:

```
NAME         AGE
basic-auth   5s
```

5\. Create the IngressRoute for whoami

Save the following as `04-whoami-ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami-ingressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`whoami.localhost`)
      kind: Rule
      middlewares:
        - name: basic-auth
          namespace: default
      services:
        - name: whoami
          port: 80
```

Apply it:

```bash
kubectl apply -f 04-whoami-ingressroute.yaml
```

Verify the IngressRoute is created:

```bash
kubectl get ingressroute
```

Expected output:

```
NAME                  AGE
whoami-ingressroute   5s
```

#### Test basic auth on whoami

**No credentials — expect `401 Unauthorized`:**

```bash
curl -v http://whoami.localhost
```

**Wrong password — expect `401 Unauthorized`:**

```bash
curl -v -u admin:wrongpassword http://whoami.localhost
```

**Correct credentials — expect `200 OK`:**

```bash
curl -v -u admin:admin123 http://whoami.localhost
```

A successful response looks like this:

```
Hostname: whoami-xxx
IP: 10.42.0.5
GET / HTTP/1.1
Host: whoami.localhost
X-Forwarded-For: 10.42.0.1
X-Forwarded-Host: whoami.localhost
...
```

This confirms, `whoami` is running on k3d and protected by basic auth.

### Deploy httpbin with Rate Limiting

No teardown needed. You will add `httpbin` to the running cluster.

1. Add httpbin.localhost to /etc/hosts

```bash
echo "127.0.0.1 httpbin.localhost" | sudo tee -a /etc/hosts
```

2\. Deploy httpbin

Save the following as `05-httpbin-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: mccutchen/go-httpbin
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: default
spec:
  selector:
    app: httpbin
  ports:
    - port: 80
      targetPort: 8080
```

Apply it:

```bash
kubectl apply -f 05-httpbin-deployment.yaml
```

Verify both pods are running:

```bash
kubectl get pods
```

Expected output:

```
NAME                      READY   STATUS    RESTARTS
whoami-xxx                1/1     Running   0
httpbin-xxx               1/1     Running   0
```

3\. Create the Rate Limit middleware

Save the following as `06-ratelimit-middleware.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 5
    burst: 10
```

Apply it:

```bash
kubectl apply -f 06-ratelimit-middleware.yaml
```

Verify both middlewares are present:

```bash
kubectl get middleware
```

Expected output:

```
NAME         AGE
basic-auth   10m
rate-limit   5s
```

4\. Create the IngressRoute for httpbin

Save the following as `07-httpbin-ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: httpbin-ingressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`httpbin.localhost`)
      kind: Rule
      middlewares:
        - name: rate-limit
          namespace: default
      services:
        - name: httpbin
          port: 80
```

Apply it:

```bash
kubectl apply -f 07-httpbin-ingressroute.yaml
```

Verify both IngressRoutes are present:

```bash
kubectl get ingressroute
```

Expected output:

```
NAME                   AGE
whoami-ingressroute    10m
httpbin-ingressroute   5s
```

5\. Test rate limiting on httpbin

**Single request — expect `200 OK`:**

```bash
curl -v http://httpbin.localhost/get
```

**Burst test — fire 15 rapid requests and watch for `429 Too Many Requests`:**

```bash
for i in $(seq 1 15); do
  echo -n "Request $i: "
  curl -s -o /dev/null -w "%{http_code}\n" http://httpbin.localhost/get
done
```

Expected output:

```
Request 1: 200
Request 2: 200
Request 3: 200
...
Request 11: 429
Request 12: 429
Request 13: 429
Request 14: 429
Request 15: 429
```

6\. Confirm whoami is unaffected

```bash
curl -v -u admin:admin123 http://whoami.localhost
```

Should still return `200 OK`.

`httpbin` is running and protected by rate limiting.

### Tear down

```bash
k3d cluster delete traefik-demo
```

Remove the `/etc/hosts` entries:

```bash
sudo sed -i '' '/whoami.localhost/d' /etc/hosts
sudo sed -i '' '/httpbin.localhost/d' /etc/hosts
```
