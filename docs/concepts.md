# LVM

Logical Volume Manager.

Acts as a storage management layer between the physical disk and the filesystem.
Pool of storage, carves out logical volumes from that pool.

Advantages

- Flexible storage
- Easier resizing
- Widely used in enterprise Linux

## Local LVM
- Where actual virtual machine files are stored such as installtion files

---


# SSH

Secure Shell.

Encrypted protocol allows to remotely access and manage another computer over a network using an encrypted connection.

Used to remotely execute commands over a network.

---

# vmbr0

Linux bridge created by Proxmox.
Proxmosx creates a virtual switch inside the optiplex.

Acts like a virtual network switch that connects virtual machines to the physical network.

---

# Docker

Platform used to package and run applications inside isolated containers.

Unlike virtual machines, containers share the host operating system kernel, making them lightweight, portable and quick to deploy.

Docker follows the philosophy that containers should be disposable and easily recreated from images rather than manually repaired.

---

# Docker Image

A read-only blueprint used to create Docker containers.

Images contain:

- Application code
- Dependencies
- Runtime environment
- Configuration defaults

Multiple containers can be created from a single image.

---

# Docker Container

A running instance of a Docker image.

Containers provide application isolation while sharing the host Linux kernel.

Containers are designed to be temporary and replaceable.

---

# Docker Volume

Persistent storage managed by Docker.

Volumes separate application data from containers so that data survives container deletion and recreation.

Commonly used for:

- Databases
- Application configuration
- User uploads

---

# Bind Mount

Creates a direct connection between a directory on the host machine and a directory inside a container.

Unlike Docker volumes, changes made on the host appear immediately inside the running container.

Primarily used during development.

---

# Docker Network

Virtual network created by Docker allowing containers to communicate securely.

Containers on the same Docker network communicate using container names rather than IP addresses through Docker's built-in DNS.

---

# Docker Compose

Tool used to define and manage multi-container applications.

Application infrastructure is described declaratively inside a `compose.yaml` file, allowing entire applications to be recreated consistently using a single command.

---

# Dockerfile

A text file containing instructions used to build a Docker image.

Common instructions include:

- `FROM`
- `COPY`
- `RUN`
- `CMD`

Dockerfiles make deployments reproducible by packaging applications together with their dependencies.

---

# Environment Variables

Configuration values passed into a container when it starts.

Applications read these variables during startup to determine settings such as usernames, passwords, ports and time zones.

Changing environment variables typically requires recreating the container.

---

# Restart Policy

Docker setting that determines what happens if a container stops or the host system reboots.

The `unless-stopped` policy is commonly used for servers because containers automatically restart after system boot while respecting containers intentionally stopped by an administrator.
