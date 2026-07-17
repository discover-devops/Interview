# Docker Interview Handbook

# Part 1 — Deep Architectural & Core Concepts (Principal DevOps / Lead Engineer Level)

---

# Question 1

## Explain Docker Architecture in detail. What happens internally when you execute `docker run nginx`?

### Why Interviewers Ask This

This question separates someone who has merely used Docker from someone who truly understands its internals. A Principal DevOps Engineer should be able to explain not only the Docker CLI but also the sequence of events inside the Docker Engine, the Linux kernel, and the container runtime.

---

## High-Level Architecture

```
                    docker run nginx
                           │
                           ▼
                     Docker CLI
                           │
                     REST API Call
                           │
                           ▼
                    Docker Daemon
                      (dockerd)
                           │
         ┌─────────────────┼────────────────┐
         │                 │                │
         ▼                 ▼                ▼
   Image Manager      Network Manager   Volume Manager
         │
         ▼
      containerd
         │
         ▼
       containerd-shim
         │
         ▼
          runc
         │
         ▼
 Linux Kernel (Namespaces + Cgroups + OverlayFS)
```

---

## Step 1 — Docker CLI

When you execute:

```bash
docker run nginx
```

The Docker CLI does **not** create the container itself.

Instead, it sends an HTTP REST API request to the Docker daemon.

You can actually verify this:

```bash
docker version
```

The output contains:

```
Client:
...

Server:
 Docker Engine
```

The CLI is simply a client.

---

## Step 2 — Docker Daemon (`dockerd`)

The daemon receives the request.

Its responsibilities include:

* Image management
* Container lifecycle
* Network creation
* Storage management
* Logging
* Security
* Volume management

The daemon first checks whether the image exists locally.

```bash
docker images
```

If not found:

```
docker pull nginx
```

is automatically performed.

---

## Step 3 — Image Download

The daemon contacts Docker Hub (or another registry).

```
Registry
    │
 Layer 1
 Layer 2
 Layer 3
 Layer 4
```

Docker downloads only missing layers.

Each layer is verified using SHA256.

Example:

```
sha256:7a...
sha256:91...
sha256:f8...
```

If a layer already exists locally, it isn't downloaded again.

This is why pulling images is fast after the first time.

---

## Step 4 — Image Extraction

Docker stores image layers under

```
/var/lib/docker/
```

For Overlay2:

```
/var/lib/docker/overlay2/
```

Each image consists of multiple read-only layers.

Example:

```
Ubuntu Base
      │
Python
      │
Application
      │
Configuration
```

---

## Step 5 — Creating Writable Layer

Images are immutable.

Docker therefore creates one additional writable layer.

```
Container

------------------
Writable Layer
------------------
Layer 5
Layer 4
Layer 3
Layer 2
Layer 1
```

Every modification happens only in this writable layer.

---

## Step 6 — containerd

Docker delegates runtime management to **containerd**.

Responsibilities include:

* Pulling images
* Snapshot management
* Runtime lifecycle
* OCI image management

Docker no longer launches containers directly.

```
dockerd
   │
containerd
   │
containerd-shim
```

---

## Step 7 — containerd-shim

Every container gets its own shim process.

Example:

```bash
ps -ef | grep containerd-shim
```

Output:

```
containerd-shim
containerd-shim
containerd-shim
```

Benefits:

* Container survives Docker daemon restart.
* Parent process management.
* STDIN/STDOUT handling.
* Exit status reporting.

---

## Step 8 — runc

`runc` is the low-level OCI runtime.

Responsibilities:

* Create namespaces
* Apply cgroups
* Mount OverlayFS
* Launch PID 1

After the container starts:

```
runc exits
```

The container continues running.

---

## Step 9 — Linux Kernel

The Linux kernel finally creates the isolated execution environment using:

* PID Namespace
* Mount Namespace
* Network Namespace
* IPC Namespace
* UTS Namespace
* User Namespace
* Cgroups
* Seccomp
* Capabilities

At this point the process is simply another Linux process with isolation.

---

## Verify Running Processes

```bash
ps -ef | grep nginx
```

You'll see an Nginx process on the host.

Containers are **not virtual machines**. They are isolated host processes.

---

## Production Interview Follow-up

**Q:** Does Docker emulate a Linux kernel?

**Answer:**

No.

Containers always share the host kernel.

Only user space is isolated.

---

# Question 2

## Explain Linux Namespaces in Docker. How do they isolate containers?

---

Namespaces isolate different kernel resources.

Without namespaces:

```
Process A
Process B
Process C
```

All processes see each other.

With namespaces:

```
Container A

PID 1
PID 2

-----------------

Container B

PID 1
PID 2
```

Each container believes it owns the machine.

---

## PID Namespace

Responsible for:

* Process isolation

Inside container:

```bash
ps -ef
```

Output:

```
PID CMD

1 nginx
7 worker
8 worker
```

Host:

```bash
ps -ef
```

```
3251 nginx
3258 worker
3260 worker
```

The container only sees its own processes.

---

## Network Namespace

Each container receives:

* Virtual Ethernet
* Routing table
* iptables rules
* Loopback interface

Check:

```bash
ip addr
```

You'll see

```
eth0
lo
```

completely separate from the host.

---

## Mount Namespace

Each container has its own filesystem.

```
/
├── bin
├── etc
├── app
```

Even though all containers share image layers.

---

## IPC Namespace

Isolates:

* Shared memory
* Semaphores
* Message queues

Critical for databases.

---

## UTS Namespace

Allows separate:

```
hostname
```

Example:

```bash
hostname
```

Inside:

```
nginx-container
```

Host:

```
prod-server-12
```

---

## User Namespace

Maps container users to different host users.

Example:

```
Container Root
↓

Host UID 100000
```

Root inside container is **not** root on host.

---

## Verify Namespace IDs

```bash
lsns
```

Example:

```
NS TYPE

4026531993 pid
4026532440 net
4026532556 mnt
```

---

## Production Benefit

If one container crashes:

* Processes remain isolated.
* Network unaffected.
* Mounts unaffected.
* IPC isolated.

---

# Question 3

## Explain Cgroups v2. How does Docker enforce CPU and Memory limits?

---

Namespaces isolate.

**Cgroups control resource consumption.**

Without cgroups:

One container can consume:

```
100% CPU
100 GB RAM
```

and starve the host.

---

## Memory Limit

Example:

```bash
docker run \
--memory=512m \
nginx
```

Docker creates cgroup entries.

Check:

```bash
cat /sys/fs/cgroup/memory.max
```

Output:

```
536870912
```

512 MB.

---

## CPU Limit

```bash
docker run \
--cpus=2
```

Container cannot exceed two CPU cores.

---

## CPU Shares

```bash
docker run \
--cpu-shares=512
```

Default:

```
1024
```

Higher shares receive more CPU during contention.

---

## PIDs Limit

```bash
docker run \
--pids-limit=200
```

Prevents fork bombs.

---

## Block IO

```bash
docker run \
--device-read-bps=/dev/sda:10mb
```

Restricts disk throughput.

---

## Verify Limits

```bash
docker inspect containerID
```

Look under:

```
HostConfig
```

---

## Monitor

```bash
docker stats
```

Example:

```
CPU %
MEM USAGE
NET IO
BLOCK IO
```

---

## Production Recommendation

Always set:

* CPU
* Memory
* PID limits

Never run unlimited containers in production.

---

# Question 4

## Explain Overlay2 Storage Driver and Copy-on-Write in Detail

---

Docker images consist of immutable layers.

Example:

```
Ubuntu

↓

Python

↓

Requirements

↓

Application
```

Each layer is read-only.

---

## Overlay2 Structure

```
Upper Layer

↓

Merged

↓

Lower Layers
```

Directories:

```
lowerdir

upperdir

merged

workdir
```

---

## Reading a File

```
Container

↓

Merged View

↓

Lower Layer
```

No copying required.

---

## Writing a File

Suppose:

```
/etc/app.conf
```

exists in lower layer.

Container modifies it.

Overlay2 performs:

```
Copy

↓

Upper Layer

↓

Modify
```

Original remains untouched.

---

## Copy-on-Write Overhead

Large files incur additional cost because the entire file is copied before modification.

Example:

```
2 GB SQLite file
```

Updating one byte duplicates the full file into the writable layer.

---

## Best Practice

Never store mutable databases inside container layers.

Use:

```bash
docker volume create postgres-data
```

---

## Inspect Driver

```bash
docker info
```

Output:

```
Storage Driver: overlay2
```

---

## Inspect Layer Directory

```bash
ls

/var/lib/docker/overlay2
```

---

## Interview Tip

Overlay2 is optimized for:

* Read-heavy workloads
* Immutable application binaries

Not for frequently changing large files.

---

# Question 5

## Explain Multi-Stage Builds vs Traditional Builder Pattern. Why do Multi-Stage Builds Improve Security?

---

Traditional Dockerfile:

```dockerfile
FROM golang:1.24

WORKDIR /app

COPY . .

RUN go build

CMD ["./server"]
```

Problem:

Compiler remains.

```
Go Compiler

Git

Curl

APT

Source Code

Binary
```

Huge attack surface.

---

## Multi-Stage Build

```dockerfile
FROM golang:1.24 AS builder

WORKDIR /app

COPY . .

RUN go build -o server
```

Second stage:

```dockerfile
FROM alpine:3.22

WORKDIR /app

COPY --from=builder /app/server .

CMD ["./server"]
```

Final image contains:

```
server
```

Only.

---

## Benefits

* Smaller image
* Faster pull
* Reduced CVEs
* No compiler
* No source code
* Better cache usage
* Faster deployment

---

## Image Comparison

Traditional:

```
1.2 GB
```

Multi-stage:

```
18 MB
```

---

## Interview Question

**Why is this more secure?**

Because:

* Build tools removed
* Package managers removed
* Shell often absent
* Source code removed
* Smaller attack surface
* Lower vulnerability count

---

# Question 6

## Explain ENTRYPOINT vs CMD (Exec Form vs Shell Form)

---

Consider:

```dockerfile
CMD python app.py
```

Shell form becomes:

```
/bin/sh -c "python app.py"
```

Process tree:

```
sh

↓

python
```

PID 1 becomes:

```
sh
```

Signals may not reach Python properly.

---

## Exec Form

```dockerfile
CMD ["python","app.py"]
```

Process:

```
python
```

PID 1.

Signals delivered correctly.

---

## ENTRYPOINT

Always executes.

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

Running:

```bash
docker run image test.py
```

Actually executes:

```
python test.py
```

---

## Difference

CMD:

Default arguments.

ENTRYPOINT:

Main executable.

---

## Best Practice

```dockerfile
ENTRYPOINT ["java"]

CMD ["-jar","application.jar"]
```

Allows runtime overrides while preserving the executable.

---

## Production Recommendation

Always prefer:

* Exec form
* ENTRYPOINT for the executable
* CMD for default parameters

Avoid shell form unless shell expansion is explicitly required.

---

# Question 7

## Explain Docker Port Mapping Internals. How Does `-p 8080:80` Work?

---

Command:

```bash
docker run -p 8080:80 nginx
```

Docker does **not** modify the application.

Instead, it configures Linux networking to forward traffic from the host to the container.

---

## Network Flow

```
Client
   │
   ▼
Host IP:8080
   │
iptables NAT
   │
Docker Bridge (docker0)
   │
veth Pair
   │
Container eth0:80
   │
Nginx
```

---

## Components Involved

1. **Docker bridge (`docker0`)** – Virtual Layer 2 bridge that connects containers on the default bridge network.
2. **veth pair** – A virtual Ethernet cable with one end in the host namespace and the other inside the container's network namespace.
3. **iptables NAT rules** – Perform destination NAT (DNAT) from the host port to the container's private IP and port.

---

## Verify Port Mapping

List running containers and published ports:

```bash
docker ps
```

Inspect detailed mappings:

```bash
docker port <container_id>
```

Example output:

```
80/tcp -> 0.0.0.0:8080
```

Inspect the bridge network:

```bash
docker network inspect bridge
```

You'll see the container IP (for example, `172.17.0.2`) and gateway information.

---

## Inspect iptables Rules

On Linux hosts using iptables, Docker creates NAT rules automatically.

```bash
sudo iptables -t nat -L -n -v
```

Typical flow:

```
PREROUTING
    ↓
DOCKER chain
    ↓
DNAT
    ↓
172.17.0.2:80
```

These rules forward packets arriving on the host's port 8080 to the container's port 80.

---

## Verify Listening Ports

On the host:

```bash
ss -tulnp | grep 8080
```

or

```bash
netstat -tulnp | grep 8080
```

---

## Performance Considerations

For high-throughput environments:

* Publish only the ports that are required.
* Prefer user-defined bridge networks for better isolation and built-in DNS.
* Avoid unnecessary host port exposure for internal services.
* For large-scale production, ingress/load balancing is typically handled by Kubernetes Services, cloud load balancers, or reverse proxies rather than direct host port publishing.

---

# Key Takeaways for Principal-Level Interviews

A strong senior candidate should be able to articulate that:

* A Docker container is **not a virtual machine**; it is an isolated Linux process sharing the host kernel.
* **Namespaces** provide isolation, while **cgroups v2** enforce resource governance.
* **Overlay2** implements a layered filesystem using Copy-on-Write, which is efficient for immutable workloads but can introduce overhead for large mutable files.
* **Multi-stage builds** significantly reduce image size and attack surface by excluding build tools and source code from the final image.
* **ENTRYPOINT** defines the executable, while **CMD** provides default arguments. The **exec form** should be preferred for correct signal handling.
* Docker port publishing relies on Linux networking primitives—bridge interfaces, veth pairs, network namespaces, and iptables-based NAT—to expose container services.

---

This completes **Part 1 – Section 1 (7 Principal-level architectural questions)**. The next part will move from architecture into **production engineering**, focusing on designing secure, enterprise-ready Docker deployments and the kinds of scenario-based questions commonly asked in Staff, Lead, and Principal DevOps interviews.
