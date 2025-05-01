# Docker Networking

Networking allows containers to communicate with each other and with the host system. Containers run isolated from the host system
and need a way to communicate with each other and with the host system.

By default, Docker provides two network drivers for you, the bridge and the overlay drivers. 

```
docker network ls
```

```
NETWORK ID          NAME                DRIVER
xxxxxxxxxxxx        none                null
xxxxxxxxxxxx        host                host
xxxxxxxxxxxx        bridge              bridge
```


### Bridge Networking

The default network mode in Docker. It creates a private network between the host and containers, allowing
containers to communicate with each other and with the host system.

![image](https://user-images.githubusercontent.com/43399466/217745543-f40e5614-ac34-4b78-85a9-91b24512388d.png)

If you want to secure your containers and isolate them from the default bridge network you can also create your own bridge network.

```
docker network create -d bridge my_bridge
```

Now, if you list the docker networks, you will see a new network.

```
docker network ls

NETWORK ID          NAME                DRIVER
xxxxxxxxxxxx        bridge              bridge
xxxxxxxxxxxx        my_bridge           bridge
xxxxxxxxxxxx        none                null
xxxxxxxxxxxx        host                host
```

This new network can be attached to the containers, when you run these containers.

```
docker run -d --net=my_bridge --name db training/postgres
```

This way, you can run multiple containers on a single host platform where one container is attached to the default network and 
the other is attached to the my_bridge network.

These containers are completely isolated with their private networks and cannot talk to each other.

![image](https://user-images.githubusercontent.com/43399466/217748680-8beefd0a-8181-4752-a098-a905ebed5d2a.png)


However, you can at any point of time, attach the first container to my_bridge network and enable communication

```
docker network connect my_bridge web
```

![image](https://user-images.githubusercontent.com/43399466/217748726-7bb347d0-3736-4f89-bdff-31d240b15150.png)


### Host Networking

This mode allows containers to share the host system's network stack, providing direct access to the host system's network.

To attach a host network to a Docker container, you can use the --network="host" option when running a docker run command. When you use this option, the container has access to the host's network stack, and shares the host's network namespace. This means that the container will use the same IP address and network configuration as the host.

Here's an example of how to run a Docker container with the host network:

```
docker run --network="host" <image_name> <command>
```

Keep in mind that when you use the host network, the container is less isolated from the host system, and has access to all of the host's network resources. This can be a security risk, so use the host network with caution.

Additionally, not all Docker image and command combinations are compatible with the host network, so it's important to check the image documentation or run the image with the --network="bridge" option (the default network mode) first to see if there are any compatibility issues.

### Overlay Networking

This mode enables communication between containers across multiple Docker host machines, allowing containers to be connected to a single network even when they are running on different hosts.

### Macvlan Networking

This mode allows a container to appear on the network as a physical host rather than as a container.

Here’s a comprehensive explanation of **Docker networking modes** — **bridge**, **host**, **overlay**, and **null** — along with:

---

## 🔧 Docker Network Modes — Overview & Comparison

| Feature / Mode  | `bridge`                       | `host`                       | `overlay`                          | `null` (none)                     |
|-----------------|--------------------------------|------------------------------|------------------------------------|-----------------------------------|
| **Default?**    | ✅ Yes (for single-host)       | ❌                            | ❌                                 | ❌                                |
| **Isolation**   | High (NAT between host & cont) | Low (shares host network)    | High (network across multiple hosts) | Full (no network at all)         |
| **Container IP**| Separate IP (internal subnet)  | Same as host’s IP            | Virtual IP managed by Docker       | None                              |
| **Use Case**    | Web apps, DBs, typical apps    | High-perf apps like Prometheus | Swarm clusters, microservices     | Full isolation, testing           |
| **Cross-Host?** | ❌ No                          | ❌ No                         | ✅ Yes (via Docker Swarm)          | ❌ No                              |
| **Customizable?**| ✅ Yes (custom bridge nets)    | ❌ No                         | ✅ Yes (requires Swarm)            | ❌ No                              |

---

## 1️⃣ **Bridge Network (default)**

**How it works:**
- Docker creates a virtual bridge (`docker0`) on the host.
- Each container gets a private IP.
- Traffic is NAT’d through the host.

📦 2 containers can talk to each other **if they are in the same bridge network**.

🖼️ **Diagram:**

```
[ EC2 Host ]
   |
   |-- docker0 (bridge)
   |    |-- Container A (172.17.0.2)
   |    |-- Container B (172.17.0.3)
   |
  [ Outside world ]
```

✅ **Use case:** Default networking for web apps, APIs, databases on the same host.

---

## 2️⃣ **Host Network**

**How it works:**
- Containers share the host’s **network namespace**.
- No private IP — they use the host’s IP directly.
- No port mapping needed.

🖼️ **Diagram:**

```
[ EC2 Host ]
   |-- Container A (host network) -- shares host IP (e.g., 10.0.0.1)
   |-- Container B (host network) -- same
```

❌ You must **avoid port conflicts**.

✅ **Use case:** High-performance apps like monitoring agents (Prometheus, Telegraf).

---

## 3️⃣ **Overlay Network**

**How it works:**
- Creates a **virtual network across multiple Docker hosts**.
- Requires **Docker Swarm** or **Kubernetes**.
- Uses VXLAN tunnels between hosts.

🖼️ **Diagram:**

```
[ EC2 Host A ]           [ EC2 Host B ]
   |                         |
   |-- Cont A (10.0.0.2)     |-- Cont B (10.0.0.3)
   |                         |
   +-------- overlay --------+
         (managed by Swarm)
```

✅ **Use case:** Microservices across nodes, Docker Swarm clusters.

---

## 4️⃣ **None Network (null)**

**How it works:**
- Container has **no networking stack**.
- No external or internal communication.

🖼️ **Diagram:**

```
[ EC2 Host ]
   |
   |-- Container A (no network) ❌
         ↳ no inbound/outbound connections
```

✅ **Use case:** Security isolation, malware sandboxing, heavy compute apps.

---

## 🔄 Communication Across Different Networks (Same EC2)

If **2 containers on the same EC2** are in **different networks**, like:

- Container A → in `bridge`
- Container B → in `host`

They **cannot communicate directly** using container names.

To enable communication:
- Use **host IP** + exposed port.
- Or connect both containers to the **same user-defined bridge network**.

👉 Example:

```bash
docker network create mynet
docker run -dit --name c1 --network mynet nginx
docker run -dit --name c2 --network mynet alpine
```

Now they can ping each other using container names.

---

this is a real-world architecture scenario in AWS. Let’s break down how the **WordPress container in a public subnet** communicates with the **MySQL container in a private subnet**, even though the private subnet has **no internet access and no NAT Gateway**.

---

## 🗺️ High-Level Architecture

| Component | Location | Details |
|----------|----------|---------|
| EC2 A (Public Subnet) | Public Subnet | Runs WordPress container |
| EC2 B (Private Subnet) | Private Subnet (no NAT/Internet) | Runs MySQL container |
| VPC | Same VPC | Both subnets are part of the same VPC |
| Docker Network | On each EC2, isolated to that EC2 | Docker containers use **bridge** or **host** network (not across EC2s) |

---

## 🔗 How Communication Happens

- **Docker containers on different EC2s do not directly talk Docker-to-Docker**.
- Instead, **WordPress container talks to MySQL using EC2 B’s private IP address**, like this:

```bash
DB_HOST=10.0.2.45  # EC2 B's private IP
```

### So what's the real communication flow?

> WordPress (container) → EC2 A network interface → VPC routing → EC2 B private IP → Docker on EC2 B → MySQL (container)

---

| Item | Value |
|------|-------|
| Communication Path | WordPress (Docker) → EC2 A → VPC → EC2 B → MySQL (Docker) |
| Docker Networking | **Bridge network (local only inside ec2 for container)** |
| Cross-machine Network | **AWS VPC network using private IPs** |
| Internet Needed? | ❌ No |
| NAT Needed? | ❌ No |
| Use case match? | ✅ Yes – this is best practice for secure DB access |

---
Excellent follow-up! This is the **exact reasoning** DevOps engineers must master — let’s clarify it fully.

---

## ✅ Yes, if EC2 A can talk to EC2 B, then…

**Apps running inside Docker containers on those EC2s _can also communicate_** — because:

> Docker containers **use the EC2's networking stack** to reach outside the host.

---

### 🔍 Let’s break it down with an analogy:

Imagine:

- EC2 = apartment building
- Docker container = a tenant in one of the rooms
- VPC = neighborhood
- Communication = calling a phone number

If EC2 A can “call” EC2 B in the same VPC (like neighbors talking), then the **people (apps inside containers)** inside those EC2s can also talk — **using the EC2's phone line (network interface).**

---

## 🔧 Technically, how does it work?

Let’s say:

- EC2 A has WordPress container
- EC2 B has MySQL container

The **WordPress app inside its container** is configured like this:

```php
define('DB_HOST', '10.0.2.45');  // EC2 B's private IP
define('DB_PORT', '3306');
```

When WordPress tries to connect to `10.0.2.45:3306`, here's what happens:

### 🔁 Under the hood:

1. **Docker container on EC2 A** (WordPress) sends a TCP packet to `10.0.2.45:3306`.
2. That packet goes via **bridge network → EC2 A’s eth0 → VPC internal network**.
3. It reaches **EC2 B**, which is listening on port 3306 (MySQL container).
4. **Docker on EC2 B** routes this to the container using `-p 3306:3306`.

✅ Done. Communication successful.  
🧠 **Docker bridge network only handles internal routing _inside_ a single EC2.**

---

## 💡 Why a Docker network is **not required across EC2s**

Because:

| Purpose | Handled By |
|--------|-------------|
| Cross-host traffic | **AWS VPC** routing (uses EC2 private IPs) |
| Container-to-container on same host | **Docker network** (bridge, host, etc.) |
| Port access from outside container | Docker’s `-p` or `--publish` option |

So unless you're using **Docker Swarm** (with overlay network), **you don't need Docker networks across hosts** — because **Docker doesn't natively support cross-host container DNS/discovery.**

---

### 🚨 Important Note:

For EC2 A’s container to reach EC2 B’s container, **EC2 B must publish the container’s port** using:

```bash
docker run -p 3306:3306 mysql
```

Otherwise, the container won't be reachable from outside its own EC2 if there will be no port mapping/forwarding through **-p process**.

---

## ✅ Summary

| Question | Answer |
|---------|--------|
| Can WordPress (in container on EC2 A) talk to MySQL (in container on EC2 B)? | ✅ Yes |
| How? | Through EC2's private IP using VPC internal network |
| Is Docker network used across EC2s? | ❌ No |
| Why not? | Docker networks (like bridge) are **host-local**. VPC handles cross-host routing |
| What’s needed? | Proper `docker run -p`, security group rules, and private IPs |

---


