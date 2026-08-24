# Week 6 - Monitoring, Troubleshooting and Security Hardening

## Objective

Improve the homelab by introducing monitoring and defensive security controls while developing a more structured approach to diagnosing real infrastructure problems.

The main areas covered were:

* Host monitoring
* Container monitoring
* Metrics collection
* Time-series data
* Grafana dashboards
* Linux storage troubleshooting
* Firewall configuration
* SSH hardening
* Brute-force protection
* Initial TLS and certificate authority work

---

# Monitoring & Observability

## Why Monitoring?

Until this point, most checks were based on current state.

For example:

```bash
docker ps
```

can show whether a container is currently running.

However, this does not answer questions such as:

* Was CPU usage high ten minutes ago?
* Has memory usage been increasing?
* Is disk utilisation gradually approaching capacity?
* Which container consumes the most resources?
* When did a problem begin?

This introduced the concept of **observability**.

Rather than only viewing the current state of a system, metrics can be collected over time and stored for later analysis.

---

## Monitoring Architecture

The monitoring environment was designed as:

```text
                  Ubuntu Host
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
      Node Exporter          cAdvisor
      Host Metrics       Container Metrics
            │                   │
            └─────────┬─────────┘
                      ▼
                 Prometheus
                      │
                      ▼
                   Grafana
```

Each component has a separate responsibility.

### Node Exporter

Exposes metrics about the Ubuntu host.

### cAdvisor

Exposes metrics about Docker containers.

### Prometheus

Collects and stores those metrics as time-series data.

### Grafana

Queries Prometheus and visualises the results.

---

## Node Exporter

Node Exporter was deployed to expose host-level metrics.

These include information such as:

* CPU
* Memory
* Filesystems
* Load average
* Network interfaces

To inspect the host filesystem, the Ubuntu root filesystem is mounted into the container read-only.

```text
Ubuntu /
   │
   ▼
Bind Mount
   │
   ▼
Container /host
```

The mount is read-only because Node Exporter only needs to inspect the host.

It does not need permission to modify it.

Node Exporter itself does not store historical metrics.

Its responsibility is simply to expose the current values.

---

## cAdvisor

cAdvisor was deployed to collect container-level metrics.

It requires read-only visibility into several areas of the host, including Docker's own storage and Linux system information.

For example:

```text
/var/lib/docker
       │
       ▼
Read-only bind mount
       │
       ▼
cAdvisor
```

This reinforced the idea that monitoring tools often require visibility into a system without requiring permission to modify that system.

cAdvisor exposes information such as:

* Container CPU usage
* Container memory usage
* Network traffic
* Container resource statistics

---

## Monitoring Docker Network

A dedicated Docker network was created:

```bash
docker network create monitoring
```

The monitoring services were attached to this network.

```text
monitoring
├── node-exporter
├── cadvisor
├── prometheus
└── grafana
```

Instead of configuring Prometheus using temporary container IP addresses, Docker DNS allows services to communicate using their container names.

```text
node-exporter:9100
cadvisor:8080
prometheus:9090
```

This is more reliable because container IP addresses can change when containers are recreated.

---

## Prometheus

Prometheus was introduced as the central metrics collector and time-series database.

It was configured to scrape:

```text
node-exporter:9100
cadvisor:8080
```

every 15 seconds.

This introduced the **pull monitoring model**.

Instead of exporters continuously sending metrics somewhere, Prometheus periodically asks each exporter for its current metrics.

```text
Prometheus
     │
     ├──── GET metrics ────> Node Exporter
     │
     └──── GET metrics ────> cAdvisor
```

Prometheus then stores these values over time.

Because Prometheus owns historical data, it uses a persistent Docker volume.

---

## Bind Mount Troubleshooting

While deploying Prometheus I encountered a bind-mount error.

I accidentally had:

```text
prometheus.yaml   ← configuration file
prometheus.yml    ← directory
```

Docker Compose attempted to mount the directory as though it were a file.

This produced an error explaining that the mount destination was not a directory.

The issue reinforced an important Docker storage rule:

```text
Host file      → Container file       ✅
Host directory → Container directory  ✅
Directory      → File                 ❌
```

After identifying the mismatch, the Compose configuration was corrected to reference the real configuration file.

This was another example of why inspecting the filesystem before changing configuration is important.

---

## Docker DNS Investigation

During testing, a networking issue initially appeared to exist because a utility inside the Prometheus container failed to resolve:

```text
node-exporter
cadvisor
```

Instead of immediately changing the Docker network, I verified each assumption.

The investigation included:

* Inspecting the Docker network
* Confirming all containers were attached
* Checking container aliases
* Checking Docker's embedded DNS configuration
* Testing direct IP connectivity
* Testing DNS using a temporary BusyBox container
* Querying Prometheus's own target API

Docker DNS successfully resolved the services and Prometheus reported both targets as:

```text
UP
```

The conclusion was that Prometheus networking was functioning correctly and the original utility was not a reliable test in that environment.

This reinforced an important troubleshooting lesson:

> Do not trust a single diagnostic tool more than the evidence from the system itself.

---

## Grafana

Grafana was deployed as the visualisation layer.

Grafana communicates with Prometheus internally using:

```text
http://prometheus:9090
```

This reinforced the difference between Docker networking and LAN networking.

```text
Windows Browser
      │
      └── prometheus:9090 ❌

Grafana Container
      │
      └── prometheus:9090 ✅
```

The hostname `prometheus` exists inside the Docker network through Docker's embedded DNS.

It is not a hostname available to devices on the normal home network.

---

## Grafana Dashboard

A dashboard was created to monitor both the Ubuntu host and Docker containers.

Host metrics included:

* CPU utilisation
* Memory utilisation
* Root filesystem usage
* System load

Container metrics included:

* CPU usage
* Memory usage
* Network receive traffic
* Network transmit traffic

This required learning basic PromQL and understanding the types of metrics being queried.

---

## CPU Metrics

Host CPU utilisation was calculated using the rate of change of idle CPU time.

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

This helped me understand that:

```text
node_cpu_seconds_total
```

is a continuously increasing counter.

`rate()` calculates how quickly that counter changes over time.

Instead of directly measuring "CPU percentage", the query calculates idle CPU time and subtracts it from 100%.

---

## Memory Metrics

Host memory utilisation was calculated using:

```promql
(
  1 - (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
) * 100
```

This helped me understand why Linux memory should not be interpreted simply as:

```text
Total RAM - Free RAM
```

Linux intentionally uses otherwise unused memory for caches.

`MemAvailable` provides a better estimate of the memory the system could make available to applications.

---

## System Load

The dashboard also uses:

```promql
node_load1
```

for one-minute system load.

A useful lesson was that **load average is not CPU percentage**.

Load represents work waiting for or currently using system resources rather than simply measuring processor utilisation.

---

# Real Troubleshooting Incident — 96% Root Disk Usage

One of the most valuable parts of the project happened when Grafana showed:

```text
Root Disk Usage: ~96%
```

Rather than immediately deleting Docker data, I investigated the problem systematically.

---

## Step 1 — Verify the Metric

Grafana showed the symptom.

I verified it directly using:

```bash
df -h /
```

which confirmed that the root filesystem was approximately 96% full.

```text
Grafana
   │
   ▼
96% usage
   │
   ▼
df -h /
   │
   ▼
Confirmed
```

---

## Step 2 — Investigate Docker

Because Docker was running several services, Docker storage was an obvious hypothesis.

I checked:

```bash
sudo docker system df
```

Docker images consumed several gigabytes of storage.

However, almost none of that space was reclaimable.

Deleting images would therefore not have solved the underlying problem.

---

## Step 3 — Inspect the Storage Layout

I then inspected the VM's storage using:

```bash
lsblk
sudo vgs
sudo lvs
```

The VM had approximately:

```text
32 GB virtual disk
```

but the root logical volume was only approximately:

```text
14.5 GB
```

Further inspection showed that approximately another:

```text
14.5 GB
```

was still free inside the LVM volume group.

The disk itself was not full.

The actual problem was that only around half of the available LVM capacity had been assigned to the root filesystem.

---

## Step 4 — Extend the Logical Volume

Rather than consuming all available capacity, I deliberately added 10 GB:

```bash
sudo lvextend -L +10G /dev/ubuntu-vg/ubuntu-lv
```

This increased the size of the logical volume.

However, increasing the logical volume does not automatically mean the filesystem can use the additional space.

---

## Step 5 — Resize the Filesystem

The ext4 filesystem was then expanded:

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

This helped clarify the distinction:

```text
lvextend
    │
    ▼
Makes the logical volume larger

resize2fs
    │
    ▼
Makes the ext4 filesystem use that space
```

---

## Result

The root filesystem increased from approximately:

```text
14.5 GB
```

to:

```text
24.5 GB
```

Disk utilisation dropped from around:

```text
96%
```

to approximately:

```text
58%
```

while several gigabytes remained unallocated inside the volume group for future flexibility.

The complete troubleshooting path was:

```text
Grafana detects symptom
        │
        ▼
df confirms filesystem usage
        │
        ▼
Docker storage investigated
        │
        ▼
lsblk / vgs / lvs
        │
        ▼
Unused LVM capacity discovered
        │
        ▼
Logical volume expanded
        │
        ▼
Filesystem resized
        │
        ▼
Grafana verifies recovery
```

This was one of the strongest examples so far of using monitoring to discover and resolve a real infrastructure issue.

---

# Security Hardening

After establishing monitoring, I began improving the defensive security of the Ubuntu server.

Rather than relying on one security control, the aim was to build several layers.

```text
Network
   │
   ▼
UFW Firewall
   │
   ▼
Hardened SSH
   │
   ▼
Public-Key Authentication
   │
   ▼
Fail2Ban
```

---

## UFW Firewall

UFW was configured using a default-deny inbound policy.

Before enabling the firewall, I made sure SSH access was explicitly permitted to avoid locking myself out of the server.

Required services were then allowed, including:

```text
22/tcp   SSH
53/tcp   DNS
53/udp   DNS
80/tcp   HTTP
443/tcp  HTTPS
```

Default behaviour was configured as:

```text
Incoming → Deny
Outgoing → Allow
```

After enabling UFW I verified:

* New SSH connections
* DNS resolution
* Web service access

This reinforced an important security principle:

> Only expose network services that are required.

---

## TCP and UDP

Firewall configuration also helped reinforce transport-layer networking.

### TCP

TCP is connection-oriented and provides reliable ordered delivery.

Services such as:

```text
SSH
HTTP
HTTPS
```

commonly use TCP.

### UDP

UDP is connectionless and has lower overhead.

Normal DNS queries commonly use UDP, although DNS can also use TCP when required.

Understanding the difference made firewall rules more meaningful than simply memorising port numbers.

---

## SSH Public-Key Authentication

Previously, SSH access used the Ubuntu account password.

I migrated administration to Ed25519 public-key authentication.

A key pair was generated on the Windows administration machine:

```text
id_ed25519       ← private key
id_ed25519.pub   ← public key
```

The public key was added to:

```text
~/.ssh/authorized_keys
```

on Ubuntu.

The private key remains on the Windows client.

The authentication flow became:

```text
Windows
Private Key
    │
    ▼
SSH Authentication
    │
    ▼
Ubuntu
Public Key
```

The private key itself is never sent to the server.

---

## SSH Key Passphrase

The private key was protected with a passphrase.

An important distinction was understanding that the passphrase:

> Unlocks the private key locally.

It is **not** the Ubuntu account password and is not sent to the remote server.

---

## SSH Hardening

After confirming that public-key authentication worked, SSH was hardened further.

A separate configuration file was created under:

```text
/etc/ssh/sshd_config.d/
```

with settings equivalent to:

```text
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

Before restarting SSH, the configuration was validated.

This was important because a syntax error in SSH configuration could make remote administration unavailable.

The final posture became:

```text
Public-key authentication     ✅
Password SSH authentication   ❌
Direct root SSH login         ❌
```

---

## Fail2Ban

Fail2Ban was then deployed to provide automated protection against repeated authentication failures.

The SSH jail uses the systemd backend.

This means Fail2Ban obtains authentication evidence from the systemd journal rather than a traditional standalone log file.

Current policy:

```text
5 failed SSH attempts
within 10 minutes
        │
        ▼
Source IP banned
        │
        ▼
1 hour
```

The configuration was validated before restarting the service.

Fail2Ban confirmed that the SSH jail was active.

This introduced the idea of **automated defensive response**.

Instead of an administrator manually detecting repeated login attempts, the system can temporarily block the source automatically.

---

## Defence in Depth

The combination of controls produced:

```text
Incoming Connection
        │
        ▼
UFW
Is the port allowed?
        │
        ▼
OpenSSH
Is authentication valid?
        │
        ▼
Public Key
Does the client possess the private key?
        │
        ▼
Fail2Ban
Has this source repeatedly failed authentication?
```

No individual control is expected to provide complete security.

Instead, several controls reduce risk at different stages.

---

# TLS and Private Certificate Authority

I also began investigating HTTPS for internal `.home.lab` services.

Because names such as:

```text
homepage.home.lab
grafana.home.lab
portainer.home.lab
```

exist only on the private network, a normal public certificate authority was not being used.

Instead, I began building a private homelab Certificate Authority.

---

## Root Certificate Authority

A private root CA was created.

The CA has:

```text
home-lab-ca.key
```

as its private signing key and:

```text
home-lab-ca.crt
```

as its public root certificate.

The most important lesson was understanding that the CA private key is effectively the master identity of the internal PKI.

It must remain private and must **never** be committed to GitHub.

---

## Wildcard Certificate

A wildcard certificate was then generated for:

```text
*.home.lab
```

with Subject Alternative Names covering:

```text
*.home.lab
home.lab
```

The wildcard certificate was signed by the private Home Lab Root CA.

This produced a basic internal trust hierarchy:

```text
Home Lab Root CA
       │
       ▼
Signs
       │
       ▼
*.home.lab Certificate
       │
       ▼
Internal Web Services
```

---

## Windows Trust Store

The public root CA certificate was copied to the Windows administration machine and installed into the Windows Trusted Root Certification Authorities store.

This means Windows can trust certificates signed by the Home Lab Root CA.

The CA **private key was not copied**.

---

## HTTPS Status

The certificate work is currently incomplete.

The next stage is to import the wildcard certificate into Nginx Proxy Manager and apply it to internal proxy hosts.

Current status:

```text
Private Root CA             ✅
Wildcard certificate        ✅
Windows trusts Root CA      ✅
Nginx certificate import    ⏳
HTTPS proxy hosts           ⏳
```

This section will be expanded once HTTPS deployment is complete.

---

# Security Consideration — Docker Published Ports

One important issue identified during firewall work is that Docker-published ports require additional attention.

Docker manages its own firewall rules and published ports can interact with UFW in ways that are not immediately obvious.

Therefore I am not assuming:

> UFW default-deny automatically protects every Docker-published port.

Future hardening will include:

* Reviewing all published Docker ports
* Removing unnecessary LAN exposure
* Moving more services behind Nginx Proxy Manager
* Investigating Docker firewall behaviour
* Applying least privilege to network exposure

Identifying unresolved security issues is an important part of the project rather than treating a configuration as secure simply because a firewall is enabled.

---

# What I Learned

This stage of the project connected monitoring, Linux administration and cybersecurity together.

The biggest lessons were:

* Monitoring should provide evidence rather than just visual dashboards.
* Metrics are useful when they help identify real problems.
* Troubleshooting should test hypotheses rather than rely on random changes.
* Linux storage capacity can exist at several layers.
* A VM disk, LVM volume group, logical volume and filesystem are separate concepts.
* Security should be built in layers.
* SSH keys reduce reliance on reusable passwords.
* Firewalls reduce unnecessary network exposure.
* Fail2Ban can automate responses to repeated suspicious activity.
* Certificate trust depends on who signs a certificate and which certificate authorities a client trusts.
* A system should never be assumed secure without understanding how the underlying components interact.

---

## Reflection

This was probably the most significant stage of the homelab so far.

Monitoring started as a way of learning Prometheus and Grafana, but quickly became useful when it exposed a genuine storage problem.

Instead of simply clearing files, I was able to trace the problem through the Linux storage stack and identify unused LVM capacity.

The security work also changed how I think about administration.

Previously, successful remote access was enough.

Now I think about:

* Who should be allowed to connect?
* Which authentication methods should be permitted?
* What happens after repeated failures?
* Which services genuinely need network exposure?
* How can controls be layered?

The homelab is increasingly becoming less about deploying applications and more about understanding how infrastructure is **operated, monitored, secured and troubleshot**.
