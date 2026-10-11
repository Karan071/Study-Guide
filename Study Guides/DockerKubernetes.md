# Docker and Kubernetes: A Backend Engineer's Reference

Audience: SDEs learning to package, run, network, secure, and operate containerized applications.
Scope: Core Docker and Kubernetes concepts, practical manifests and commands, production pitfalls, deployment guidance, and trusted learning resources.
Last reviewed: 2026-10-11. Kubernetes APIs and cloud-provider behavior evolve; verify version-specific details against the documentation for the version you run.

Related guides: [Authentication, Authorization & Modern Security Protocols](./AuthenticationConcepts.md) and [Caching Techniques](./CachingTechniques.md).

## Table of Contents

0. [System Map: From Source Code to Running Service](#0-system-map-from-source-code-to-running-service)
1. [Containers and Container Runtimes](#1-containers-and-container-runtimes)
2. [Docker Images and Registries](#2-docker-images-and-registries)
3. [Dockerfiles and Image Builds](#3-dockerfiles-and-image-builds)
4. [Build Context, Layers, and BuildKit](#4-build-context-layers-and-buildkit)
5. [Container Lifecycle and Runtime Configuration](#5-container-lifecycle-and-runtime-configuration)
6. [Container Networking](#6-container-networking)
7. [Volumes and Persistent Data](#7-volumes-and-persistent-data)
8. [Docker Compose](#8-docker-compose)
9. [Kubernetes Architecture](#9-kubernetes-architecture)
10. [Pods and Workload Design](#10-pods-and-workload-design)
11. [Deployments, ReplicaSets, and Rollouts](#11-deployments-replicasets-and-rollouts)
12. [Services and Service Discovery](#12-services-and-service-discovery)
13. [Ingress and Gateway APIs](#13-ingress-and-gateway-apis)
14. [Configuration and Secrets](#14-configuration-and-secrets)
15. [Health Checks and Probes](#15-health-checks-and-probes)
16. [Requests, Limits, and Scheduling](#16-requests-limits-and-scheduling)
17. [Autoscaling and Capacity](#17-autoscaling-and-capacity)
18. [Persistent Volumes and Stateful Workloads](#18-persistent-volumes-and-stateful-workloads)
19. [Security: Images, Containers, and Clusters](#19-security-images-containers-and-clusters)
20. [Observability and Troubleshooting](#20-observability-and-troubleshooting)
21. [Production Checklist and Cheat Sheet](#21-production-checklist-and-cheat-sheet)
22. [Self-Check Questions](#22-self-check-questions)
23. [Master Resource List](#23-master-resource-list)

---

## 0. System Map: From Source Code to Running Service

Docker packages an application and its user-space dependencies into an image. A container runtime starts an isolated process from that image. Kubernetes schedules and reconciles those containers across a cluster.

| Question | Layer | Concepts |
| --- | --- | --- |
| How is the application packaged? | Image build and distribution | 2, 3, 4 |
| How does a process run? | Container runtime | 1, 5 |
| How do containers communicate and store data? | Local runtime services | 6, 7, 8 |
| How does a cluster keep workloads running? | Kubernetes control plane and workloads | 9, 10, 11 |
| How do clients find workloads? | Cluster networking | 12, 13 |
| How is runtime behavior configured? | Configuration and health | 14, 15 |
| How are capacity and data managed? | Scheduling and storage | 16, 17, 18 |
| How do we reduce operational risk? | Security and observability | 19, 20, 21 |

### Build-to-production flow

```text
Source + dependencies
        |
        | Dockerfile + build context [3,4]
        v
OCI image (immutable content, tags are references) [2]
        |
        | push/pull through authenticated registry
        v
Kubernetes control plane [9]
  API server -> scheduler -> controller managers
        |
        | schedule Pod onto a node
        v
Node: kubelet + container runtime (containerd/CRI-O) [1,10]
  image -> container process + probes + resource limits
        |
        +--> Service / DNS --> other workloads [12]
        +--> Ingress or Gateway --> external clients [13]
        +--> PersistentVolume --> durable data [18]
        +--> logs, metrics, traces --> observability [20]
```

### Decision defaults

| Decision | Starting point |
| --- | --- |
| Image | Minimal, supported base; build once and promote the same image through environments |
| Identity | Pin deployment images by immutable digest; use tags for human-readable releases |
| Process | Run one primary service process in the foreground; handle termination signals |
| Configuration | Inject environment-specific configuration at runtime, not by rebuilding images |
| Kubernetes workload | `Deployment` for stateless services; `StatefulSet` only for stable identity/storage needs |
| Exposure | `ClusterIP` internally; use a managed ingress/gateway for external HTTP |
| Health | Startup, readiness, and liveness probes with distinct meanings |
| Capacity | Set realistic CPU/memory requests; set limits deliberately and load-test them |
| Secrets | Use a secret manager or encrypted Kubernetes Secret workflow; never bake credentials into images |
| Data | Use managed databases when practical; container writable layers are ephemeral |
| Operations | Observe rollouts and logs; test rollback, node drain, and dependency outages |

---

## 1. Containers and Container Runtimes

A container is an isolated process, not a small virtual machine. The Linux kernel provides primitives such as namespaces and cgroups; a runtime configures these boundaries and starts the process.

### Core mechanism

- Namespaces isolate views of resources such as processes, mounts, networking, and hostnames.
- Control groups (cgroups) account for and constrain resource consumption.
- The container filesystem is assembled from image layers plus a writable runtime layer.
- The process runs using the host kernel; a container image does not include a separate kernel.
- Docker Engine is one container platform. Kubernetes nodes commonly use CRI-compatible runtimes such as containerd or CRI-O.

### Engineering blueprint

```text
image + runtime configuration
  -> namespaces, cgroups, mounts, network
  -> container process (PID 1 in its namespace)
  -> exit status and runtime events
```

Keep the main service in the foreground so the runtime can observe its exit. If the container launches child processes, use an init process or proper signal forwarding and child reaping.

### Production pitfalls

- Containers are not a security boundary equivalent to a separate host or VM.
- A container can be killed for exceeding memory limits; a restart policy does not fix the underlying memory leak.
- Data written only to the writable container layer is lost when the container is replaced.
- Running multiple unrelated daemons in one container complicates health, signals, and lifecycle.

### SDE best practices

- Treat containers as replaceable processes and keep durable state outside their writable layer.
- Make shutdown graceful and bounded; stop accepting work, finish or cancel in-flight work, then exit.
- Set resource controls and inspect actual behavior under load.
- Learn the runtime used in production; Docker CLI presence is not required for a Kubernetes cluster to run containers.

### Resources

- [Docker: What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Kubernetes: Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec)

---

## 2. Docker Images and Registries

An image is an immutable, content-addressed template used to create containers. A registry stores and distributes images. A tag is a mutable name pointing to an image; a digest identifies exact content.

### Core mechanism

- Images are composed of filesystem layers and configuration metadata.
- Registries expose repositories and tags and may enforce authentication, scanning, signing, and retention.
- Pulling an image resolves a reference and downloads missing layers.
- OCI image manifests and indexes allow portable image distribution and multi-platform images.

```text
registry.example.com/team/service:1.8.2    # readable but mutable reference
registry.example.com/team/service@sha256:... # immutable content identity
```

### Engineering blueprint

- Use explicit registry, repository, and version references.
- Build and test an image once; promote the same digest between environments.
- Use multi-platform manifests only when the target CPU architectures require them.
- Configure registry authentication through short-lived credentials or workload identity where supported.
- Set retention policies that preserve deployed and rollback images.

### Production pitfalls

- `latest` is a mutable tag, not a guarantee of newest or reproducible content.
- A private registry outage or missing pull credentials can prevent new Pods from starting.
- Rebuilding the same source later may produce different bytes if dependencies or base images are mutable.
- A multi-architecture tag may resolve to a different platform-specific image on different nodes.

### SDE best practices

- Pin production workloads to image digests, optionally alongside a readable release tag.
- Record source revision, build provenance, and dependency versions as image metadata.
- Scan images and establish a process for rebuilding when base images receive security updates.
- Test image pulls with the same identity and network path used by cluster nodes.

### Resources

- [Docker: Images](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [Docker: Image digests](https://docs.docker.com/dhi/core-concepts/digests/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Kubernetes: Images](https://kubernetes.io/docs/concepts/containers/images/)

---

## 3. Dockerfiles and Image Builds

A Dockerfile describes instructions for building an image. A reliable Dockerfile makes the runtime artifact reproducible, efficient to distribute, and safe by default.

### Core mechanism

- `FROM` selects a base image.
- `WORKDIR`, `COPY`, and build instructions construct filesystem layers.
- `RUN` executes commands during build; `CMD` provides a default runtime command; `ENTRYPOINT` defines the executable.
- Build-time `ARG` and runtime `ENV` are different interfaces; neither should be used for secrets.
- Multi-stage builds separate compilation tools from the final runtime image.

### Engineering blueprint

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:24-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=build --chown=node:node /app/package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The exact build and runtime commands depend on the application. Keep the final stage limited to runtime dependencies and required artifacts.

Shell form vs exec form, the most common signal-handling bug:

```dockerfile
CMD node dist/server.js            # shell form: runs /bin/sh -c "...", sh is PID 1,
                                   # SIGTERM goes to sh and often never reaches node
CMD ["node", "dist/server.js"]     # exec form: node is PID 1 and receives SIGTERM directly
```

`ENTRYPOINT` vs `CMD`: `ENTRYPOINT` is the fixed executable; `CMD` supplies default arguments that `docker run image <args>` replaces. With `ENTRYPOINT ["python", "app.py"]` and `CMD ["--port", "8080"]`, running `docker run image --port 9090` executes `python app.py --port 9090`.

Docker's `HEALTHCHECK` instruction is used by `docker run` and Compose, but **Kubernetes ignores it**; define probes in the Pod spec instead (see section 15).

### Production pitfalls

- Copying the entire source tree before dependency installation invalidates expensive cache layers on every code change.
- Secrets passed through `ARG`, `ENV`, or copied files can remain in image history or layers.
- A base image tag can move, silently changing future builds.
- Shell-form commands can interfere with signal delivery and exit-code handling.
- A build stage may accidentally leave compilers, package caches, tests, or credentials in the final image.

### SDE best practices

- Use `.dockerignore` to exclude `.git`, local dependencies, secrets, build outputs, and editor files.
- Prefer multi-stage builds and non-root runtime users.
- Use BuildKit secret mounts for private build credentials; do not persist secrets in image layers.
- Use exec-form `ENTRYPOINT`/`CMD` so the application receives signals as expected.
- Pin critical dependencies and base images; rebuild intentionally to consume security updates.
- Add image build, test, scan, and provenance steps to CI.

### Resources

- [Docker: Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker: Build best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker: Build secrets](https://docs.docker.com/build/building/secrets/)

---

## 4. Build Context, Layers, and BuildKit

The build context is the set of files made available to the builder. Layers and build cache allow unchanged build steps to be reused, but cache reuse depends on instruction order and inputs.

### Core mechanism

- `docker build` sends or exposes a context to the builder; `COPY` and `ADD` can only read eligible context files.
- `.dockerignore` excludes files from the context.
- Filesystem-changing build steps can create layers; later layers build on earlier ones.
- BuildKit provides parallel, efficient builds, cache mounts, secret mounts, and advanced build features.
- A cached step is valid only while the relevant instruction and inputs remain equivalent.

### Engineering blueprint

Order steps from stable to frequently changing:

```dockerfile
COPY package-lock.json package.json ./
RUN npm ci
COPY src ./src
RUN npm run build
```

Use build cache as a performance optimization, not as a correctness dependency. CI should be able to produce a valid image with a cold cache.

### Production pitfalls

- A large context slows local and remote builds and can unintentionally include secrets.
- `COPY . .` can copy local artifacts that differ from CI or production expectations.
- Cache mounts are optimizations and may be empty or unavailable; builds must still succeed.
- Reordering instructions without understanding cache invalidation can make builds slow or stale.

### SDE best practices

- Review `.dockerignore` as part of the security boundary.
- Build from a clean checkout in CI and reproduce locally with the same context.
- Keep dependency manifests copied before application source when that improves cache reuse.
- Use BuildKit features deliberately and document required builder versions.

### Resources

- [Docker: Build context](https://docs.docker.com/build/concepts/context/)
- [Docker: Build cache](https://docs.docker.com/build/cache/)
- [Docker: BuildKit](https://docs.docker.com/build/buildkit/)
- [Docker: `.dockerignore`](https://docs.docker.com/build/concepts/context/#dockerignore-files)

---

## 5. Container Lifecycle and Runtime Configuration

A container is created from an image with runtime configuration for its command, environment, mounts, network, user, and resource limits. Container state is separate from image content.

### Core mechanism

```bash
docker build -t example-api:dev .
docker run --rm --name example-api -p 8080:3000 example-api:dev
docker logs example-api
docker stop example-api
```

- `docker run` creates and starts a container.
- `-p` publishes a host port to a container port.
- `--rm` removes the container after it exits.
- `docker stop` sends a termination signal and eventually force-kills after its grace period.

Everyday Docker commands:

| Need | Command |
| --- | --- |
| List running / all containers | `docker ps` / `docker ps -a` |
| Shell into a running container | `docker exec -it NAME sh` |
| Follow logs | `docker logs -f --tail 100 NAME` |
| Inspect config, mounts, network, exit code | `docker inspect NAME` |
| Live CPU / memory per container | `docker stats` |
| Show image layers and their sizes | `docker history IMAGE` |
| Set resource limits | `docker run --memory 512m --cpus 1 IMAGE` |
| Build for another CPU architecture | `docker buildx build --platform linux/amd64,linux/arm64 -t IMAGE --push .` |
| Reclaim disk space | `docker system df`, then `docker system prune` (removes stopped containers, unused networks, dangling images) |

Exit codes you will see: `0` clean exit; `1` application error; `125`–`127` Docker or command-not-found problems; `137` = 128 + 9 (SIGKILL, often an out-of-memory kill or a forced stop); `143` = 128 + 15 (SIGTERM, normal stop).

### Engineering blueprint

- Pass non-secret configuration at runtime.
- Define a shutdown grace period long enough for request draining but short enough for recovery.
- Make the process emit logs to stdout/stderr or use the platform's logging driver.
- Use restart policies for process recovery, not as a substitute for health checks and root-cause fixes.

### Production pitfalls

- Container port exposure is not the same as host or cluster network exposure.
- A container that exits successfully may still indicate a failed service if its main process daemonized.
- `docker exec` is for troubleshooting, not a persistent production change mechanism.
- Changing a running container manually creates configuration drift from its image and deployment definition.

### SDE best practices

- Recreate containers from declarative configuration rather than patching them in place.
- Validate required configuration at startup and use consistent variable names.
- Handle `SIGTERM` and test actual shutdown behavior.
- Limit diagnostic access and do not print secrets in environment dumps or logs.

### Resources

- [Docker: `docker run`](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker: `docker stop`](https://docs.docker.com/reference/cli/docker/container/stop/)
- [Docker: Container logs](https://docs.docker.com/engine/logging/)

---

## 6. Container Networking

Container networking provides connectivity between containers, the host, and external networks. Docker networking and Kubernetes networking are related operationally but use different abstractions.

### Core mechanism

- Docker bridge networks connect containers on a host; user-defined bridges provide name-based discovery.
- Port publishing maps a host port to a container port.
- Kubernetes gives each Pod an IP and provides cluster networking between Pods; a Service supplies a stable virtual address for changing Pod endpoints.
- Network policies can restrict Pod-to-Pod traffic when supported by the cluster network plugin.

### Engineering blueprint

```bash
docker network create app-net
docker run -d --name db --network app-net postgres:16
docker run --rm --network app-net -e DB_HOST=db example-api:dev
```

In Kubernetes, use a `Service` name rather than hard-coding Pod IP addresses. Apply network policies based on required communication paths and confirm the CNI plugin enforces them.

### Production pitfalls

- `localhost` inside a container refers to that same container, not another container or the host.
- A service listening only on `127.0.0.1` may not accept traffic arriving through its container interface.
- Publishing every port exposes services unnecessarily.
- DNS caches and connection pools can retain old endpoints longer than expected.
- Kubernetes `NetworkPolicy` has no effect if the installed network implementation does not enforce it.

### SDE best practices

- Bind network services to the appropriate interface (`0.0.0.0` in many containerized server cases).
- Expose only required ports and use private networks by default.
- Use service discovery names, readiness signals, and resilient connection handling.
- Verify DNS, routing, and policy behavior from the actual workload network context.

### Resources

- [Docker: Networking overview](https://docs.docker.com/engine/network/)
- [Kubernetes: Cluster networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Kubernetes: Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

---

## 7. Volumes and Persistent Data

Container writable layers are ephemeral. Docker volumes and bind mounts provide data outside that layer; Kubernetes persistent volumes provide storage through cluster storage abstractions.

### Core mechanism

- A **volume** is managed by Docker and survives a container's removal, subject to its lifecycle and driver.
- A **bind mount** maps a host path into a container and couples the container to host filesystem layout.
- In Kubernetes, a `PersistentVolumeClaim` requests storage; a provisioner or administrator supplies a `PersistentVolume`.
- Storage behavior depends on access mode, storage class, filesystem, and provider.

### Engineering blueprint

```bash
docker volume create db-data
docker run -d --name db -v db-data:/var/lib/postgresql/data postgres:16
```

Use volumes for local development and stateful experiments. For production databases, prefer a managed database where practical; otherwise design backups, restore tests, replication, and upgrades explicitly.

### Production pitfalls

- A persistent volume is not automatically a backup.
- Deleting a claim can trigger deletion of the backing disk depending on reclaim policy.
- A volume may attach to only one node or zone at a time.
- Bind mounts can expose sensitive host paths or create ownership/permission mismatches.
- Container replacement does not mean attached storage is safe from corruption or operator error.

### SDE best practices

- Separate compute lifecycle from data lifecycle.
- Understand persistence, access modes, encryption, snapshots, and reclaim policy before production use.
- Test restore from backup and disaster-recovery procedures.
- Mount only the required paths and use appropriate filesystem permissions.

### Resources

- [Docker: Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Docker: Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Kubernetes: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

---

## 8. Docker Compose

Docker Compose describes and runs a multi-container application, commonly for local development, testing, and small single-host deployments. Compose is not a Kubernetes cluster orchestrator.

### Core mechanism

- A Compose YAML file declares services, networks, volumes, environment, and dependencies.
- Compose creates and manages resources for the project.
- Service names are available as DNS names on the Compose network.
- Health checks can express service health; `depends_on` can express startup ordering/health conditions, not full runtime resilience.

### Engineering blueprint

```yaml
services:
  api:
    build: .
    ports:
      - "8080:3000"
    environment:
      DATABASE_URL: postgres://app:dev-only@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev-only
      POSTGRES_DB: app
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  db-data:
```

The sample credentials are only for disposable local development; use a secret-management approach for deployed environments.

### Production pitfalls

- Startup order does not guarantee the dependency remains healthy.
- Local volumes and developer credentials can be mistaken for production-ready persistence and secrets.
- Compose services run on a single Docker host; availability and rescheduling guarantees differ from a cluster orchestrator.
- YAML interpolation and environment precedence can surprise across shell and `.env` files.

### SDE best practices

- Keep Compose configuration as reproducible developer tooling.
- Provide health checks and ensure applications retry transient dependency startup failures.
- Keep secrets out of committed Compose files and `.env` files.
- Validate the file with `docker compose config` and document required setup.

### Resources

- [Docker Compose documentation](https://docs.docker.com/compose/)
- [Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Docker: Compose networking](https://docs.docker.com/compose/how-tos/networking/)

---

## 9. Kubernetes Architecture

Kubernetes is a declarative orchestration system. Users submit desired state through the API; controllers continuously compare that state with observed state and take actions to reconcile differences.

### Core mechanism

Control plane:

- **API server** validates and serves the Kubernetes API.
- **etcd** stores cluster state in many self-managed installations; managed services operate this component for users.
- **Scheduler** assigns unscheduled Pods to nodes.
- **Controller manager** runs controllers that reconcile resources.

Node components:

- **kubelet** ensures Pods assigned to a node are running.
- **Container runtime** starts and manages containers.
- **kube-proxy** or an alternative data plane implements Service traffic routing, depending on cluster configuration.

### Engineering blueprint

```text
kubectl apply -f deployment.yaml
        |
        v
API server stores desired Deployment
        |
        v
Deployment controller creates/updates ReplicaSet
        |
        v
ReplicaSet controller requests Pods
        |
        v
Scheduler binds Pods to suitable nodes
        |
        v
kubelet + runtime start containers and report status
```

### Production pitfalls

- Applying a manifest successfully means the API accepted it, not that the workload is healthy.
- Controllers may retry indefinitely when quotas, image pulls, scheduling, or admission policies prevent progress.
- A healthy control plane does not guarantee application data-plane connectivity.
- Kubernetes does not make an application highly available if all replicas share one node, zone, or external dependency.

### SDE best practices

- Observe resource status and events after applying changes.
- Learn the cluster's version, CNI, CSI, ingress/gateway, admission, and policy implementations.
- Prefer managed control planes unless there is a strong reason to operate one.
- Define ownership and access boundaries with namespaces, RBAC, quotas, and policy.

### Resources

- [Kubernetes: Concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes: Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes: Architecture](https://kubernetes.io/docs/concepts/architecture/)

---

## 10. Pods and Workload Design

A Pod is Kubernetes' smallest deployable unit. It contains one or more tightly coupled containers that share network identity and can share storage.

### Core mechanism

- Containers in a Pod share the Pod IP and port space and can communicate over `localhost`.
- Containers can mount the same Pod volumes.
- Pods are ephemeral; a replacement Pod normally receives a different name and IP.
- Workload controllers create and replace Pods according to desired state.

### Engineering blueprint

Use one primary application container per Pod by default. Multiple containers make sense for tightly coupled helper patterns such as a proxy, log sidecar, or init container, but add lifecycle and resource overhead.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-example
spec:
  containers:
    - name: api
      image: registry.example.com/team/api@sha256:REPLACE_WITH_DIGEST
      ports:
        - name: http
          containerPort: 8080
```

Direct Pod manifests are useful for learning and diagnostics. Production stateless applications generally belong in a `Deployment`, which manages Pods.

Choosing a workload controller:

| Controller | Use for | Example |
| --- | --- | --- |
| `Deployment` | Stateless, interchangeable replicas | Web APIs, frontends, stateless workers |
| `StatefulSet` | Stable names, ordered startup, per-replica storage | Databases, Kafka, ZooKeeper, Elasticsearch |
| `DaemonSet` | Exactly one Pod per node (or per matching node) | Log shippers, node monitoring agents, CNI plugins |
| `Job` | Run to completion, with retries | Database migration, one-off batch processing |
| `CronJob` | `Job` on a schedule | Nightly reports, cleanup tasks |

Since Kubernetes v1.33, **native sidecar containers** are stable: declare a helper in `initContainers` with `restartPolicy: Always`. It starts before the app containers, keeps running alongside them, and stops after them. This fixes old problems such as a log or proxy sidecar exiting before the main app finished, or a `Job` never completing because its sidecar kept running.

### Production pitfalls

- Storing a Pod IP in configuration breaks when the Pod is replaced.
- A multi-container Pod shares fate: scheduling, restart, and scaling occur at the Pod level.
- Debugging an ephemeral Pod after it exits may be difficult without logs and events.
- Setting `containerPort` documents a port but does not expose the Pod externally.

### SDE best practices

- Keep Pods replaceable; externalize durable state.
- Use controllers for production workloads.
- Add labels consistently for selection and operations.
- Use init containers for ordered setup tasks that must finish before app containers start.

### Resources

- [Kubernetes: Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes: Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Kubernetes: Sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Kubernetes: Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/), [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/), [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

---

## 11. Deployments, ReplicaSets, and Rollouts

A Deployment manages ReplicaSets, which maintain a desired number of matching Pods. Deployments provide declarative updates and rollout history for stateless applications.

### Core mechanism

- A Deployment's Pod template describes the desired workload.
- A ReplicaSet creates or deletes Pods to match its replica count.
- A rollout creates a new ReplicaSet and gradually replaces old Pods according to the update strategy.
- Rollback restores a prior Pod template revision, not necessarily external database or API changes.

### Engineering blueprint

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: registry.example.com/team/api@sha256:REPLACE_WITH_DIGEST
          ports:
            - name: http
              containerPort: 8080
```

The selector must match the template labels and should be treated as stable identity.

Three replicas do not give high availability if the scheduler puts them on the same node, and a node drain can take them all down at once. Add spread constraints and a disruption budget:

```yaml
# In the Deployment's Pod template spec:
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api
spec:
  minAvailable: 2          # voluntary disruptions (node drains, upgrades) must leave 2 Pods running
  selector:
    matchLabels:
      app: api
```

A PodDisruptionBudget protects only against **voluntary** disruptions such as `kubectl drain` and cluster upgrades. It does not prevent node crashes.

Useful rollout commands:

```bash
kubectl set image deployment/api api=registry.example.com/team/api@sha256:NEW_DIGEST
kubectl rollout status deployment/api       # wait for success or failure
kubectl rollout history deployment/api      # list revisions
kubectl rollout undo deployment/api         # go back one revision
kubectl rollout restart deployment/api      # recreate Pods, e.g. to pick up changed Secrets
```

Rolling updates are the built-in strategy. Blue-green (run the new version alongside the old, then switch the Service selector) and canary (send a small share of traffic to the new version, then increase it) need either two Deployments or a tool such as Argo Rollouts, Flagger, or a service mesh.

### Production pitfalls

- A rollout can stall when new Pods fail readiness, cannot pull images, or cannot schedule.
- `maxUnavailable: 0` may require spare cluster capacity for `maxSurge`.
- A rollback may not reverse a database migration or external side effect.
- Changing selectors or labels carelessly can orphan or unexpectedly adopt Pods.

### SDE best practices

- Use backward-compatible expand/contract database migrations.
- Set rollout strategy based on capacity and availability objectives.
- Monitor rollout status and stop promotion when error or latency budgets are exceeded.
- Keep a known-good image digest and practice rollback.

### Resources

- [Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [kubectl: `rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
- [Kubernetes: Rolling updates](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- [Kubernetes: Disruptions and PodDisruptionBudgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes: Topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

---

## 12. Services and Service Discovery

A Service provides a stable network abstraction for a changing set of Pods. It selects endpoints, normally by label, and supplies a stable virtual IP and DNS name within the cluster.

### Core mechanism

- `ClusterIP` exposes a virtual IP inside the cluster.
- `NodePort` exposes a port on cluster nodes and is usually an implementation building block rather than the preferred public interface.
- `LoadBalancer` requests an external load balancer when supported by the cluster.
- `ExternalName` maps a Service name to an external DNS name; it does not proxy traffic.
- EndpointSlices represent the backends selected by a Service.

### Engineering blueprint

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - name: http
      port: 80
      targetPort: http
```

Other Pods can use `http://api` in the same namespace, or `http://api.<namespace>.svc` across namespaces.

### Production pitfalls

- A selector mismatch produces a Service with no ready endpoints.
- `port`, `targetPort`, and container listening port are distinct values.
- A Service does not by itself provide application-level retries or request draining.
- Client DNS caching and long-lived connections can delay endpoint changes.

### SDE best practices

- Use named ports and consistent labels.
- Check EndpointSlices when a Service is unreachable.
- Use readiness to remove unready Pods from Service endpoints.
- Keep internal services private unless external access is required.

### Resources

- [Kubernetes: Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes: DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes: EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

---

## 13. Ingress and Gateway APIs

Ingress and Gateway APIs describe HTTP/TCP routing from outside a cluster to Services. They require a controller or implementation that realizes those API resources.

### Core mechanism

- An Ingress resource describes HTTP host/path routing; it is not an implementation on its own.
- A Gateway API separates infrastructure ownership (GatewayClass/Gateway) from route ownership (HTTPRoute and related resources).
- Controllers integrate with cloud load balancers, proxies, certificates, and policy systems.
- TLS termination can occur at the edge, at an in-cluster proxy, or end-to-end depending on design.

**Status in 2026:** the Ingress API is stable but frozen; new features go into Gateway API. The widely used community **ingress-nginx** controller was retired: best-effort maintenance ended in March 2026, and it no longer receives releases or security fixes. Existing installs keep working, but plan a migration to a maintained Gateway API implementation or another supported controller.

A Gateway API route:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
spec:
  parentRefs:
    - name: public-gateway          # Gateway owned by the platform team
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api                 # the Service from section 12
          port: 80
```

The platform team owns the `Gateway` (listeners, TLS certificates, load balancer); application teams own `HTTPRoute` objects in their own namespaces. Ingress puts all of this in one object.

### Engineering blueprint

- Select a maintained controller supported by the platform.
- Define DNS, TLS certificate issuance/renewal, health checks, timeouts, body limits, and source IP handling.
- Use explicit hostnames and routes; avoid broad wildcard exposure without a use case.
- Test redirects, path rewriting, websocket/streaming behavior, and forwarded headers.

### Production pitfalls

- Creating an Ingress without a controller may do nothing.
- Incorrect forwarded headers can break client IP auditing, HTTPS redirects, or secure-cookie behavior.
- Ingress class/controller changes can alter routing and TLS behavior.
- A route can be accepted while its backend Service has no endpoints.

### SDE best practices

- Confirm which API and controller are supported by the cluster version/provider.
- Treat TLS private keys and routing configuration as sensitive.
- Monitor certificate expiry and load-balancer health.
- Prefer Gateway API for new designs when its implementation support and platform standards justify it.

### Resources

- [Kubernetes: Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Kubernetes Blog: Ingress NGINX Retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Kubernetes: Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

---

## 14. Configuration and Secrets

Configuration should vary by environment without requiring a different application image. ConfigMaps hold non-confidential configuration; Kubernetes Secrets are intended for confidential values but require additional security controls.

### Core mechanism

- ConfigMaps and Secrets can be exposed as environment variables or mounted files.
- Environment variables are fixed for a running process; changing the source object does not automatically restart the process.
- Mounted projected files may update eventually, but application reload behavior is separate.
- Secrets are base64-encoded in manifests by default, not thereby encrypted.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: info
  FEATURE_X_ENABLED: "true"
---
# In the container spec:
envFrom:
  - configMapRef:
      name: api-config               # every key becomes an environment variable
env:
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: api-db                 # Secret created by your secret-management tooling
        key: password
```

### Engineering blueprint

- Keep defaults safe and validate required configuration at startup.
- Use an external secret manager or encryption-at-rest integration for sensitive production secrets.
- Restrict Secret read permissions with RBAC; avoid broad namespace-level access.
- Mount a secret only into containers that need it.
- Roll workloads when configuration changes if the app does not reload changes safely.

### Production pitfalls

- Committing credentials in YAML, Docker build arguments, image layers, or Git history exposes them.
- Base64 is encoding, not encryption.
- Environment values can leak through crash reports, diagnostics, or process inspection.
- A Secret is not automatically safe if etcd encryption and access controls are misconfigured.

### SDE best practices

- Never bake environment-specific credentials into an image.
- Use workload identity for cloud APIs where supported instead of static long-lived keys.
- Rotate credentials and test rotation without downtime.
- Separate config release changes from image builds when appropriate, and track which version is active.

### Resources

- [Kubernetes: ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes: Encrypting Secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [External Secrets Operator](https://external-secrets.io/)

---

## 15. Health Checks and Probes

Kubernetes probes let the kubelet evaluate container startup and health. Startup, liveness, and readiness probes have different purposes and should not be treated as interchangeable.

### Core mechanism

- **Startup probe:** gives slow-starting applications time to initialize; other probes are held until startup succeeds.
- **Liveness probe:** detects a process that should be restarted.
- **Readiness probe:** determines whether the Pod should receive Service traffic.
- Probe mechanisms include HTTP, TCP, gRPC (where supported by the Kubernetes version), and exec commands.

### Engineering blueprint

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: http
  periodSeconds: 5
  failureThreshold: 30
readinessProbe:
  httpGet:
    path: /health/ready
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
livenessProbe:
  httpGet:
    path: /health/live
    port: http
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

The endpoint behavior must match its purpose. Readiness may check dependencies necessary to serve traffic; liveness should generally test whether the process can make progress, not whether every dependency is reachable.

### Graceful shutdown sequence

When a Pod is deleted (rollout, scale-down, node drain), two things start **in parallel**:

```text
1. kubelet: run preStop hook (if any) -> send SIGTERM -> wait up to terminationGracePeriodSeconds (default 30) -> SIGKILL
2. control plane: remove the Pod from Service EndpointSlices -> kube-proxy / load balancers stop routing to it
```

Because step 2 takes time to propagate, a Pod that exits immediately on `SIGTERM` can still receive new requests for a moment, and those requests fail. The usual fix:

```yaml
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: api
      lifecycle:
        preStop:
          sleep:
            seconds: 10        # keep serving while endpoints are removed (needs v1.30+;
                               # older clusters use exec: ["sleep", "10"], which requires sleep in the image)
```

Then the application, on `SIGTERM`, stops accepting new connections, finishes in-flight requests, closes DB connections, and exits. The preStop time plus the app's drain time must fit inside `terminationGracePeriodSeconds`.

### Production pitfalls

- A liveness probe that depends on a shared database can restart every replica during a database outage.
- Aggressive timeouts can cause restart loops during normal CPU pressure.
- A readiness endpoint that always returns success sends traffic to a broken application.
- A long startup threshold can mask a genuine boot failure.

### SDE best practices

- Separate startup, readiness, and liveness semantics.
- Set timing values based on measured startup and response times.
- Include graceful shutdown and connection draining behavior.
- Load-test probes during overload and dependency failure.

### Resources

- [Kubernetes: Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes: Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes: Container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

---

## 16. Requests, Limits, and Scheduling

Kubernetes schedules Pods based primarily on declared resource requests. Limits constrain resource use at runtime for supported resources. Incorrect values can cause unschedulable Pods, CPU throttling, or out-of-memory termination.

### Core mechanism

- CPU requests inform scheduling and relative CPU allocation; CPU limits commonly throttle usage.
- Memory requests inform scheduling; exceeding a memory limit can result in an OOM kill.
- A Pod's effective scheduling needs include container resources and applicable Pod-level overhead.
- Node allocatable capacity is less than raw node capacity because system components reserve resources.

### Engineering blueprint

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

These are examples only. Derive values from representative load tests, production metrics, startup peaks, and runtime behavior.

CPU and memory behave differently when a limit is reached:

- **CPU is compressible.** At the limit, the container is throttled (made to wait), so it gets slower but keeps running. `250m` means 0.25 of a core.
- **Memory is not compressible.** At the limit, the kernel kills the process (`OOMKilled`, exit code 137).

Common practice: set memory request equal to memory limit (predictable, no surprise kills from overcommit), and set a CPU request but consider omitting the CPU limit for latency-sensitive services so they can use idle CPU instead of being throttled. Some platform teams require CPU limits for fairness; check your cluster's policy.

The request/limit settings decide the Pod's **QoS class**, which determines who is evicted first when a node runs short of memory:

| QoS class | Condition | Eviction order |
| --- | --- | --- |
| `Guaranteed` | Every container has CPU and memory requests equal to limits | Last |
| `Burstable` | At least one request or limit set, but not Guaranteed | Middle |
| `BestEffort` | No requests or limits at all | First |

### Production pitfalls

- Requests that are too high waste capacity and leave Pods Pending.
- Requests that are too low lead to contention and unstable latency.
- Strict CPU limits can cause throttling and latency spikes.
- Memory limit breaches can abruptly terminate a process; the resulting restart may look like an application crash.
- Node selectors, affinity, taints, and topology constraints can prevent scheduling even when aggregate cluster capacity appears sufficient.

### SDE best practices

- Set requests for all production containers, including sidecars.
- Inspect container working set, throttling, restarts, and node pressure.
- Configure graceful handling for termination and OOM recovery.
- Use quotas and limit ranges to make namespace capacity predictable.

### Resources

- [Kubernetes: Manage resources for containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes: Assign memory resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Kubernetes: Assign CPU resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)
- [Kubernetes: Pod Quality of Service classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

---

## 17. Autoscaling and Capacity

Autoscaling adjusts workload replicas or cluster nodes based on observed or predicted demand. It does not create capacity instantly and depends on good metrics, resource requests, and application startup behavior.

### Core mechanism

- Horizontal Pod Autoscaler (HPA) changes replica count based on CPU, memory, or custom/external metrics.
- Vertical Pod Autoscaler (VPA), where installed and configured, recommends or adjusts resource requests.
- Cluster autoscalers add or remove nodes based on unschedulable Pods and utilization.
- Metrics Server provides basic resource metrics; custom metrics require additional adapters and monitoring integrations.

### Engineering blueprint

- Set realistic requests; CPU-based HPA uses them to interpret utilization.
- Define minimum and maximum replicas and scale-up/scale-down behavior.
- Ensure the cluster can obtain node capacity and that quotas/cloud limits permit it.
- Account for cold start, image-pull time, cache warming, and downstream capacity.
- Load-test scaling lag and metric failure behavior.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70      # percent of the CPU *request*, not of the node
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # wait 5 minutes of lower load before removing Pods
```

The HPA's core formula is `desired = ceil(current_replicas × current_metric / target_metric)`. With 4 replicas at 105% CPU and a 70% target: `ceil(4 × 105 / 70) = 6` replicas.

For queue workers, CPU is often the wrong signal; scale on queue length or message age instead, for example with **KEDA**, which feeds external metrics (Kafka lag, SQS depth, Prometheus queries) to the HPA and can scale to zero.

### Production pitfalls

- HPA cannot scale from CPU utilization meaningfully when requests are missing or unrealistic.
- Scaling can overload a fixed-capacity database or external API.
- Fast scale-down may terminate in-flight requests or erase warmed local caches.
- Metric outages or noisy metrics can cause oscillation or failed scaling.
- Cluster autoscaling may not help if Pods cannot fit due to affinity, zones, or volume constraints.

### SDE best practices

- Treat autoscaling as one part of a capacity plan, not a substitute for load testing.
- Add stabilization windows and behavior policies to avoid flapping.
- Scale on a metric that correlates with saturation and user impact.
- Reserve downstream capacity and enforce per-replica connection/concurrency limits.

### Resources

- [Kubernetes: Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes: HPA concepts](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Kubernetes: Node autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [KEDA: Kubernetes Event-driven Autoscaling](https://keda.sh/)

---

## 18. Persistent Volumes and Stateful Workloads

Stateful applications need identity, storage, and recovery semantics that survive Pod replacement. Kubernetes provides PersistentVolumes, PersistentVolumeClaims, StorageClasses, and StatefulSets, but does not automatically make an application database-safe.

### Core mechanism

- A `StorageClass` describes dynamic provisioning behavior.
- A `PersistentVolumeClaim` requests storage with capacity and access requirements.
- A `PersistentVolume` represents provisioned storage.
- A `StatefulSet` provides stable Pod identity and ordered operations, often paired with per-Pod claims.
- A headless Service can provide stable network identities for StatefulSet members.

### Engineering blueprint

Use StatefulSets only when the application benefits from stable identity or per-replica storage. Validate the storage provider's zone, attach, expansion, snapshot, and failure semantics. For databases, define backup, replication, upgrade, and recovery procedures at the database layer.

### Production pitfalls

- `ReadWriteOnce` generally describes mounting semantics, not application-level single-writer safety under every failure.
- A volume stuck in another zone can prevent a rescheduled Pod from starting.
- Deleting a StatefulSet does not always delete associated claims; deleting a claim may delete backing storage depending on policy.
- Recreating a Pod does not repair corrupt application data.

### SDE best practices

- Prefer managed stateful services when operational maturity or uptime requirements justify them.
- Test node loss, zone loss, volume detach/reattach, restore, and upgrade paths.
- Understand finalizers, retention policy, reclaim policy, and snapshot consistency.
- Avoid treating a volume snapshot as a valid application-consistent backup unless coordinated with the workload.

### Resources

- [Kubernetes: StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes: Volume snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Kubernetes: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

---

## 19. Security: Images, Containers, and Clusters

Container security is a lifecycle practice spanning source, build, artifact, deployment, runtime, and cluster access. Containers share a kernel, so least privilege and defense in depth matter.

### Core mechanism

- Image provenance and vulnerability scanning help evaluate what is deployed.
- Runtime settings control user identity, Linux capabilities, privilege escalation, mounts, and seccomp.
- Kubernetes RBAC controls API permissions; service accounts provide workload identities.
- Network policies restrict traffic when implemented by the cluster networking layer.
- Admission policies can validate or constrain workloads before they are stored.

### Engineering blueprint

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

Some applications need writable temporary directories or specific capabilities; grant only the required access. Pod- and container-level security settings interact, so validate against cluster policy and application behavior.

By default, every Pod can talk to every other Pod in the cluster. Start each namespace with a default-deny policy, then allow only the required paths:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}            # all Pods in this namespace
  policyTypes: ["Ingress"]   # no ingress rules listed = deny all inbound
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 5432
```

If you also deny egress, remember to allow DNS (UDP/TCP 53 to the cluster DNS Pods), or every name lookup will fail.

Least-privilege RBAC for a workload that only needs to read ConfigMaps in its namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-config
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-read-config
subjects:
  - kind: ServiceAccount
    name: api
roleRef:
  kind: Role
  name: read-config
  apiGroup: rbac.authorization.k8s.io
```

Set `automountServiceAccountToken: false` on Pods that do not call the Kubernetes API at all.

### Production pitfalls

- `privileged: true`, host networking, host PID, and broad hostPath mounts weaken isolation significantly.
- A non-root image can still gain dangerous access through capabilities or mounted credentials.
- Cluster-admin kubeconfigs and wildcard RBAC make compromise high impact.
- Vulnerability scanning is not proof of safety and can generate noise or miss runtime risks.
- Image signatures and provenance only help if verification is enforced at deployment.

### SDE best practices

- Build from trusted, maintained base images; scan and rebuild regularly.
- Run as non-root, drop capabilities, use read-only filesystems where practical, and set seccomp profiles.
- Use least-privilege RBAC and dedicated service accounts per workload.
- Restrict egress and ingress to required paths; protect the control-plane API.
- Use admission policy and verify image provenance/signatures where available.
- Keep Kubernetes, nodes, runtimes, and dependencies patched.

### Resources

- [Kubernetes: Security checklist](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [Kubernetes: Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes: RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Docker: Engine security](https://docs.docker.com/engine/security/)
- [NIST SP 800-190: Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)

---

## 20. Observability and Troubleshooting

Kubernetes has several layers of state and failure. Diagnose from the workload outward: desired object, scheduling, container state, probes, Service endpoints, network, and dependency health.

### Core mechanism

- **Logs** explain process events and errors.
- **Metrics** describe resource use, saturation, traffic, and error rates.
- **Traces** connect work across service boundaries.
- **Events** report Kubernetes scheduling and lifecycle decisions; they are typically retained for a limited time.
- `kubectl` inspects API resources but cannot replace application-level observability.

### Engineering blueprint

Useful initial commands:

```bash
kubectl get deploy,pods,svc,endpointslices -n my-namespace
kubectl describe pod my-pod -n my-namespace
kubectl logs deploy/my-api -n my-namespace --since=15m
kubectl logs my-pod -n my-namespace --previous
kubectl get events -n my-namespace --sort-by=.lastTimestamp
kubectl rollout status deployment/my-api -n my-namespace
```

For a stuck Pod, check status, events, node capacity, image pull errors, volume attach state, and probe failures before changing the manifest.

Common Pod states and what to check:

| Status / reason | Usual meaning | First things to check |
| --- | --- | --- |
| `Pending` | Not scheduled yet | `kubectl describe pod` events: insufficient CPU/memory, unmatched node selector or affinity, taints, unbound PVC, volume in another zone |
| `ImagePullBackOff` / `ErrImagePull` | Node cannot pull the image | Typo in image name or tag, missing `imagePullSecrets`, registry auth or network, wrong architecture |
| `CreateContainerConfigError` | Container spec references something missing | Missing ConfigMap, Secret, or key referenced by `env` / `envFrom` |
| `CrashLoopBackOff` | Container starts and keeps exiting; restarts back off exponentially | `kubectl logs POD --previous`, exit code in `describe`, missing config, failing liveness probe, app crash on start |
| `OOMKilled` (exit 137) | Exceeded its memory limit | Memory limit vs actual usage, leaks, JVM/Node heap settings that ignore the container limit |
| `Running` but `0/1 READY` | Readiness probe failing | Probe path and port, dependency the readiness check needs, startup time |
| `Terminating` for a long time | Shutdown or cleanup blocked | App ignoring SIGTERM, long grace period, finalizers on the object, unreachable node |
| `Evicted` | Node ran short of memory or disk | Node pressure conditions, Pod QoS class, ephemeral storage use (logs, `/tmp`) |

Debugging minimal images that have no shell:

```bash
kubectl debug -it POD --image=busybox:1.37 --target=api     # ephemeral container sharing the app's process namespace
kubectl debug node/NODE -it --image=busybox:1.37            # shell on a node (host filesystem mounted at /host)
kubectl port-forward svc/api 8080:80                        # call a ClusterIP Service from your laptop
kubectl run tmp --rm -it --image=busybox:1.37 -- nslookup api.my-namespace.svc   # test DNS from inside the cluster
```

### Production pitfalls

- Logs may disappear with a Pod unless shipped off-node.
- High-cardinality labels (user IDs, request IDs) can make metrics expensive or unusable.
- `kubectl exec` into a running Pod can hide an immutable-image defect if the fix is not committed to deployment configuration.
- Looking only at Pod `Running` misses readiness, application errors, and Service endpoint problems.

### SDE best practices

- Emit structured logs with request/correlation IDs while redacting secrets and personal data.
- Monitor service-level indicators (availability, latency, errors, saturation) and deployment health.
- Alert on user impact and sustained saturation, not every transient event.
- Capture traces across service calls and attach deploy/image version metadata.
- Use ephemeral debug containers or controlled tooling where production images are intentionally minimal.

### Resources

- [Kubernetes: Debugging applications](https://kubernetes.io/docs/tasks/debug/)
- [Kubernetes: Debug running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes: Logging architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Kubernetes: Monitoring](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)

---

## 21. Production Checklist and Cheat Sheet

### Image and build checklist

- [ ] Build from a reviewed Dockerfile and clean, minimal build context.
- [ ] `.dockerignore` excludes secrets, local dependencies, and unrelated files.
- [ ] Final image contains only runtime artifacts and dependencies.
- [ ] Image runs as a non-root user where possible.
- [ ] Base image and dependencies are maintained, scanned, and rebuilt for fixes.
- [ ] Production deploy uses an immutable digest and records source/build provenance.
- [ ] Application handles `SIGTERM`, uses stdout/stderr logging, and exits with meaningful status.
- [ ] No credentials are baked into image layers or build arguments.

### Kubernetes workload checklist

- [ ] A controller manages production Pods (typically a Deployment for stateless services).
- [ ] Labels and selectors are consistent; Service has the expected endpoints.
- [ ] Replicas, rollout strategy, and rollback are documented and tested.
- [ ] Startup/readiness/liveness probes reflect distinct health semantics.
- [ ] CPU and memory requests are set and validated under representative load.
- [ ] Container security context is least-privileged; no unnecessary host access.
- [ ] Configuration is external to the image; secret access is narrowly scoped.
- [ ] Network exposure and ingress/gateway routes are explicitly reviewed.
- [ ] Persistent data has tested backup, restore, retention, and failure procedures.
- [ ] Pod disruption, node drain, and zone failure scenarios are considered.

### Operations checklist

- [ ] Deployment pipeline promotes the same image digest through environments.
- [ ] CI validates manifests and policy before deployment.
- [ ] Monitoring covers latency, errors, saturation, restarts, rollout health, and capacity.
- [ ] Logs and traces are retained outside ephemeral Pod storage.
- [ ] Alerts have runbooks and actionable ownership.
- [ ] Autoscaling is tested with realistic metrics and downstream capacity.
- [ ] Registry, DNS, certificates, storage, and control-plane dependencies are monitored.
- [ ] Rollback and disaster recovery have been practiced.

### Common `kubectl` commands

| Need | Command |
| --- | --- |
| List resources | `kubectl get all -n NAMESPACE` |
| Describe a Pod | `kubectl describe pod POD -n NAMESPACE` |
| Read logs | `kubectl logs POD -n NAMESPACE` |
| Read prior container logs | `kubectl logs POD -n NAMESPACE --previous` |
| Check events | `kubectl get events -n NAMESPACE --sort-by=.lastTimestamp` |
| Watch rollout | `kubectl rollout status deployment/NAME -n NAMESPACE` |
| Undo rollout | `kubectl rollout undo deployment/NAME -n NAMESPACE` |
| Apply manifests | `kubectl apply -f DIRECTORY/` |
| Inspect config | `kubectl get configmap,secret -n NAMESPACE` |
| Debug service endpoints | `kubectl get endpointslices -n NAMESPACE` |

### Quick glossary

| Term | One line |
| --- | --- |
| Image | Immutable template from which containers are created |
| Container | Isolated process running from an image |
| Registry | Service that stores and distributes images |
| Digest | Content-derived identifier for exact image data |
| Build context | Files available to a Docker image builder |
| Pod | Kubernetes unit that groups one or more co-located containers |
| Deployment | Controller for stateless replicated Pods and rollouts |
| ReplicaSet | Controller maintaining a desired count of matching Pods |
| Service | Stable network endpoint for a changing set of Pods |
| Ingress | HTTP routing API requiring an implementing controller |
| Gateway API | Extensible Kubernetes API for traffic routing |
| ConfigMap | Kubernetes object for non-confidential configuration |
| Secret | Kubernetes object intended for confidential configuration; needs protection |
| Probe | Health check that affects startup, restart, or traffic eligibility |
| Request / limit | Scheduling reservation / runtime resource ceiling |
| CNI / CSI | Container Network Interface / Container Storage Interface |
| Reconciliation | Repeatedly driving observed state toward declared desired state |

---

## 22. Self-Check Questions

Try to answer each question before opening the answer.

<details>
<summary>1. How is a container different from a virtual machine?</summary>

A container is an isolated process that shares the host's kernel; namespaces limit what it can see and cgroups limit what it can use. A VM runs its own kernel on virtualized hardware. Containers start faster and are lighter, but the kernel is a shared attack surface, so isolation is weaker.
</details>

<details>
<summary>2. Why should you deploy by image digest rather than by tag?</summary>

Tags are mutable: `api:1.4` or `latest` can be repointed to different content, so two nodes can run different code under the same tag. A digest identifies exact bytes, so every environment runs what was tested and a rollback returns to exactly the previous image.
</details>

<details>
<summary>3. Your container ignores docker stop and is killed after 10 seconds. What is the likely cause?</summary>

The process is not receiving or handling SIGTERM. Usually the Dockerfile uses shell-form `CMD`, so `/bin/sh` is PID 1 and does not forward signals; or the app has no SIGTERM handler. Use exec-form `CMD ["node", "server.js"]` (or an init like `tini`) and handle SIGTERM in the app.
</details>

<details>
<summary>4. What is the difference between liveness and readiness probes? What happens if a liveness probe checks the database?</summary>

Readiness failing removes the Pod from Service endpoints but does not restart it. Liveness failing restarts the container. If liveness checks a shared database, a database outage fails liveness on every replica, so Kubernetes restarts all of them at once and makes the outage worse.
</details>

<details>
<summary>5. A Pod is stuck in Pending. Walk through how you diagnose it.</summary>

`kubectl describe pod` and read the events. Common causes: no node has enough unreserved CPU or memory for the requests, node selectors or affinity match no node, taints without tolerations, a PVC that cannot bind or whose volume is in another zone, or a namespace quota exceeded.
</details>

<details>
<summary>6. What does CrashLoopBackOff mean and what is the first command you run?</summary>

The container keeps exiting and Kubernetes waits longer between each restart. Run `kubectl logs POD --previous` to see the output of the crashed container, then `kubectl describe pod` for the exit code and reason (for example, 137 / OOMKilled).
</details>

<details>
<summary>7. What happens when a container hits its CPU limit versus its memory limit?</summary>

At the CPU limit it is throttled and slows down. At the memory limit it is killed (OOMKilled, exit code 137).
</details>

<details>
<summary>8. During a rolling update, some requests fail with connection errors even though readiness probes are configured. Why?</summary>

When a Pod is deleted, SIGTERM and endpoint removal happen in parallel. If the app exits immediately, load balancers and kube-proxy may still route requests to it for a short time. Add a short `preStop` sleep and make the app drain in-flight requests before exiting, within `terminationGracePeriodSeconds`.
</details>

<details>
<summary>9. Is a Kubernetes Secret encrypted?</summary>

Not by default. Secret values are base64-encoded, which is encoding, not encryption. Protection comes from enabling encryption at rest for etcd, restricting access with RBAC, and often syncing values from an external secret manager.
</details>

<details>
<summary>10. What is the difference between port, targetPort, and containerPort?</summary>

`port` is the port the Service listens on. `targetPort` is the port on the Pod that the Service forwards to. `containerPort` documents the port the container listens on; it does not open or expose anything by itself. `targetPort` must match the port the app actually listens on.
</details>

<details>
<summary>11. When would you use a StatefulSet instead of a Deployment?</summary>

When each replica needs a stable identity (`db-0`, `db-1`), its own persistent volume that follows it across rescheduling, or ordered startup and shutdown, as databases and Kafka do. Stateless services should use Deployments.
</details>

<details>
<summary>12. Three replicas are running, but a single node drain caused downtime. What was missing?</summary>

The replicas were probably on the same node, and nothing limited voluntary disruptions. Add `topologySpreadConstraints` or pod anti-affinity to spread replicas across nodes and zones, and a `PodDisruptionBudget` so a drain keeps a minimum number running.
</details>

## 23. Master Resource List

### Docker and OCI

| Topic | Link |
| --- | --- |
| Docker Get Started | [Docker Docs](https://docs.docker.com/get-started/) |
| Dockerfile reference | [Docker Docs](https://docs.docker.com/reference/dockerfile/) |
| Build best practices | [Docker Docs](https://docs.docker.com/build/building/best-practices/) |
| Build cache | [Docker Docs](https://docs.docker.com/build/cache/) |
| Docker Compose | [Docker Docs](https://docs.docker.com/compose/) |
| Docker networking | [Docker Docs](https://docs.docker.com/engine/network/) |
| Docker storage | [Docker Docs](https://docs.docker.com/engine/storage/) |
| OCI Image Specification | [Open Container Initiative](https://github.com/opencontainers/image-spec) |
| OCI Runtime Specification | [Open Container Initiative](https://github.com/opencontainers/runtime-spec) |

### Kubernetes foundations

| Topic | Link |
| --- | --- |
| Concepts overview | [Kubernetes Docs](https://kubernetes.io/docs/concepts/) |
| Cluster architecture | [Kubernetes Docs](https://kubernetes.io/docs/concepts/architecture/) |
| Workloads overview | [Kubernetes Docs](https://kubernetes.io/docs/concepts/workloads/) |
| Services and networking | [Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/) |
| Configuration | [Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/) |
| Storage | [Kubernetes Docs](https://kubernetes.io/docs/concepts/storage/) |
| Scheduling and resource management | [Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) |
| Kubernetes tutorials | [Kubernetes Docs](https://kubernetes.io/docs/tutorials/) |

### Security and operations

| Topic | Link |
| --- | --- |
| Kubernetes Security Checklist | [Kubernetes Docs](https://kubernetes.io/docs/concepts/security/security-checklist/) |
| Pod Security Standards | [Kubernetes Docs](https://kubernetes.io/docs/concepts/security/pod-security-standards/) |
| RBAC authorization | [Kubernetes Docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) |
| Debugging applications | [Kubernetes Docs](https://kubernetes.io/docs/tasks/debug/) |
| Logging architecture | [Kubernetes Docs](https://kubernetes.io/docs/concepts/cluster-administration/logging/) |
| Application Container Security Guide | [NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final) |
| Kubernetes release and version skew policy | [Kubernetes Docs](https://kubernetes.io/releases/version-skew-policy/) |
| OpenTelemetry | [Documentation](https://opentelemetry.io/docs/) |

### Learning paths and tools

| Resource | Link |
| --- | --- |
| Kubernetes interactive tutorials | [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/) |
| Local Kubernetes clusters | [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/docs/) |
| Kubernetes package manager | [Helm](https://helm.sh/docs/) |
| Kubernetes configuration management | [Kustomize](https://kustomize.io/) |
| Certifications (hands-on exams) | [CKAD (application developer)](https://www.cncf.io/training/certification/ckad/), [CKA (administrator)](https://www.cncf.io/training/certification/cka/) |
| kubectl quick reference | [Kubernetes Docs](https://kubernetes.io/docs/reference/kubectl/quick-reference/) |
