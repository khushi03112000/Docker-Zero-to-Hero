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


