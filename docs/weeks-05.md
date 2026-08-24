# Week 5 - Building the Homelab Infrastructure

## Objective

Move beyond isolated Docker exercises and begin using the concepts learned in previous weeks to build a functional self-hosted infrastructure environment.

The focus this week was understanding how multiple services can work together through Docker networking, persistent storage, internal DNS and reverse proxying.

Rather than deploying applications individually, I wanted to understand how they form part of a larger system.

---

## Moving From Docker Practice to Infrastructure

During Weeks 3/4 I focused primarily on understanding Docker itself:

* Images
* Containers
* Volumes
* Bind mounts
* Networks
* Docker Compose
* Restart policies

This week I began applying those concepts to real services.

The environment grew to include:

```text
Docker
├── Homepage
├── Docker Socket Proxy
├── Portainer
├── Uptime Kuma
├── AdGuard Home
└── Nginx Proxy Manager
```

Each service introduced a different infrastructure problem to solve.

Instead of asking only:

> How do I deploy this container?

I started thinking about:

* What responsibility does this service own?
* What data needs to persist?
* Which other services does it need to communicate with?
* Does it need to be accessible from the LAN?
* Should it expose a host port?
* What happens if the container is recreated?

---

## Persistent Storage Decisions

One of the biggest lessons this week was that not every container should use storage in the same way.

A useful question became:

> **If I delete this container, what data would I be unhappy to lose?**

This helped me decide whether a service required a bind mount, Docker volume or no persistent storage.

### Bind Mounts

Bind mounts were useful when configuration files were intended to be directly viewed or edited from the Ubuntu host.

For example, Homepage configuration is administrator-managed YAML, so keeping the configuration visible on the host made sense.

```text
Ubuntu Host
     │
     │ configuration files
     ▼
Bind Mount
     │
     ▼
Homepage Container
```

### Docker Volumes

Docker-managed volumes were more suitable for application-owned state.

For example, Portainer stores internal application data which I do not need to manually edit from the host filesystem.

This helped reinforce that storage decisions should be based on **who owns the data and how it needs to be managed**, rather than using the same volume configuration for every application.

---

## Homepage

Homepage was deployed as a central dashboard for accessing homelab services.

This was useful not only as an application but also as a way to practise:

* Bind-mounted configuration
* YAML
* Container networking
* Service discovery
* Docker integration

Homepage also introduced an important security consideration.

To display Docker container information, Homepage needs access to information from the Docker API.

Directly exposing:

```text
/var/run/docker.sock
```

to another container provides extremely powerful access to Docker.

Rather than giving Homepage unrestricted access to the Docker socket, I introduced a Docker Socket Proxy.

---

## Docker Socket Proxy

The architecture became:

```text
Homepage
    │
    ▼
Docker Socket Proxy
    │
    ▼
Docker API
```

This helped me understand that the Docker socket is effectively an administrative interface to the Docker daemon.

Giving an application direct access to it can allow that application to perform highly privileged Docker operations.

Using a proxy introduces an additional control layer between the application and Docker.

The main lesson was that functionality should not automatically be given unrestricted privileges simply because it is convenient.

---

## Portainer

Portainer was deployed to provide a graphical interface for viewing and managing the Docker environment.

Using Portainer alongside the Docker CLI helped reinforce what Docker management tools are actually doing.

I learned that Portainer is effectively another client communicating with the Docker API.

```text
Administrator
     │
     ▼
Portainer
     │
     ▼
Docker API
     │
     ▼
Docker Engine
```

This helped connect graphical administration tools back to the Docker architecture I had already learned.

---

## Uptime Kuma

Uptime Kuma was introduced to monitor whether services were reachable.

This helped establish an important difference between:

* **Availability monitoring**
* **Performance monitoring**

Uptime Kuma can answer questions such as:

> Is this service reachable?

Later monitoring tools such as Prometheus and Grafana would answer deeper questions such as:

> How much CPU or memory is the system using?

This helped me understand that different monitoring tools solve different types of problems.

---

## AdGuard Home

AdGuard Home was deployed as the internal DNS server for the homelab.

Before this, services were accessed using addresses such as:

```text
192.168.0.67:3000
192.168.0.67:3001
192.168.0.67:9443
```

This works technically, but it becomes difficult to manage as the number of services increases.

Instead, I configured internal DNS names such as:

```text
homepage.home.lab
portainer.home.lab
status.home.lab
```

AdGuard resolves these hostnames to the Ubuntu Server:

```text
homepage.home.lab
        │
        ▼
    AdGuard DNS
        │
        ▼
   192.168.0.67
```

DNS resolution was verified using:

```bash
nslookup homepage.home.lab
```

which correctly returned the server's LAN address.

---

## Understanding DNS

One of the most important networking concepts that clicked this week was that DNS does **not** decide which application receives a request.

DNS only maps a hostname to an IP address.

For example:

```text
homepage.home.lab
        │
        ▼
   192.168.0.67
```

But several applications can exist on the same IP address.

This introduced the need for a reverse proxy.

---

## Nginx Proxy Manager

Nginx Proxy Manager was deployed to route requests to the correct service.

The architecture became:

```text
Browser
   │
   ▼
homepage.home.lab
   │
   ▼
AdGuard Home
DNS Resolution
   │
   ▼
192.168.0.67
   │
   ▼
Nginx Proxy Manager
   │
   ▼
Homepage :3000
```

Example proxy mappings included:

```text
homepage.home.lab
        ↓
192.168.0.67:3000

portainer.home.lab
        ↓
192.168.0.67:9443

status.home.lab
        ↓
192.168.0.67:3001
```

This was one of the most important architecture concepts I learned.

I now understand the difference as:

**DNS**

```text
Hostname → IP address
```

**Reverse Proxy**

```text
Hostname arriving at that IP → Application
```

---

## Host Headers

I also learned how the reverse proxy determines which application the user requested.

Although several DNS names resolve to the same server IP, the browser includes the requested hostname in the HTTP request.

Nginx can inspect this hostname and forward the request to the correct backend service.

```text
Request:
Host: homepage.home.lab
          │
          ▼
Nginx Proxy Manager
          │
          ▼
Homepage
```

This explains how several websites can operate behind the same IP address and standard HTTP/HTTPS ports.

---

## Ports Belong to IP Addresses

While configuring DNS services I developed a better understanding of port binding.

An important lesson was:

> **Ports belong to IP addresses, not simply to a machine.**

For example:

```text
127.0.0.53:53
```

and:

```text
192.168.0.67:53
```

are different network sockets even though both exist on the same Ubuntu server.

This became important while understanding DNS listeners and avoiding conflicts between services attempting to use port 53.

---

## Current Request Flow

By the end of this stage, the homelab request flow looked like:

```text
Windows Laptop
      │
      ▼
homepage.home.lab
      │
      ▼
AdGuard Home
Internal DNS
      │
      ▼
192.168.0.67
      │
      ▼
Nginx Proxy Manager
Reverse Proxy
      │
      ▼
Homepage Container
```

This was the first point where the homelab started to feel like an interconnected infrastructure environment rather than a collection of individual containers.

---

## Troubleshooting Approach

As the number of services increased, troubleshooting also became more architecture-focused.

Instead of immediately changing configurations, I started asking:

* Did DNS resolve the correct IP?
* Did the request reach the correct host?
* Which port is the application listening on?
* Is the container running?
* Is the reverse proxy forwarding to the correct destination?
* Is persistent configuration located where I expect?
* Are containers connected to the correct Docker network?

The workflow continued to be:

> **Inspect → Understand → Change → Verify**

---

## What I Learned

This week helped connect several previously separate concepts:

```text
Linux
  +
Docker
  +
Networking
  +
DNS
  +
HTTP
  +
Reverse Proxying
      │
      ▼
Functional Homelab Infrastructure
```

The biggest change in my understanding was moving from thinking about services individually to thinking about **responsibilities and communication between components**.

AdGuard owns internal DNS resolution.

Nginx Proxy Manager owns application routing.

Docker owns container lifecycle and networking.

Each application owns its own state and functionality.

Understanding those boundaries makes the overall system easier to design and troubleshoot.

---

## Reflection

This week was an important transition from learning individual technologies to building infrastructure with them.

The biggest lesson was that deploying an application is only a small part of operating a service.

A working environment also needs:

* Networking
* Name resolution
* Routing
* Persistent storage
* Access control
* Monitoring
* Troubleshooting

Building internal DNS and reverse proxying helped me understand how services can be presented cleanly to users while remaining separated internally.

The next stage of the project will focus on **monitoring and observability**, allowing me to understand not only whether services are running, but how the underlying host and containers are performing.
