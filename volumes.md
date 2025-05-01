# Docker Volumes

## Problem Statement

It is a very common requirement to persist the data in a Docker container beyond the lifetime of the container. However, the file system
of a Docker container is deleted/removed when the container dies. 

## Solution

There are 2 different ways how docker solves this problem.

1. Volumes
2. Bind Directory on a host as a Mount

### Volumes 

Volumes aims to solve the same problem by providing a way to store data on the host file system, separate from the container's file system, 
so that the data can persist even if the container is deleted and recreated.

![image](https://user-images.githubusercontent.com/43399466/218018334-286d8949-d155-4d55-80bc-24827b02f9b1.png)


Volumes can be created and managed using the docker volume command. You can create a new volume using the following command:

```
docker volume create <volume_name>
```

Once a volume is created, you can mount it to a container using the -v or --mount option when running a docker run command. 

For example:

```
docker run -it -v <volume_name>:/data <image_name> /bin/bash
```

This command will mount the volume <volume_name> to the /data directory in the container. Any data written to the /data directory
inside the container will be persisted in the volume on the host file system.

### Bind Directory on a host as a Mount

Bind mounts also aims to solve the same problem but in a complete different way.

Using this way, user can mount a directory from the host file system into a container. Bind mounts have the same behavior as volumes, but
are specified using a host path instead of a volume name. 

For example, 

```
docker run -it -v <host_path>:<container_path> <image_name> /bin/bash
```

## Key Differences between Volumes and Bind Directory on a host as a Mount

Volumes are managed, created, mounted and deleted using the Docker API. However, Volumes are more flexible than bind mounts, as 
they can be managed and backed up separately from the host file system, and can be moved between containers and hosts.

In a nutshell, Bind Directory on a host as a Mount are appropriate for simple use cases where you need to mount a directory from the host file system into
a container, while volumes are better suited for more complex use cases where you need more control over the data being persisted
in the container.

### **Persistent Storage Problem in Docker – Explained with Examples** 🚀  

Docker containers are **ephemeral**, meaning that any data stored inside the container is **lost when the container stops or restarts**. This creates issues when applications need **persistent storage** (data that survives container restarts).  

Let’s understand this problem with **three real-world examples** and how **bind mounts & volumes** solve it.  

---

## **📝 Example 1: Nginx Losing Log Files After Restart**  
Imagine you are running an Nginx container:  
```bash
docker run -d --name nginx -p 80:80 nginx
```
📂 Nginx logs requests in `/var/log/nginx/access.log`.  

### **❌ Problem:**  
- When the Nginx container **stops or restarts**, all logs **inside the container** are lost.  
- If you want to debug requests, you cannot access logs after the container is removed.  

### **✅ Solution: Bind Mount**  
Bind mounts allow you to **store logs on the host machine**, ensuring they persist across restarts.  

```bash
docker run -d --name nginx -p 80:80 -v /host/path/nginx_logs:/var/log/nginx nginx
```
🔹 **Now, logs are stored in `/host/path/nginx_logs` on the host machine and remain even if the container is deleted.**  

---

## **📝 Example 2: Backend Database Losing Data When Container Crashes**  
Consider a **backend API** running inside a Docker container that stores user data in a SQLite database:  

```bash
docker run -d --name backend -p 5000:5000 my-backend
```
The backend writes data to **`/app/database.db` inside the container**.  

### **❌ Problem:**  
- If the container **crashes, stops, or is recreated**, all user data stored in `database.db` is **lost**.  
- Users will have to start over, which is unacceptable for real applications.  

### **✅ Solution: Docker Volume**  
Volumes provide a **persistent storage solution managed by Docker**, separate from the container.  

```bash
docker volume create backend_data
docker run -d --name backend -p 5000:5000 -v backend_data:/app my-backend
```
🔹 **Now, user data is stored in `backend_data`, and even if the container is removed, the data remains.**  

---

## **📝 Example 3: Cron Job Generating Reports That Disappear**  
Suppose you have a **cron job** running inside a container that generates a report every hour and saves it to `/reports/report.txt`.  

```bash
docker run -d --name report-job cronjob-image
```
### **❌ Problem:**  
- Each time the cron job runs in a **new container**, previous reports are **lost** because they were stored inside a temporary container.  
- You need a way to **persist reports** across container restarts.  

### **✅ Solution: Use a Volume to Store Reports**  
```bash
docker volume create report_storage
docker run -d --name report-job -v report_storage:/reports cronjob-image
```
🔹 **Now, reports are stored in `report_storage`, ensuring persistence even if the container is recreated.**  

---

## **🚀 Key Differences: Bind Mounts vs Volumes**
| Feature            | Bind Mounts 📂 | Docker Volumes 📦 |
|-------------------|--------------|----------------|
| Stored In | Any path on the host | Managed by Docker |
| Performance | Slower | Optimized for Docker |
| Best For | Accessing specific host files | Storing persistent app data |
| Example | Logs (`/var/log/nginx`) | Database (`backend_data`) |

---

## **Summary**
✔ **Nginx logs lost?** ➝ Use **bind mounts** to store logs on the host.  
✔ **Backend database lost?** ➝ Use **volumes** to persist data across restarts.  
✔ **Cron job reports disappearing?** ➝ Use **volumes** to store generated files permanently.  

