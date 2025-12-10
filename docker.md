# 📘 **TOPIC 6 — DOCKER CHEAT SHEET (Junior → Intermediate Level)**

---

## **1. Dockerfile Basics & Best Practices**

* Use minimal base images (`python:3.10-slim`, not `latest`)
* Use `.dockerignore` to reduce build context
* Order instructions to maximize cache

  ```
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  COPY . .
  ```
* Avoid running as root → create a non-root user
* Pin versions for reproducibility
* Use multi-stage builds to reduce image size

---

## **2. Multi-Stage Build Pattern**

```
FROM golang:1.20 AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.18
COPY --from=builder /app/server .
ENTRYPOINT ["./server"]
```

**Why it's useful:**

* Smaller, cleaner image
* Removes build tools
* Faster deployments
* Improved security

---

## **3. ENTRYPOINT vs CMD**

### ENTRYPOINT

* Defines the **main executable**
* Not overridden unless `--entrypoint` is used

### CMD

* Provides **default arguments** or a fallback command
* Overridden by `docker run <args>`

**Model Answer:**

> “ENTRYPOINT sets the fixed command the container always runs. CMD provides default arguments that the user can override.”

---

## **4. ARG vs ENV**

### ARG

* Build-time variable
* Not available at runtime unless passed into ENV
* Set using `--build-arg`

### ENV

* Runtime environment variable
* Available to the container process

**Conversion:**

```
ARG APP_ENV
ENV APP_ENV=$APP_ENV
```

---

## **5. Docker Networking**

### Bridge (default)

* Private subnet (172.x.x.x)
* Port mapping required for host access
* Containers communicate via private IPs

### Host

* No isolation; container uses host network
* No port mapping needed
* Fastest networking

### None

* No external network
* Isolation mode

### Container

* Shares another container’s network namespace

### Overlay

* Multi-host networking (Docker Swarm)

**Model Answer:**

> “Services inside Compose don’t need port mapping. They communicate over an internal bridge network using service names.”

---

## **6. Docker Compose Core Concepts**

### Services

Each container defined in YAML.

### Networking

* Compose auto-creates a private network
* Containers communicate via service names: `db:3306`

### Volumes

**Bind Mounts:** Host ↔ Container
**Named Volumes:** Docker-managed, persistent.

### env_file

Load environment variables from `.env` or a custom file.

**Model Answer:**

> “Use bind mounts for development, named volumes for persistent database storage.”

---

## **7. Debugging Containers**

### View logs:

```
docker logs -f <container>
```

### Enter container:

```
docker exec -it <container> sh
```

### Inspect metadata:

```
docker inspect <container>
```

### Process check:

```
docker top <container>
```

---

## **8. Common Container Failure Scenarios**

### Container exits immediately

* Main process ends
* Wrong CMD or ENTRYPOINT
* Missing environment variables
* Permission errors

**Debug:** `docker logs`, `docker exec`, run interactively.

### Port issues

* App listening on 127.0.0.1 instead of 0.0.0.0

### Service-to-service connection fails

* Using localhost inside container → wrong
* Use service names instead

**Model Answer:**

> “Containers stop when their main process exits. I debug using logs, `docker exec`, and checking ENTRYPOINT/CMD.”

---
