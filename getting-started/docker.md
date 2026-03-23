---
description: Learn about how the various core concepts of Traefik work together in Docker
---

# Docker

In this guide learn to :&#x20;

1. Deploy Traefik and `whoami` with basic auth
2. Add `httpbin` with rate limiting to the running stack

### Before you begin

* [Docker](https://docs.docker.com/get-docker/) installed
* [Docker Compose](https://docs.docker.com/compose/install/) installed
* `curl` available in your terminal

### Deploy Traefik whoami with Basic Auth

1. Generate a Basic Auth password

Run this command to generate a bcrypt hash for user `admin` with password `admin123`:

```bash
echo $(htpasswd -nB admin) | sed -e s/\\$/\\$\\$/g
```

> **Note:** If `htpasswd` is not installed:
>
> * Linux: `sudo apt-get install apache2-utils`
> * macOS: `brew install httpd`

Copy the output and save it. You need this for the Docker compose file .

Copy the output and save it . You need this when you create the compose file.

2. Create the Docker Compose file

Create a new directory and save the following as `docker-compose.yml`. Ensure that you replace `<YOUR_HASH>` with the output from generate Basic Auth password.

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

Expected output:

```
NAME                SERVICE    STATUS    PORTS
your-dir-traefik-1  traefik    running   0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp
your-dir-whoami-1   whoami     running
```

4. Test basic auth on whoami using:

* curl
* dashboard

#### Test basic auth on whoami

**No credentials — expect `401 Unauthorized`:**

```bash
curl -v http://localhost
```

**Wrong password — expect `401 Unauthorized`:**

```bash
curl -v -u admin:wrongpassword http://localhost
```

**Correct credentials — expect `200 OK`:**

```bash
curl -v -u admin:admin123 http://localhost
```

A successful response looks like this:

```
Hostname: 3f4a2b1c
IP: 172.18.0.3
GET / HTTP/1.1
Host: localhost
X-Forwarded-For: 172.18.0.1
...
```

Confirm in the Traefik dashboard

Open your browser and go to:

```
http://localhost:8080/dashboard/
```

Go to **HTTP → Middlewares** and confirm `secure-gate` is listed and attached to the `whoami` router.

This confirms that `whoami` is running and protected by basic auth.

### Deploy  httpbin with Rate Limiting

No need to tear down. You will add `httpbin` to the running stack.

1\. Update docker-compose.yml

Add the `limit-gate` middleware to the Traefik labels and append the `httpbin` service.

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
      - "traefik.http.routers.httpbin.middlewares=limit-gate"Apply the changes
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

#### Test rate limiting on httpbin

3. Test rate limiting on httpbin

**Single request — expect `200 OK`:**

```bash
curl -v http://httpbin.localhost/get
```

**Burst test — fire 15 rapid requests and watch for `429`:**

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

#### Confirm whoami still works

Make sure adding httpbin did not affect whoami:

```bash
curl -v -u admin:admin123 http://localhost
```

Should still return `200 OK` with the whoami response.



4. Confirm in the Traefik dashboard

#### Confirm in the Traefik dashboard

Go back to `http://localhost:8080/dashboard/` → **HTTP → Middlewares**.

You should now see both middlewares:

This confirms that httpbin is running and protected by rate limiting.

***

***

### Summary

| Service           | URL                                | Protection                        |
| ----------------- | ---------------------------------- | --------------------------------- |
| `whoami`          | `http://localhost`                 | Basic auth (`admin` / `admin123`) |
| `httpbin`         | `http://httpbin.localhost/get`     | Rate limit (5 req/s, burst 10)    |
| Traefik dashboard | `http://localhost:8080/dashboard/` | None (insecure mode)              |
