---
description: >-
  Learn about how the various core concepts of Traefik work together in
  Kubernetes
---

# Kubernetes

Learn how to use Traefik's native **IngressRoute CRD** to manage traffic in a Kubernetes cluster:

1. [Deploy `whoami` protected by basic auth](kubernetes.md#deploy-traefik-and-whoami-with-basic-auth)
2. [Verify that Traefik adds authentication for the `whoami` service](kubernetes.md#verify-basic-auth-for-whoami-service)
3. [Add `httpbin` protected by rate limiting](kubernetes.md#deploy-httpbin-with-rate-limiting)
4. [Verify that Traefik sets rate limiting for `httpb`](kubernetes.md#verify-rate-limiting-on-httpbin)

### Before you begin

* [Docker](https://docs.docker.com/get-docker/) installed
* [k3d](https://k3d.io/#installation) installed
* [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
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
NAME                        STATUS   ROLES                  AGE   VERSION
k3d-traefik-demo-server-0   Ready    control-plane,master   15s   v1.33.6+k3s1
```

Verify Traefik is running . k3d ships with Traefik by default:

```bash
kubectl get pods -n kube-system | grep traefik
```

Output is similar to:

```
helm-install-traefik-c5g4t                0/1     Completed   1          91s
helm-install-traefik-crd-mlxqc            0/1     Completed   0          91s
svclb-traefik-95f1b584-h9jwt              2/2     Running     0          41s
traefik-865bd56545-8wlrl                  1/1     Running     0          41ss
```

2\. Add whoami.localhost to /etc/hosts

```bash
echo "127.0.0.1 whoami.localhost" | sudo tee -a /etc/hosts
```

3. Deploy `whoami service:`

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
NAME                      READY   STATUS    RESTARTS   AGE
whoami-64f6cf779d-kgztc   1/1     Running   0          20s
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

c. Save the following as `03-basic-auth-middleware.yaml`:

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
basic-auth   6s
```

5\. Create the IngressRoute for `whoami service.`

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

Output is similar to:

```
NAME                  AGE
whoami-ingressroute   5s
```

### Verify Basic Auth for \`whoami\` service

1. Access the `whoami` service with no credentials:

```bash
curl -v http://whoami.localhost
```

Output is similar to:

```
* Host whoami.localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to whoami.localhost (::1) port 80
> GET / HTTP/1.1
> Host: whoami.localhost
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: text/plain
< Www-Authenticate: Basic realm="traefik"
< Date: Mon, 23 Mar 2026 12:31:12 GMT
< Content-Length: 17
< 
401 Unauthorized
* Connection #0 to host whoami.localhost left intact
```

2. Access the `whoami` service with wrong password:

```bash
curl -v -u admin:wrongpassword http://whoami.localhost
```

Output is similar to:

```
* Host whoami.localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to whoami.localhost (::1) port 80
* Server auth using Basic with user 'admin'
> GET / HTTP/1.1
> Host: whoami.localhost
> Authorization: Basic YWRtaW46d3JvbmdwYXNzd29yZA==
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: text/plain
* Authentication problem. Ignoring this.
< Www-Authenticate: Basic realm="traefik"
< Date: Mon, 23 Mar 2026 12:31:24 GMT
< Content-Length: 17
< 
401 Unauthorized
* Connection #0 to host whoami.localhost left intact
```

3. Access the `whoami` service with correct credentials. If you used a different password, then ensure that you replace `admin123` with the credentials that you set when creating the basic auth password.

```bash
curl -v -u admin:admin123 http://whoami.localhost
```

Output is similar to:

```
* Host whoami.localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to whoami.localhost (::1) port 80
* Server auth using Basic with user 'admin'
> GET / HTTP/1.1
> Host: whoami.localhost
> Authorization: Basic YWRtaW46YWRtaW4xMjM=
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Content-Length: 452
< Content-Type: text/plain; charset=utf-8
< Date: Mon, 23 Mar 2026 12:31:31 GMT
< 
Hostname: whoami-64f6cf779d-kgztc
IP: 127.0.0.1
IP: ::1
IP: 10.42.0.9
IP: fe80::b44c:f1ff:fe51:dc58
RemoteAddr: 10.42.0.8:56228
GET / HTTP/1.1
Host: whoami.localhost
User-Agent: curl/8.6.0
Accept: */*
Accept-Encoding: gzip
Authorization: Basic YWRtaW46YWRtaW4xMjM=
X-Forwarded-For: 10.42.0.1
X-Forwarded-Host: whoami.localhost
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-865bd56545-8wlrl
X-Real-Ip: 10.42.0.1

* Connection #0 to host whoami.localhost left intact

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

Output is similar to:

```
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-57956b77c7-m6t6c   1/1     Running   0          13s
whoami-64f6cf779d-kgztc    1/1     Running   0          5m41s
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

Output is similar to:

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

Output is similar to:

```
NAME                   AGE
httpbin-ingressroute   7s
whoami-ingressroute    3m32s
```

### Verify  rate limiting on httpbin service:

1. Send a Single request :

```bash
curl -v http://httpbin.localhost/get
```

Output is similar to:

```
* Host httpbin.localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to httpbin.localhost (::1) port 80
> GET /get HTTP/1.1
> Host: httpbin.localhost
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Access-Control-Allow-Credentials: true
< Access-Control-Allow-Origin: *
< Content-Length: 606
< Content-Type: application/json; charset=utf-8
< Date: Mon, 23 Mar 2026 12:34:52 GMT
< 
{
  "args": {},
  "headers": {
    "Accept": [
      "*/*"
    ],
    "Accept-Encoding": [
      "gzip"
    ],
    "Host": [
      "httpbin.localhost"
    ],
    "User-Agent": [
      "curl/8.6.0"
    ],
    "X-Forwarded-For": [
      "10.42.0.1"
    ],
    "X-Forwarded-Host": [
      "httpbin.localhost"
    ],
    "X-Forwarded-Port": [
      "80"
    ],
    "X-Forwarded-Proto": [
      "http"
    ],
    "X-Forwarded-Server": [
      "traefik-865bd56545-8wlrl"
    ],
    "X-Real-Ip": [
      "10.42.0.1"
    ]
  },
  "method": "GET",
  "origin": "10.42.0.1",
  "url": "http://httpbin.localhost/get"
}
* Connection #0 to host httpbin.localhost left intact
```

2. Burst test  by sending 15 rapid requests and watch for `429`:

```bash
for i in $(seq 1 15); do
  echo -n "Request $i: "
  curl -s -o /dev/null -w "%{http_code}\n" http://httpbin.localhost/get
done
```

Output is similar to:

```
Request 1: 200
Request 2: 200
Request 3: 200
Request 4: 200
Request 5: 200
Request 6: 200
Request 7: 200
Request 8: 200
Request 9: 200
Request 10: 200
Request 11: 429
Request 12: 429
Request 13: 429
Request 14: 429
Request 15: 200
```

### Tear down

```bash
k3d cluster delete traefik-demo
```

Remove the `/etc/hosts` entries:

```bash
sudo sed -i '' '/whoami.localhost/d' /etc/hosts
sudo sed -i '' '/httpbin.localhost/d' /etc/hosts
```
