---
description: Learn about how the various core concepts of Traefik work together in Docker
---

# Docker

In this guide learn to :&#x20;

1. [Deploy Traefik and `whoami` with basic auth](docker.md#deploy-traefik-whoami-with-basic-auth)
2. [Verify that Traefik adds authentication for the `whoami` service](docker.md#verify-that-the-whoami-service-works-with-basic-auth)
3. [Add `httpbin` with rate limiting to the running stack](docker.md#deploy-httpbin-with-rate-limiting)
4. [Verify that Traefik sets rate limiting for `httpbin`  service](docker.md#verify-that-rate-limiting-works-on-httpbin)

### Before you begin

* [Docker](https://docs.docker.com/get-docker/) installed
* [Docker Compose](https://docs.docker.com/compose/install/) installed
* `curl` available in your terminal

### Deploy Traefik whoami with Basic Auth

1. Generate a Basic Auth password

Run this command to generate a bcrypt hash for user `admin` and set the  password as `admin123`  when prompted:

```bash
echo $(htpasswd -nB admin) | sed -e s/\\$/\\$\\$/g
```

> **Note:** If `htpasswd` is not installed:
>
> * Linux: `sudo apt-get install apache2-utils`
> * macOS: `brew install httpd`

Copy the output and save it. You need this for the `docker-compose.yml` file.

2. Create the Docker Compose file

Create a new directory and save the following as `docker-compose.yml`. Ensure that you replace `<YOUR_HASH>` with the output from generate Basic Auth password.

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.http.middlewares.secure-gate.basicauth.users=admin:<YOUR_HASH>"

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"
      - "traefik.http.routers.whoami.middlewares=secure-gate"
```

3. Start the stack

```bash
docker compose up -d
```

4. Verify both containers are running:

```bash
docker compose ps
```

Output is similar to:

```
NAME                IMAGE            COMMAND                  SERVICE   CREATED         STATUS         PORTS
traefik-traefik-1   traefik:v3.0     "/entrypoint.sh --ap…"   traefik   8 seconds ago   Up 8 seconds   0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp
traefik-whoami-1    traefik/whoami   "/whoami"                whoami    8 seconds ago   Up 8 seconds   80/tcp
```

### Verify that the whoami service works with basic auth

You can confirm that the whoami service works with basic auth using curl or the dashboard

#### Confirm using curl

1. Access the `whoami` service with no credentials:

```bash
curl -v http://localhost
```

Output is similar to:

```
* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to localhost (::1) port 80
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: text/plain
< Www-Authenticate: Basic realm="traefik"
< Date: Mon, 23 Mar 2026 09:44:27 GMT
< Content-Length: 17
< 
401 Unauthorized
* Connection #0 to host localhost left intact
```

2. Access the `whoami` service with wrong password:

```bash
curl -v -u admin:wrongpassword http://localhost
```

Output is simialr to:

```
* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to localhost (::1) port 80
* Server auth using Basic with user 'admin'
> GET / HTTP/1.1
> Host: localhost
> Authorization: Basic YWRtaW46d3JvbmdwYXNzd29yZA==
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: text/plain
* Authentication problem. Ignoring this.
< Www-Authenticate: Basic realm="traefik"
< Date: Mon, 23 Mar 2026 09:45:00 GMT
< Content-Length: 17
< 
401 Unauthorized
* Connection #0 to host localhost left intact
```

3. Access the `whoami` service with correct credentials. If you used a different password, then ensure that you replace `admin123` with the credentials that you set when creating the basic auth password.

```bash
curl -v -u admin:admin123 http://localhost
```

Output is similar to:

```
* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
* Connected to localhost (::1) port 80
* Server auth using Basic with user 'admin'
> GET / HTTP/1.1
> Host: localhost
> Authorization: Basic YWRtaW46YWRtaW4xMjM=
> User-Agent: curl/8.6.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Content-Length: 389
< Content-Type: text/plain; charset=utf-8
< Date: Mon, 23 Mar 2026 09:46:05 GMT
< 
Hostname: 0a90889dacea
IP: 127.0.0.1
IP: ::1
IP: 172.19.0.2
RemoteAddr: 172.19.0.3:47198
GET / HTTP/1.1
Host: localhost
User-Agent: curl/8.6.0
Accept: */*
Accept-Encoding: gzip
Authorization: Basic YWRtaW46YWRtaW4xMjM=
X-Forwarded-For: 172.19.0.1
X-Forwarded-Host: localhost
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 5d30cc46a1f6
X-Real-Ip: 172.19.0.1

* Connection #0 to host localhost left intact
```

#### **Confirm in the Traefik dashboard**

Open your browser and go to:

```
http://localhost:8080/dashboard/
```

Go to **HTTP > HTTP Middlewares** and click **secure-gate@docker** confirm that it is attached to the `whoami` router.

<figure><img src="../.gitbook/assets/docker-dashboard.png" alt=""><figcaption></figcaption></figure>

This confirms that `whoami` is running and protected by basic auth.

### Deploy  httpbin with Rate Limiting

Now add `httpbin` to the running stack.

1\. Update docker-compose.yml

Add the `limit-gate` middleware to the Traefik _labels_ and append the `httpbin` service. Ensure that you still have the same `<YOUR_HASH>` with the output from generate Basic Auth password.

Your updated `docker-compose.yml` should look like this:

```yaml

services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      # Basic auth — for whoami (unchanged)
      - "traefik.http.middlewares.secure-gate.basicauth.users=admin:<YOUR_HASH>"
      # Rate limit — for httpbin (new)
      - "traefik.http.middlewares.limit-gate.ratelimit.average=5"
      - "traefik.http.middlewares.limit-gate.ratelimit.burst=10"

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"
      - "traefik.http.routers.whoami.middlewares=secure-gate"

  httpbin:
    image: mccutchen/go-httpbin
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.httpbin.rule=Host(`httpbin.localhost`)"
      - "traefik.http.routers.httpbin.entrypoints=web"
      - "traefik.http.routers.httpbin.middlewares=limit-gate"
```

2. Apply the changes

Docker Compose only creates the new `httpbin` container and update Traefik labels and `whoami` is untouched:

```bash
docker compose up -d
```

Verify all three containers are now running:

```bash
docker compose ps
```

Expected output:

```
NAME                 SERVICE    STATUS    PORTS
your-dir-traefik-1   traefik    running   0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp
your-dir-whoami-1    whoami     running
your-dir-httpbin-1   httpbin    running
```

### Verify that rate limiting works on httpbin

You can confirm that the ratelimiting works on `httpbin` using curl or the dashboard.

#### Confirm using curl

Single request — expect `200 OK`:

```bash
curl -v http://httpbin.localhost/get
```

Burst test  by sending 15 rapid requests and watch for `429`:

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
...
Request 11: 429
Request 12: 429
...
Request 15: 429
```

#### Confirm in the Traefik dashboard

Go back to `http://localhost:8080/dashboard/` → **HTTP → Middlewares**.

You should now see both middlewares:

This confirms that httpbin is running and protected by rate limiting.

### Summary

| Service           | URL                                | Protection                        |
| ----------------- | ---------------------------------- | --------------------------------- |
| `whoami`          | `http://localhost`                 | Basic auth (`admin` / `admin123`) |
| `httpbin`         | `http://httpbin.localhost/get`     | Rate limit (5 req/s, burst 10)    |
| Traefik dashboard | `http://localhost:8080/dashboard/` | None (insecure mode)              |
