##  `ENTRYPOINT` and `CMD` only run when the container is **run**, not when the image is **built**.

### 🔧 Summary:

| Dockerfile Instruction | When It Executes       |
|------------------------|------------------------|
| `RUN`                  | During `docker build`  |
| `CMD`                  | During `docker run`    |
| `ENTRYPOINT`           | During `docker run`    |

---

### 🛠️ During `docker build`:
Only instructions like:
- `FROM`
- `COPY`, `ADD`
- `RUN`
- `ENV`
- `WORKDIR`
are executed.

📌 **`ENTRYPOINT` and `CMD` are just metadata saved into the image.**

They don’t run yet.

---

### ▶️ During `docker run`:
Docker takes the **saved ENTRYPOINT and CMD** and runs them together like:

```bash
[ENTRYPOINT] + [CMD or override from docker run]
```

---

### 💡 Example:

#### Dockerfile:

```Dockerfile
FROM alpine
RUN echo "Built the image"           # Executed at build time
ENTRYPOINT ["echo", "Hello"]
CMD ["World"]                        # Default argument
```

---

#### On build:

```bash
docker build -t my-image .
```

🟢 Output:
```
Built the image
```

No `echo Hello World` runs during build!

---

#### On run:

```bash
docker run my-image
```

🟢 Output:
```
Hello World
```

---

### 🚀 Run with override:

```bash
docker run my-image Universe
```

🟢 Output:
```
Hello Universe
```

---

### ✅ Final Takeaway:

- `RUN` = build-time
- `ENTRYPOINT` and `CMD` = run-time
- At `docker run`, Docker combines them:
  ```bash
  [ENTRYPOINT] + [CMD or override]
  ```
  Excellent question — you're getting close to the core idea of **multi-stage builds**.

Let’s break it down in **simple, clear steps** so you truly understand how **one Dockerfile with two `FROM` statements still creates a **small** image**.

---

## 🧠 First, Understand This Key Rule:

> **Only the final stage** in a multi-stage Dockerfile becomes the final image that Docker saves and tags — **not the whole Dockerfile.**

---

## 🔧 Let’s Walk Through a Multi-Stage Example

### 🐳 Dockerfile:

```Dockerfile
# Stage 1: Builder
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Final
FROM alpine:latest
COPY --from=builder /app/myapp /myapp
CMD ["/myapp"]
```

---

### 🔍 Step-by-Step What Happens:

#### ✅ Stage 1 (`golang:1.21 AS builder`):
- Docker **creates an intermediate container** with full Go compiler
- Builds the binary (`myapp`)
- This stage has a large image (~1GB)

BUT: It's **not part of the final image** — it's temporary!

---

#### ✅ Stage 2 (`FROM alpine:latest`):
- Docker starts **a new, clean image** from Alpine (~5MB)
- It **copies only the compiled binary** from Stage 1:
  ```Dockerfile
  COPY --from=builder /app/myapp /myapp
  ```
- Final image has:
  - ✅ only `/myapp`
  - ❌ no Go compiler
  - ❌ no source code
  - ❌ no package manager
  - ❌ no temporary files

 COPY --from=builder /app/myapp /myapp
🧠 Meaning:
--from=builder:
Tells Docker:
👉 “Take the file from the stage named builder (earlier in the Dockerfile)”

/app/myapp:
👉 The source file path inside the builder stage
(i.e., where the compiled binary is saved)

/myapp:
👉 The destination path inside the final image
(where this file will be placed)

---

## ✅ Result:

Even though the Dockerfile had 2 stages (and the first used a large image), **the final image only contains what’s in the last stage**, which is tiny and production-ready.

---

### 📦 When You Run:

```bash
docker build -t myapp .
```

You are tagging the image created from the **last `FROM` stage only**, not both.

So yes — to answer your question clearly:

> **Docker only keeps the image created by the last `FROM` block** in a multi-stage build. Previous stages are used temporarily during the build and are discarded (unless explicitly saved).

---

### 🔥 Bonus Tip: Reuse Multiple Stages

You can even have more than two stages, like:
```Dockerfile
FROM node AS builder
FROM python AS test-runner
FROM alpine AS final
```
Each stage can do a job (e.g., build, test, scan), but only the last one becomes your deliverable image.

---

