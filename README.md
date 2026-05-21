# Advanced Programming — Module 11: Deployment on Kubernetes

This repository contains my work and reflection notes for the Kubernetes deployment
tutorial (Hello Minikube + Rolling Update Deployment).

Environment used:
- **OS:** Windows 11 Home Single Language (25H2)
- **Driver:** Docker (Docker Desktop) — minikube `docker` driver
- **minikube:** v1.38.1
- **Kubernetes:** v1.35.1
- **kubectl:** v1.34.1
- **Apps deployed:** `agnhost` (hello-node) and `springcommunity/spring-petclinic-rest`

The Kubernetes manifest files for the Spring Petclinic REST app live next to this README:
- [`deployment.yaml`](deployment.yaml) — Deployment using the **RollingUpdate** strategy
- [`service.yaml`](service.yaml) — LoadBalancer Service exposing port `9966`
- [`deployment-recreate.yaml`](deployment-recreate.yaml) — Deployment using the **Recreate** strategy
  (reuses the same [`service.yaml`](service.yaml))

---

## Part 1 — Hello Minikube

I started a local cluster with `minikube start --driver=docker`, then deployed the
`agnhost` sample image as a Deployment named `hello-node`, exposed it as a
`LoadBalancer` Service on port `8080`, and enabled the `metrics-server` addon to use
`kubectl top`.

> Note: On Windows + Git Bash, the literal argument `/agnhost` gets mangled by MSYS path
> translation into `C:/Program Files/Git/agnhost`, which makes the container crash with
> `RunContainerError`. Running the `kubectl create deployment ... -- /agnhost netexec
> --http-port=8080` command from **PowerShell** (or prefixing with `MSYS_NO_PATHCONV=1`)
> avoids this and the Pod starts cleanly.

`kubectl top` output after the metrics-server became operational:

```
NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
minikube   1206m        6%       1033Mi          7%

NAME                                    CPU(cores)   MEMORY(bytes)
hello-node-585ccb7764-vfjjv             1m           6Mi
spring-petclinic-rest-69cbbd78b-bf2xn   1533m        200Mi
```

### Reflection on Hello Minikube

**1. Compare the application logs before and after you exposed it as a Service. Does the
number of logs increase each time you open the app?**

Yes — the number of log lines increases by exactly one HTTP request entry every time the
app is accessed. Right after the Deployment was created (and before any client hit it),
`kubectl logs` only showed the two start-up lines:

```
I... log.go:195] Started HTTP server on port 8080
I... log.go:195] Started UDP server on port  8081
```

After exposing the Pod as a Service and sending 5 requests to it (through a
`kubectl port-forward` tunnel, which behaves like `minikube service`), the log grew from
**2 lines to 7 lines**, one new `GET /` entry per request:

```
I... log.go:195] Started HTTP server on port 8080
I... log.go:195] Started UDP server on port  8081
I... log.go:195] GET /
I... log.go:195] GET /
I... log.go:195] GET /
I... log.go:195] GET /
I... log.go:195] GET /
```

This happens because the Service does not run any application logic of its own — it is
only a stable network endpoint (a virtual IP plus kube-proxy rules) that forwards each
incoming connection to the Pod. The container is what actually handles the HTTP request
and writes a log line, so each browser refresh / `curl` reaches the same container and
adds one more `GET /` line. The Service simply makes that container reachable from outside
the cluster; it does not multiply, cache, or absorb the traffic.

**2. There are two `kubectl get` invocations: one with no option, one with `-n
kube-system`. What is the purpose of `-n`, and why did the first command not list the
pods/services you created?**

The `-n` (`--namespace`) flag tells `kubectl` which **Namespace** to query. Namespaces are
virtual partitions inside a single physical cluster; they isolate and group resources so
that different teams or system components do not collide. When you do not pass `-n`,
`kubectl` operates on the namespace configured as the default for the current context,
which (out of the box) is literally named `default`. The resources I created — `hello-node`
and `spring-petclinic-rest` — were placed in the `default` namespace, so a plain
`kubectl get pods` showed them. The Kubernetes control-plane and add-on components
(CoreDNS, etcd, kube-apiserver, kube-proxy, the metrics-server, etc.) instead live in the
`kube-system` namespace, so they are invisible to a plain `kubectl get` and only appear
when you explicitly scope the query with `-n kube-system`. In short, the two commands
returned different things because they were looking in two different namespaces; resources
are namespaced and queries only see the namespace they target (unless you pass
`--all-namespaces` / `-A`).

---

## Part 2 — Rolling Update Deployment

I deployed `spring-petclinic-rest:3.0.2`, exposed it on port `9966`, confirmed the REST API
works (`GET /petclinic/api/pettypes` returned the seeded pet types with HTTP 200), scaled
it to **4 replicas**, then performed a rolling update to `3.2.1`, deliberately broke it with
a non-existent `4.0` tag, and rolled back.

Evidence collected during the run:

- **Scaling** `kubectl scale --replicas=4` produced 4 running Pods (verified with
  `kubectl get pods -o wide`).
- **Rolling update to `3.2.1`** progressed incrementally — `kubectl rollout status` showed
  new replicas coming up a few at a time while old ones were terminated, and the cluster
  always kept a minimum number of available Pods (no downtime). Afterwards:
  ```
  OldReplicaSet: spring-petclinic-rest-69cbbd78b (0/0 replicas)
  NewReplicaSet: spring-petclinic-rest-695db76b69 (4/4 replicas)
  Image:         docker.io/springcommunity/spring-petclinic-rest:3.2.1
  ```
- **Bad image `4.0`** caused the new Pods to get stuck in `ErrImagePull` /
  `ImagePullBackOff`, with the event:
  ```
  Failed to pull image "...spring-petclinic-rest:4.0":
  manifest for ...spring-petclinic-rest:4.0 not found: manifest unknown
  ```
  Crucially, the **old `3.2.1` Pods kept running and serving traffic** the whole time, so
  the broken rollout never took the app down.
- **`kubectl rollout undo`** reverted the Deployment back to the last working `3.2.1`
  image and all 4 Pods returned to `Running`.

### Exporting and re-applying the manifests

Following the tutorial I exported the live configuration and then rebuilt the app purely
from YAML. The committed [`deployment.yaml`](deployment.yaml) and
[`service.yaml`](service.yaml) are cleaned-up versions (runtime-only fields such as
`status`, `resourceVersion`, `uid`, and `clusterIP` removed) so they apply immediately to
a blank Minikube cluster. I verified this by deleting the imperatively-created resources and
running:

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

The Deployment rolled out to `4/4` ready and the Service was recreated on port `9966` —
no manual edits required.

### Reflection on Rolling Update & Kubernetes Manifest File

**1. What is the difference between Rolling Update and Recreate deployment strategy?**

The two strategies differ in *how* old Pods are replaced by new ones during an update.
With **RollingUpdate** (the default), Kubernetes replaces Pods **incrementally**: it spins
up some new Pods, waits for them to become ready, then terminates an equal slice of old
Pods, repeating until the whole ReplicaSet is updated. It is bounded by `maxSurge` (how many
extra Pods may exist above the desired count) and `maxUnavailable` (how many may be missing
below it), so a minimum number of Pods is always available — the app stays up with **zero
downtime**, at the cost of briefly running two versions side by side. With **Recreate**,
Kubernetes first **terminates all** old Pods and only **then** creates the new ones. There
is a window during which no Pod is running, so the app experiences **downtime**, but only a
single version is ever live at a time. RollingUpdate favors availability and is right for
stateless web services; Recreate favors a clean cutover and is needed when two versions
cannot run simultaneously (e.g. an incompatible database schema migration, or an exclusive
lock / single-writer resource).

**2. Try deploying the Spring Petclinic REST using Recreate deployment strategy and
document your attempt.**

I switched the Deployment to the Recreate strategy by applying
[`deployment-recreate.yaml`](deployment-recreate.yaml) (`strategy.type: Recreate`) and then
triggered an update by changing the image tag. I sampled the Pod states every ~3 seconds
during the rollout. The observed behavior matched the theory exactly: **all four old Pods
moved to `Terminating` at the same time**, and the cluster waited until they were gone
before any replacement Pod appeared:

```
t1 .. t10:  4 Pods Terminating          (old version going down — NO Running Pods)
t11:        old Pods Error/gone, new Pods ContainerCreating
t12:        4 new Pods ContainerCreating -> Running
```

In other words there was a clear gap (roughly the Pod termination + new container start
time) where the application had **zero available Pods** — i.e. real downtime. This is the
defining contrast with the earlier RollingUpdate, where at least 3 of 4 Pods stayed
available throughout. After documenting this, I re-applied [`deployment.yaml`](deployment.yaml)
to return the cluster to the zero-downtime RollingUpdate configuration on image `3.2.1`.

**3. Prepare different manifest files for executing Recreate deployment strategy.**

Done — see [`deployment-recreate.yaml`](deployment-recreate.yaml). It is identical to
[`deployment.yaml`](deployment.yaml) except the `strategy` block is replaced with:

```yaml
strategy:
  type: Recreate
```

(Note that `Recreate` does not take a `rollingUpdate:` sub-block — `maxSurge`/
`maxUnavailable` only apply to the RollingUpdate strategy.) The Service manifest is
unchanged, so [`service.yaml`](service.yaml) is reused for both strategies.

**4. What are the benefits of using Kubernetes manifest files compared to deploying
manually (`kubectl create`/`scale`/`set image`)?**

Deploying with manifest files is **declarative** rather than **imperative**, and that brings
several concrete benefits. First, the manifest is the single source of truth: it captures the
entire desired state (image, replica count, strategy, ports, labels) in one place, instead of
that state being spread across a sequence of one-off `kubectl` commands that nobody can
reconstruct later. Second, it is **reproducible and portable** — I tore the app down and
brought it back identically on a fresh state with two `kubectl apply -f` commands, and a TA
can do the same on their own machine with no guesswork. Third, the files are **version-
controllable**: they live in Git next to the code, so every change to the deployment is
reviewable, diffable, and revertable, and the deployment history is auditable. Fourth,
`kubectl apply` is **idempotent and convergent** — running it repeatedly is safe and only
changes what differs from the cluster, whereas imperative commands fail or behave
differently depending on whether the resource already exists. Finally, manifests scale to
**multiple resources and environments** (you can apply a whole directory, template them, or
feed them to GitOps/CI-CD pipelines), which is impractical to do reliably by typing commands
by hand. The manual approach is fine for quick experiments, but the manifest approach is what
makes deployments repeatable, collaborative, and automatable.

**5. (Optional) Managed Kubernetes cluster (e.g. GCP/GKE).**

Not performed in this submission. The tutorial steps above were all done on a local Minikube
cluster. The expected difference on a managed cluster (GKE/EKS/AKS) is mainly that a
`LoadBalancer` Service would receive a **real external IP** from the cloud provider's load
balancer instead of staying `<pending>` and requiring a `minikube tunnel`/`minikube service`
proxy, and that nodes/scaling/networking are managed by the provider rather than by a single
local Docker container.

---

## How to reproduce

```bash
# 1. Start the cluster
minikube start --driver=docker

# 2. Hello Minikube  (run the create from PowerShell on Windows to avoid path mangling)
kubectl create deployment hello-node \
  --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
minikube addons enable metrics-server
kubectl top nodes && kubectl top pods

# 3. Spring Petclinic REST + rolling update (or just apply the manifests below)
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
minikube service spring-petclinic-rest        # opens the app in a browser (keep shell open)

# 4. Try the Recreate strategy
kubectl apply -f deployment-recreate.yaml
```
