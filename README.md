---
icon: hand-wave
---

# Welcome

Modern applications rarely run as a single service. They are made up of many smaller services such as APIs, frontends, backends, and microservices. Each running in its own container or Pod, and needs to be reachable from the outside world. Traefik helps the request from your browser find its way to the right service.

Traefik is a cloud-native reverse proxy and load balancer designed for dynamic environments like Docker and Kubernetes. Unlike traditional reverse proxies that require manual configuration every time a service changes, Traefik watches your infrastructure and updates its routing configuration automatically.

When you deploy a new container or a new Kubernetes service, Traefik detects it and starts routing traffic to it without a restart, a config reload, or manual intervention.

### Core Concepts

Traefik's architecture is built around four concepts that work together to move a request from the moment it arrives at your infrastructure to the moment it reaches your application.\
\* [Entrypoints](./#entrypoints)\
\* [Routers](./#routers)\
\* [Services](./#services)\
\* [Providers](./#providers)

#### Entrypoints

An entrypoint is the door through which traffic enters Traefik. It defines a port and a protocol such as TCP or UDP that Traefik listens on.

In most setups you will have at least two entrypoints: one for HTTP traffic on port 80, and one for HTTPS traffic on port 443. Entrypoints as the receiving end of your infrastructure they accept incoming packets but do not yet know what to do with them.

#### Routers

A router inspects each incoming request and decides which service should handle it. It does this by evaluating match rules or conditions based on the request's host, path, headers, or method.

For example, a router might say: _if the request host is `whoami.localhost`, send it to the whoami service_. Routers can also attach middlewares, which process the request before it reaches the service.

**Middlewares** sit between the router and the service. They intercept the request and can modify it, reject it, or enrich it before it is forwarded.

Common middleware use cases include:

* **Authentication:** basic auth, forward auth, digest auth
* **Rate limiting:** sets the limit for maximum number of requests per second
* **Redirects:** HTTP to HTTPS, trailing slash handling
* **Headers:** add, remove, or rewrite request and response headers
* **Circuit breaking:** stop forwarding requests to an unhealthy service

To view the complete list of all the middllewares that are available, see [HTTP Middleware Overview](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/).

#### Services

A service in Traefik represents the actual application that handles the request. After the router has matched an incoming request and passed it through any middlewares, it hands the request off to a service, which knows the address of the real backend.

In a Kubernetes environment, a Traefik service maps to a Kubernetes Service object to the Pod IP addresses and can load balance across multiple replicas.

#### Providers

A provider is how Traefik discovers routing configuration. Instead of reading a static config file, Traefik queries your infrastructure directly by asking Docker for running containers, or Kubernetes for Services and IngressRoutes and then builds its routing table from what it finds.

When you add a new label to a Docker container or apply a new `IngressRoute` to your cluster, the provider picks up the change and Traefik updates its configuration automatically, with no downtime.

In the examples in this guide, Kubernetes is the provider. Traefik watches for `IngressRoute`, `Middleware`, and `Service` objects in your cluster and configures itself accordingly.

### How They Work Together

These four concepts form a pipeline that every request passes through. When a request originates from the _internet_, it first arrives at an **Entrypoint**, which listens for incoming traffic on a specific port. This request is then passed to a **Router** that evaluates it against predefined rules such as the Host, Path, or headers to determine its destination. Before reaching the final application, the request may pass through one or more **Middlewares** to undergo processing tasks like authentication, rate limiting, or redirects. After it is processed, the request is handed off to a **Service**, which effectively forwards it to your _Application_ for final execution.

**Providers** keep this pipeline up to date by continuously watching your infrastructure for changes and pushing new configuration to Traefik as services are added, removed, or updated.
