# Kubernetes Learning Plan: Local-First

A hands-on path from "what is a pod?" to deploying, testing, and debugging a small multi-tier app on your own Mac. Every stage has a **lesson**, **labs you can copy-paste**, **checkpoint questions**, and **challenges**.

**Total time:** roughly 4–6 weeks at an hour or two a day. Don't rush Stages 2–4. Most of the learning happens there.

---

## Table of contents

- [How to use this plan](#how-to-use-this-plan)
- [Stage 0: Container prerequisites](#stage-0-container-prerequisites)
- [Stage 1: The mental model](#stage-1-the-mental-model)
- [Stage 2: A local cluster and kubectl](#stage-2-a-local-cluster-and-kubectl)
- [Stage 3: Deploy your own app](#stage-3-deploy-your-own-app)
- [Stage 4: Configuration, health, and state](#stage-4-configuration-health-and-state)
- [Stage 5: Networking and a multi-tier app](#stage-5-networking-and-a-multi-tier-app)
- [Stage 6: Developer workflow, packaging, and testing](#stage-6-developer-workflow-packaging-and-testing)
- [Stage 7: Beyond the basics](#stage-7-beyond-the-basics)
- [Appendix A: kubectl cheat sheet](#appendix-a-kubectl-cheat-sheet)
- [Appendix B: Troubleshooting table](#appendix-b-troubleshooting-table)
- [Appendix C: Cleanup](#appendix-c-cleanup)

---

## How to use this plan

### Conventions

- Cluster name: **`learn`** (so the kubectl context is `kind-learn`).
- Everything runs in the `default` namespace unless a lab says otherwise.
- Pin image tags (`myapp:v1`, `nginx:1.27`). Avoid `:latest`, because Kubernetes defaults to pulling `:latest` images from a registry, which breaks locally built images.
- Files are saved under a `labs/` folder next to this document:

```
kubernetes-intro/
├── k8s-learning-plan.md
└── labs/
    ├── app/                  # Stage 0: the sample app + Dockerfile
    ├── kind-config.yaml      # Stage 2
    ├── k8s/                  # Stages 3–5 manifests
    ├── kustomize/            # Stage 6
    ├── chart/                # Stage 6 (Helm)
    └── scripts/              # Stage 6 smoke tests
```

Create it once:

```bash
mkdir -p labs/{app,k8s,kustomize,chart,scripts}
```

### How to learn from each stage

1. Read the lesson.
2. Run the labs. **Type commands rather than only pasting them** for the first few stages.
3. After each lab, ask "what did Kubernetes just do?" and confirm with `kubectl get events` or `kubectl describe`.
4. Answer the checkpoint questions without looking.
5. Try at least one challenge.

---

## Stage 0: Container prerequisites

**Time:** half a day  **Goal:** build and run a container image locally, and create the sample app used for the rest of the plan.

### Lesson

Kubernetes orchestrates **containers**. You need the following.

| Concept | Meaning |
|---|---|
| **Image** | A read-only, layered package: filesystem + metadata + default command. |
| **Container** | A running instance of an image (an isolated process with its own filesystem and network). |
| **Registry** | A server that stores images (Docker Hub, GHCR, ECR). Kubernetes pulls images from registries. |
| **Tag** | A label on an image version (`myapp:v1`). `latest` is just a tag name, not "the newest". |
| **Port mapping** | `-p 8080:8080` maps host port → container port. |
| **Environment variables** | The main way to configure a containerized app, and what ConfigMaps and Secrets feed into later. |

A good Kubernetes workload is a **12-factor style** process. It logs to stdout/stderr, takes config from env vars, listens on a port, and is safe to kill and restart at any time. The sample app follows this.

### Install tools

```bash
# Container runtime: install one
#   Docker Desktop: https://www.docker.com/products/docker-desktop/
#   or OrbStack (lighter on Mac): brew install --cask orbstack

brew install kubectl kind helm k9s kubeconform
# Later stages: brew install tilt-dev/tap/tilt   (Stage 6)

docker version            # client AND server should respond
kubectl version --client
kind version
```

### Lab 0.1: The sample app

A tiny HTTP service using only the Python standard library, so there are no dependencies to manage.

**`labs/app/app.py`**

```python
import json
import os
import socket
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer


class Handler(BaseHTTPRequestHandler):
    def _send(self, code, body, content_type="application/json"):
        self.send_response(code)
        self.send_header("Content-Type", content_type)
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        if self.path == "/healthz":
            self._send(200, b"ok", "text/plain")
        elif self.path == "/":
            payload = {
                "message": os.getenv("GREETING", "Hello"),
                "hostname": socket.gethostname(),  # the pod name in Kubernetes
                "version": os.getenv("APP_VERSION", "dev"),
            }
            self._send(200, json.dumps(payload).encode())
        else:
            self._send(404, b'{"error":"not found"}')


if __name__ == "__main__":
    port = int(os.getenv("PORT", "8080"))
    print(f"listening on :{port}", flush=True)
    ThreadingHTTPServer(("0.0.0.0", port), Handler).serve_forever()
```

**`labs/app/Dockerfile`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .

# Bake a version into the image so we can tell v1 from v2 later
ARG APP_VERSION=v1
ENV APP_VERSION=$APP_VERSION
ENV PYTHONUNBUFFERED=1

EXPOSE 8080
USER 1000
CMD ["python", "app.py"]
```

Build and run:

```bash
docker build -t myapp:v1 labs/app
docker run --rm -p 8080:8080 -e GREETING="Hi from Docker" myapp:v1
# In another terminal:
curl localhost:8080/
curl localhost:8080/healthz
```

You should see JSON with a `hostname` (the container ID), the greeting, and `"version": "v1"`.

### Lab 0.2: Build a v2 image

```bash
docker build --build-arg APP_VERSION=v2 -t myapp:v2 labs/app
docker run --rm -p 8080:8080 myapp:v2 &
curl localhost:8080/        # version: v2
kill %1
```

You'll use v1 and v2 in Stage 3 for rolling updates.

### Checkpoint

1. What is the difference between an image and a container?
2. Why does the app log to stdout instead of a file?
3. Why would `FROM python` with no tag be a bad idea in production?

<details><summary>Answers</summary>

1. An image is the packaged, read-only template. A container is a running process created from it.
2. Container runtimes and Kubernetes capture stdout/stderr (`kubectl logs`). A file inside the container is lost when it restarts.
3. The tag floats, so builds aren't reproducible. Pin a version (`python:3.12-slim`), or better, a digest.
</details>

### Challenge

Add a `/slow?seconds=N` endpoint that sleeps. It's useful later for watching rolling updates and graceful shutdown.

---

## Stage 1: The mental model

**Time:** 1–2 days (mostly reading)  **Goal:** understand what Kubernetes is, its parts, and its core idea, **desired state and reconciliation**.

### Lesson

#### What problem does Kubernetes solve?

Running one container is easy. Running dozens across several machines raises questions. Where does each one run? What if one crashes? How do I update without downtime? How do they find each other? How do I scale up? Kubernetes is a **container orchestrator** that answers those.

#### The big idea: declarative desired state

You don't tell Kubernetes *what to do* ("start 3 containers"). You tell it *what you want* ("there should be 3 copies of this app"). Kubernetes runs **controllers**, which are loops that constantly compare **desired state** against **actual state** and take action to close the gap.

```
        you: "I want 3 replicas of myapp:v1"
                       │  (kubectl apply)
                       ▼
              ┌─────────────────┐
              │   API server    │ ──► stores desired state in etcd
              └─────────────────┘
                       ▲
      watch & act      │
              ┌─────────────────┐
              │   Controllers   │  "actual = 2, desired = 3 → create 1 pod"
              └─────────────────┘
```

This is why Kubernetes "self-heals". Nothing special happens on a crash. The actual state drifts from the desired state, and a controller fixes it.

#### Cluster architecture

```
┌──────────────────────── Control plane ────────────────────────┐
│  kube-apiserver   the front door; everything talks to it       │
│  etcd             the database holding all cluster state       │
│  kube-scheduler   decides which node a new pod runs on         │
│  controller-mgr   runs the reconciliation loops                │
└────────────────────────────────────────────────────────────────┘
          ▲                    ▲                    ▲
┌─────────┴────────┐ ┌─────────┴────────┐ ┌─────────┴────────┐
│ Node 1           │ │ Node 2           │ │ Node 3           │
│  kubelet         │ │  kubelet         │ │  kubelet         │
│  container       │ │  container       │ │  container       │
│  runtime         │ │  runtime         │ │  runtime         │
│  kube-proxy      │ │  kube-proxy      │ │  kube-proxy      │
│  [pod][pod]      │ │  [pod]           │ │  [pod][pod]      │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

- **kubelet:** the agent on each node. It makes sure the containers for pods assigned to that node are running and healthy.
- **kube-proxy:** programs networking rules so Services work.
- `kubectl` is just a client that sends HTTP requests to the API server.

#### The objects you'll use constantly

| Object | What it is | Analogy |
|---|---|---|
| **Pod** | One or more containers sharing a network namespace and storage. The smallest deployable unit. Ephemeral, with an IP that changes when it's replaced. | A single running app instance |
| **ReplicaSet** | Keeps N identical pods running. You rarely create one directly. | "Keep 3 copies alive" |
| **Deployment** | Manages ReplicaSets to give rolling updates and rollbacks. The standard way to run stateless apps. | "Run this app, 3 copies, update safely" |
| **Service** | A stable virtual IP and DNS name that load-balances across pods matching a label selector. | A load balancer + DNS entry |
| **ConfigMap / Secret** | Configuration / sensitive data injected into pods as env vars or files. | Config files / .env |
| **Namespace** | A virtual partition of the cluster for names, quotas, and access control. | A folder |
| **Ingress** | HTTP(S) routing from outside the cluster to Services. | A reverse proxy config |
| **PersistentVolumeClaim** | A request for durable storage that survives pod restarts. | "Give me a disk" |
| **Job / CronJob** | Run-to-completion work, one-off or scheduled. | A batch job / cron |

#### Labels and selectors, the glue

Objects are connected by **labels**, not by name. A Service finds its pods like this:

```yaml
# On the pod:        labels: { app: myapp }
# On the Service:    selector: { app: myapp }   → "all pods with app=myapp"
```

A huge share of "my Service doesn't work" bugs are a label/selector mismatch.

#### What happens when you create a Deployment?

1. `kubectl apply` sends the Deployment to the API server, which stores it in etcd.
2. The **Deployment controller** creates a **ReplicaSet**.
3. The **ReplicaSet controller** creates **Pod** objects (not yet running anywhere).
4. The **scheduler** assigns each pod to a node.
5. That node's **kubelet** sees the assignment, pulls the image, and starts the containers.
6. The kubelet reports status back to the API server.

Every step is a separate component watching the API server. You'll see the trail in Stage 3 with `kubectl get events`.

#### Anatomy of every manifest

```yaml
apiVersion: apps/v1        # which API group/version defines this kind
kind: Deployment           # the type of object
metadata:                  # identity: name, namespace, labels, annotations
  name: myapp
spec:                      # what YOU want (desired state)
  ...
# status:                  # what IS (actual state), written by Kubernetes, not you
```

### Lab 1.1: Paper exercise (15 minutes)

Without looking back, draw the diagram of what happens when you run `kubectl apply -f deployment.yaml` (replicas: 3). Label every component. Then compare it with the lesson above.

### Lab 1.2: Predict the behavior

Before Stage 3, write down what you think happens for each of these. You'll test them later.

1. You delete one pod owned by a 3-replica Deployment.
2. You change the image tag in the Deployment.
3. A node dies.
4. You scale from 3 to 5.
5. A Service selects a label no pod has.

### Checkpoint

1. Why does Kubernetes use "desired state" instead of commands?
2. Name the four control plane components and what each does.
3. Why can't you rely on a pod's IP address?
4. What's the difference between a Deployment and a ReplicaSet?
5. How does a Service know which pods to send traffic to?

<details><summary>Answers</summary>

1. It makes the system self-healing and idempotent. Controllers continuously reconcile, so failures and drift are corrected automatically.
2. API server (front door), etcd (state store), scheduler (picks nodes), controller manager (runs reconciliation loops).
3. Pods are ephemeral. When replaced, they get new IPs. Use a Service for a stable address.
4. A ReplicaSet maintains N pods of one template. A Deployment manages ReplicaSets over time to do rolling updates and rollbacks.
5. A label selector that matches pod labels.
</details>

### Further reading

- Kubernetes docs, [Concepts → Overview](https://kubernetes.io/docs/concepts/overview/)
- The interactive [Kubernetes Basics tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

---

## Stage 2: A local cluster and kubectl

**Time:** 1–2 days  **Goal:** create a multi-node local cluster, get fluent with `kubectl`, and learn to diagnose failures.

### Lesson

#### Local cluster options

| Tool | How it works | Good for |
|---|---|---|
| **kind** | Each "node" is a Docker container running Kubernetes. Fast, disposable, multi-node, great for CI. | **Recommended**: used throughout this plan |
| **minikube** | A VM or container with batteries included (dashboard, addons). | Beginners who want built-in extras |
| **Docker Desktop / OrbStack K8s** | One-click single-node cluster. | Zero setup, less flexible |

You pick the tool. The Kubernetes concepts are identical. This plan uses kind because clusters take about 30 seconds to create and destroy.

#### kubectl fundamentals

`kubectl` reads `~/.kube/config` to know which cluster to talk to (the current **context**). The command shape is:

```
kubectl <verb> <resource> [name] [flags]
          get    pods      myapp   -n dev -o yaml
```

| Verb | Use |
|---|---|
| `get` | List objects. `-o wide`, `-o yaml`, `-o json`, `-w` (watch), `-l key=val` (label filter) |
| `describe` | Human-readable detail **including events**, the first stop for debugging |
| `logs` | Container stdout/stderr. `-f` follow, `--previous` the last crashed container |
| `exec` | Run a command in a container. `-it ... -- sh` for a shell |
| `apply` | Create or update from YAML (declarative). The one you'll use most |
| `delete` | Remove objects |
| `explain` | Built-in docs for any field, e.g. `kubectl explain deployment.spec.strategy` |

**Imperative vs declarative:**

- *Imperative* (`kubectl run`, `kubectl create`) is quick for experiments.
- *Declarative* (`kubectl apply -f file.yaml`) is what you use for real work. The YAML is the source of truth and can be version-controlled.

### Lab 2.1: Create the cluster

**`labs/kind-config.yaml`**: one control plane and two workers. The extra settings prepare for Ingress in Stage 5.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: learn
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:        # lets http://localhost reach the cluster
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
kind create cluster --config labs/kind-config.yaml
kubectl cluster-info --context kind-learn
kubectl get nodes
```

Expected: three nodes, all `Ready`. (Ports 80/443 on your Mac must be free.)

### Lab 2.2: Peek behind the curtain

```bash
# Each kind node is just a Docker container
docker ps --format '{{.Names}}'
#   learn-control-plane, learn-worker, learn-worker2

# The control plane is itself made of pods
kubectl get pods -n kube-system -o wide
```

Find `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `kube-proxy`, and `coredns`. Note which node each is on. Everything in Stage 1's diagram is running here.

```bash
kubectl config get-contexts        # which clusters kubectl knows about
kubectl config current-context     # kind-learn
kubectl api-resources | head -30   # every kind of object the cluster supports
```

### Lab 2.3: Your first pod, imperatively

```bash
kubectl run hello --image=nginx:1.27
kubectl get pods                       # ContainerCreating → Running
kubectl get pods -o wide               # which node? what IP?
kubectl describe pod hello             # read the Events section at the bottom
kubectl logs hello
kubectl exec -it hello -- sh           # poke around; `exit` to leave
kubectl port-forward pod/hello 8081:80 # then curl localhost:8081 in another terminal
kubectl delete pod hello
```

In `describe`, read the Events: `Scheduled` → `Pulling` → `Pulled` → `Created` → `Started`. That's the lifecycle from Stage 1.

### Lab 2.4: The same pod, declaratively

**`labs/k8s/pod.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f labs/k8s/pod.yaml
kubectl get pod hello -o yaml          # note the huge `status:` block Kubernetes added
kubectl apply -f labs/k8s/pod.yaml     # "unchanged": apply is idempotent
kubectl delete -f labs/k8s/pod.yaml
```

Generate YAML instead of writing it from scratch:

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2 --dry-run=client -o yaml
kubectl explain pod.spec.containers.resources        # field docs from the terminal
```

### Lab 2.5: Break things on purpose

Learning to read failures matters more than learning the happy path.

**Wrong image:**

```bash
kubectl run broken --image=nginx:does-not-exist
kubectl get pods                       # ErrImagePull → ImagePullBackOff
kubectl describe pod broken            # Events show the exact pull error
kubectl delete pod broken
```

**Crashing container:**

```bash
kubectl run crasher --image=busybox:1.36 -- sh -c 'echo "about to fail"; exit 1'
kubectl get pods -w                    # Error → CrashLoopBackOff (growing delays)
kubectl logs crasher                   # "about to fail"
kubectl logs crasher --previous        # logs from the prior crashed attempt
kubectl delete pod crasher
```

**Unschedulable pod:**

```bash
kubectl run hungry --image=nginx:1.27 --overrides='{"spec":{"containers":[{"name":"hungry","image":"nginx:1.27","resources":{"requests":{"cpu":"1000"}}}]}}'
kubectl get pods                       # Pending forever
kubectl describe pod hungry            # "0/3 nodes are available: Insufficient cpu"
kubectl delete pod hungry
```

### Lab 2.6: Quality-of-life setup

```bash
# ~/.zshrc
alias k=kubectl
source <(kubectl completion zsh)
compdef k=kubectl
```

Try `k9s` (a terminal UI). Press `:pods`, then arrow keys, then `l` for logs, `d` for describe, `s` for shell, and `:q` to quit.

### Checkpoint

1. What's the first command you run when a pod isn't healthy? What section do you read?
2. What does `CrashLoopBackOff` mean? How do you see why it crashed?
3. What does `Pending` usually indicate?
4. What does applying the same YAML twice do?
5. How does a kind "node" differ from a real node?

<details><summary>Answers</summary>

1. `kubectl describe pod <name>`, then the **Events** section at the bottom. Follow with `kubectl logs`.
2. The container starts, exits, and Kubernetes restarts it with increasing back-off delays. Use `kubectl logs <pod> --previous`.
3. The pod has been accepted but not scheduled to a node. Look for insufficient resources, taints, or unbound volumes in `describe`.
4. Nothing the second time, because `apply` is idempotent and reports "unchanged".
5. A kind node is a Docker container pretending to be a machine. Functionally it behaves the same for learning.
</details>

### Challenges

- Delete the cluster (`kind delete cluster --name learn`) and recreate it. Time it.
- Use `kubectl get pods -A -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName` to list every pod and its node.
- Use `kubectl explain` to find what `restartPolicy` values exist.

---

## Stage 3: Deploy your own app

**Time:** 3–5 days  **Goal:** run your image as a Deployment, expose it with a Service, and watch scaling, self-healing, rolling updates, and rollbacks.

### Lesson

#### Why Deployments, not bare pods?

A bare pod isn't rescheduled if its node dies or if you delete it. A **Deployment** owns a **ReplicaSet**, which owns the pods:

```
Deployment (myapp)           ← you manage this
  └─ ReplicaSet (myapp-7d9f…)   ← one per pod-template version
       ├─ Pod myapp-7d9f…-abcde
       ├─ Pod myapp-7d9f…-fghij
       └─ Pod myapp-7d9f…-klmno
```

On a template change (new image, new env), the Deployment creates a **new ReplicaSet** and shifts pods from old to new gradually. That is a **rolling update**. The old ReplicaSet is kept (scaled to 0) so you can **roll back**.

#### Rolling update knobs

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # never go below the desired count
    maxSurge: 1         # allow one extra pod during the update
```

#### Services

A Service gives pods a stable address.

| Type | Reachable from | Use |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Service-to-service traffic |
| **NodePort** | Each node's IP on a high port | Quick tests |
| **LoadBalancer** | An external cloud load balancer | Production on cloud (on kind it stays `<pending>`) |

Key fields:

```yaml
ports:
  - port: 80           # the port the SERVICE listens on
    targetPort: 8080   # the port the POD's container listens on
```

#### Getting local images into kind

kind nodes can't see your Mac's Docker images. Load them:

```bash
kind load docker-image myapp:v1 --name learn
```

Because the tag isn't `latest`, Kubernetes won't try to pull it from a registry (`imagePullPolicy: IfNotPresent`).

### Lab 3.1: Deploy

**`labs/k8s/myapp.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: myapp           # must match the selector above
    spec:
      containers:
        - name: myapp
          image: myapp:v1
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kind load docker-image myapp:v1 myapp:v2 --name learn
kubectl apply -f labs/k8s/myapp.yaml
kubectl get deploy,rs,pods,svc -l app=myapp
kubectl get pods -o wide                       # spread across the two workers
kubectl get endpointslices -l kubernetes.io/service-name=myapp
```

The EndpointSlice lists the three pod IPs. That is how the Service knows where to send traffic.

### Lab 3.2: Reach it

```bash
kubectl port-forward svc/myapp 8080:80
# in another terminal:
curl localhost:8080/
```

**Gotcha:** `port-forward` tunnels to **one** pod. Hitting it repeatedly will not show load balancing. To see real balancing, call the Service from *inside* the cluster:

```bash
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -c 'for i in 1 2 3 4 5 6; do curl -s http://myapp/; echo; done'
```

You'll see different `hostname` values, which are the pod names.

### Lab 3.3: Self-healing

```bash
kubectl get pods -w                  # terminal 1
kubectl delete pod <one-of-the-pods> # terminal 2
```

A replacement appears almost immediately. Then try to remember why. (Stage 1: the ReplicaSet controller saw actual = 2, desired = 3.)

```bash
kubectl get events --sort-by=.lastTimestamp | tail -20
```

Read the trail: `SuccessfulCreate`, `Scheduled`, `Pulled`, `Started`.

### Lab 3.4: Scaling

```bash
kubectl scale deployment myapp --replicas=5
kubectl get pods
kubectl scale deployment myapp --replicas=2
# Declarative way: edit replicas in the YAML and `kubectl apply -f` again.
```

### Lab 3.5: Rolling update and rollback

```bash
# terminal 1: watch pods
kubectl get pods -w

# terminal 2: update the image
kubectl set image deployment/myapp myapp=myapp:v2
kubectl rollout status deployment/myapp
kubectl get rs                                   # old RS scaled to 0, new RS has the pods

# verify from inside the cluster
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- curl -s http://myapp/
# → "version": "v2"

kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp            # back to v1
kubectl rollout status deployment/myapp
```

> **Tip:** `kubectl set image` is imperative, so your YAML file no longer matches the cluster. In real work, change the tag in the YAML and `kubectl apply`. Try that for the next update.

### Lab 3.6: A bad rollout

This is why `maxUnavailable: 0` and readiness checks matter.

```bash
kubectl set image deployment/myapp myapp=myapp:v3-does-not-exist
kubectl rollout status deployment/myapp --timeout=30s    # times out
kubectl get pods
```

You'll see 1 new pod in `ImagePullBackOff`, while **the old pods keep serving traffic**. Curl the Service to confirm. Recover:

```bash
kubectl rollout undo deployment/myapp
```

### Lab 3.7: Break the Service on purpose

```bash
kubectl patch svc myapp -p '{"spec":{"selector":{"app":"typo"}}}'
kubectl get endpointslices -l kubernetes.io/service-name=myapp   # no endpoints!
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- curl -s -m 3 http://myapp/
# → hangs/times out
kubectl apply -f labs/k8s/myapp.yaml                              # fix by re-applying
```

Lesson: when a Service doesn't work, check its **endpoints** first.

### Checkpoint

1. What's the difference between `port` and `targetPort`?
2. Why did the pod come back after you deleted it?
3. How do you see which pods back a Service?
4. What happens to old ReplicaSets after an update, and why?
5. Why didn't `port-forward` show load balancing?
6. During a bad rollout, why did users not notice?

<details><summary>Answers</summary>

1. `port` is what the Service listens on. `targetPort` is the container's port.
2. The ReplicaSet controller reconciles actual vs. desired replica count.
3. `kubectl get endpointslices -l kubernetes.io/service-name=<svc>`, or `kubectl get pods -l <selector>`.
4. They're kept at 0 replicas to enable rollbacks (`revisionHistoryLimit`, default 10).
5. It connects to a single pod. It bypasses the Service's balancing.
6. `maxUnavailable: 0` prevents old pods from being removed until new ones are Ready.
</details>

### Challenges

- Change `maxSurge`/`maxUnavailable` and observe different update patterns.
- Add `kubectl rollout pause` and `resume` mid-update.
- Add a `NodePort` Service and find out why you still can't hit it from your Mac on kind (hint: kind nodes are containers, and only the ports in `extraPortMappings` are exposed).

---

## Stage 4: Configuration, health, and state

**Time:** 3–5 days  **Goal:** externalize config, add health checks and resource limits, run a stateful database with persistent storage, and use namespaces.

### Lesson

#### Configuration

- **ConfigMap:** non-sensitive config (key/value pairs or whole files).
- **Secret:** sensitive values. **Base64 is encoding, not encryption.** Anyone who can read the Secret can decode it. In real clusters, restrict access with RBAC and enable encryption at rest.

Both can be consumed as:

1. **Environment variables** (`env`, `envFrom`). They are **not** updated live. The pod must restart.
2. **Mounted files** (volumes). The files update eventually (about a minute) without restarting, but your app must re-read them.

#### Probes: how Kubernetes knows your app is healthy

| Probe | Question | On failure |
|---|---|---|
| **startupProbe** | Has it finished starting? | Keeps other probes off until it passes, then restarts if it never does |
| **readinessProbe** | Can it take traffic *right now*? | Removed from Service endpoints (**no restart**) |
| **livenessProbe** | Is it stuck/deadlocked? | **Container is restarted** |

Be careful with liveness probes. A badly tuned one can kill a healthy but slow app and cause restart storms. Start with readiness probes only.

#### Resource requests and limits

```yaml
resources:
  requests: { cpu: 50m, memory: 32Mi }    # what the scheduler RESERVES for placement
  limits:   { cpu: 200m, memory: 64Mi }   # hard ceiling
```

- `1000m` CPU = 1 core. CPU over the limit is **throttled**.
- Memory over the limit gets the container **OOMKilled**.
- Requests drive scheduling and (later) autoscaling.

#### Persistent storage

Pod filesystems are ephemeral. For durable data:

- **PersistentVolumeClaim (PVC):** "I need 1Gi of storage."
- A **StorageClass** provisions a **PersistentVolume** to satisfy it. kind includes a default StorageClass (`kubectl get storageclass`).

For databases in production you'd typically use a **StatefulSet** (stable identities and per-pod storage) or a managed database. A single-replica Deployment with a PVC is fine for learning.

#### Namespaces

Namespaces separate environments and teams within one cluster. Most resources are namespaced. Services are reachable across namespaces at `<service>.<namespace>`.

### Lab 4.1: ConfigMap → environment variables

**`labs/k8s/config.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  GREETING: "Hello from a ConfigMap"
```

Replace the Deployment from Stage 3 with this fuller version (the Service block stays the same). **`labs/k8s/myapp.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:v1
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f labs/k8s/config.yaml -f labs/k8s/myapp.yaml
kubectl rollout status deploy/myapp
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- curl -s http://myapp/
# → "message": "Hello from a ConfigMap"
```

**Config changes need a restart (for env vars):**

```bash
kubectl patch configmap myapp-config -p '{"data":{"GREETING":"Changed!"}}'
# curl again: still the old greeting
kubectl rollout restart deploy/myapp
# curl again after rollout: "Changed!"
```

### Lab 4.2: Secrets

```bash
kubectl create secret generic demo-secret --from-literal=API_KEY=s3cr3t
kubectl get secret demo-secret -o yaml                       # value looks scrambled…
kubectl get secret demo-secret -o jsonpath='{.data.API_KEY}' | base64 -d; echo   # …but isn't
kubectl delete secret demo-secret
```

Never commit real secrets to git. Real workflows use tools like Sealed Secrets, SOPS, or External Secrets. See Stage 7.

### Lab 4.3: Probes and failure

Edit `labs/k8s/myapp.yaml` to point the **readiness** probe at a non-existent path (`/nope`), then apply:

```bash
kubectl apply -f labs/k8s/myapp.yaml
kubectl get pods -w
kubectl describe pod <new-pod>        # Events: "Readiness probe failed: HTTP probe failed with statuscode: 404"
```

The new pods run (`STATUS Running`) but are `0/1` **Ready**. The rollout stalls, and the Service sends no traffic to them. The old pods keep serving. Fix the path and re-apply.

Now try **liveness**: point it at `/nope` instead. After about 30 seconds you'll see `RESTARTS` climbing. That's the kubelet killing "unhealthy" containers. Fix it again.

### Lab 4.4: Resource limits

```bash
kubectl top pods         # error: Metrics API not available (we install metrics-server in Stage 7)
kubectl describe node learn-worker | sed -n '/Allocated resources/,/Events/p'
```

Now set `limits.memory: 8Mi` in the Deployment and apply. The container will likely die with `OOMKilled` or fail to start. Check:

```bash
kubectl describe pod <pod>      # Last State: Terminated, Reason: OOMKilled (or similar)
```

Restore the real values.

### Lab 4.5: Postgres with persistent storage

**`labs/k8s/postgres.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:                     # plain text here; Kubernetes stores it base64-encoded
  POSTGRES_PASSWORD: change-me
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  strategy:
    type: Recreate              # a ReadWriteOnce volume can't be shared by old+new pod at once
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
```

```bash
kubectl apply -f labs/k8s/postgres.yaml
kubectl get pvc,pv                                   # PVC should become Bound
kubectl rollout status deploy/postgres

# Write data
kubectl exec deploy/postgres -- psql -U postgres -c \
  "CREATE TABLE notes (id serial PRIMARY KEY, body text); INSERT INTO notes (body) VALUES ('survives restarts');"

# Kill the pod
kubectl delete pod -l app=postgres
kubectl rollout status deploy/postgres
kubectl exec deploy/postgres -- psql -U postgres -c "SELECT * FROM notes;"   # still there
```

Now prove the point of the PVC. Delete the PVC (after deleting the Deployment) and re-create everything. The data is gone, because persistent storage lasts only as long as its claim.

Connect from **another pod** through the Service DNS name:

```bash
kubectl run psql --rm -it --restart=Never --image=postgres:16 --env PGPASSWORD=change-me -- \
  psql -h postgres -U postgres -c "SELECT count(*) FROM notes;"
```

### Lab 4.6: Namespaces

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl apply -n dev -f labs/k8s/config.yaml -f labs/k8s/myapp.yaml
kubectl get pods -n dev
kubectl get pods -A                       # all namespaces

# Cross-namespace DNS, from a pod in `default`:
kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- curl -s http://myapp.dev/

# Set a default namespace for your context
kubectl config set-context --current --namespace=dev
kubectl config set-context --current --namespace=default   # switch back
```

### Checkpoint

1. Why doesn't changing a ConfigMap update env vars in running pods?
2. What's the difference between readiness and liveness probes? Which can cause restarts?
3. What happens when a container exceeds its memory limit? Its CPU limit?
4. Why did we use `strategy: Recreate` for Postgres?
5. Is a Secret encrypted? What actually protects it?
6. What's the DNS name for service `postgres` in namespace `data`, from a pod in `default`?

<details><summary>Answers</summary>

1. Env vars are set at container start. Restart the pods (`rollout restart`) or mount the ConfigMap as a file.
2. Readiness controls traffic (no restart). Liveness restarts the container.
3. Memory: OOMKilled. CPU: throttled.
4. A single ReadWriteOnce volume can't be attached to the old and new pod simultaneously, so we stop the old pod first.
5. No. It's base64-encoded. RBAC, encryption at rest, and external secret managers protect it.
6. `postgres.data` (full form: `postgres.data.svc.cluster.local`).
</details>

### Challenges

- Mount the ConfigMap as a file instead of env vars (`volumes` + `volumeMounts`) and watch it update without a restart.
- Add a `startupProbe` to Postgres.
- Create a `ResourceQuota` in `staging` that caps pods at 3 and see what happens when you scale to 5 (`kubectl explain resourcequota.spec`).

---

## Stage 5: Networking and a multi-tier app

**Time:** about 1 week  **Goal:** understand Service discovery and DNS, expose HTTP traffic with an Ingress, and run a frontend → API → database topology.

### Lesson

#### How pods find each other

Every Service gets a DNS name from **CoreDNS**:

```
<service>.<namespace>.svc.cluster.local
```

Within the same namespace the short name `myapp` works. Across namespaces use `myapp.dev`. **Always use Service names, never pod IPs.**

#### Getting traffic in from outside

```
Browser ──► localhost:80 ──► (kind port mapping) ──► Ingress controller pod
                                                          │ reads Ingress rules
                                                          ▼
                                              Service "web" ──► web pods ──► Service "myapp" ──► api pods
```

- An **Ingress** object is only *configuration*. It does nothing without an **Ingress controller** (a pod that implements the rules, e.g. ingress-nginx, Traefik).
- `ingressClassName` tells Kubernetes which controller owns the rule.

> **Heads-up:** the Kubernetes project has announced the retirement of the community `ingress-nginx` controller. It's still perfectly fine for learning how Ingress works locally, and the concepts transfer directly. For anything long-lived, evaluate the newer **Gateway API** (`HTTPRoute`) and the controllers that implement it. Check the current status in the ingress-nginx repo before relying on it.

### Lab 5.1: Install an Ingress controller

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=120s
kubectl get pods -n ingress-nginx
```

If that URL has moved, follow the "Ingress" page in the kind docs. It's the same idea.

### Lab 5.2: DNS exploration

```bash
kubectl run dns --rm -it --restart=Never --image=busybox:1.36 -- sh
# inside the pod:
nslookup myapp
nslookup myapp.default.svc.cluster.local
nslookup postgres
cat /etc/resolv.conf           # search domains explain why the short name works
exit
```

### Lab 5.3: Build a three-tier app

You already have two tiers (Stage 4):

- **api** → `myapp` (our Python service)
- **db** → `postgres`

Add a **web** tier: nginx serving a static page and proxying `/api/` to the API Service.

**`labs/k8s/web.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-nginx-conf
data:
  default.conf: |
    server {
      listen 80;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
      location /api/ {
        proxy_pass http://myapp/;       # Service DNS name; the trailing / strips /api
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-html
data:
  index.html: |
    <!doctype html>
    <html>
      <body>
        <h1>Frontend</h1>
        <pre id="out">loading...</pre>
        <script>
          fetch('/api/')
            .then(r => r.json())
            .then(j => { document.getElementById('out').textContent = JSON.stringify(j, null, 2); });
        </script>
      </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: conf
              mountPath: /etc/nginx/conf.d
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: conf
          configMap:
            name: web-nginx-conf
        - name: html
          configMap:
            name: web-html
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f labs/k8s/web.yaml
kubectl rollout status deploy/web
kubectl port-forward svc/web 8081:80
# curl localhost:8081/        → the HTML
# curl localhost:8081/api/    → JSON from the api, proxied by nginx
```

> nginx resolves `myapp` when it starts. If the `myapp` Service doesn't exist yet, nginx fails to start and the pod crash-loops. This is a real-world ordering problem, and readiness checks and retries are the usual fix.

### Lab 5.4: Expose it with Ingress

**`labs/k8s/ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

```bash
kubectl apply -f labs/k8s/ingress.yaml
kubectl get ingress
curl -H 'Host: myapp.localhost' http://localhost/        # always works
curl -H 'Host: myapp.localhost' http://localhost/api/
open http://myapp.localhost/                              # *.localhost resolves to 127.0.0.1 in most browsers
```

### Lab 5.5: Verify the data tier is reachable from the API

```bash
kubectl exec deploy/myapp -- python -c \
  "import socket; s=socket.create_connection(('postgres',5432),2); print('connected to postgres:', s.getpeername())"
```

This proves service discovery, DNS, and pod-to-pod networking. (A real app would use a Postgres driver and read the `POSTGRES_PASSWORD` Secret. Wiring that in is a good challenge.)

### Lab 5.6: Watch the whole request path

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=5   # the ingress saw it
kubectl logs -l app=web --tail=5                                                   # nginx saw it
kubectl logs -l app=myapp --tail=5                                                 # the api saw it
```

Three hops, three sets of logs. This is how you debug request flow.

### Checkpoint

1. What's the full DNS name of a Service, and when can you use the short form?
2. What's the difference between an Ingress and an Ingress controller?
3. A user gets a 502 from the Ingress. Where do you look?
4. What does `LoadBalancer` do on kind?
5. Why use `proxy_pass http://myapp/` rather than a pod IP?

<details><summary>Answers</summary>

1. `<svc>.<ns>.svc.cluster.local`. The short form works within the same namespace.
2. The Ingress is the routing rule (data). The controller is the running proxy that implements it.
3. Ingress controller logs → the Service's endpoints (`get endpointslices`) → the pod's readiness and logs.
4. Nothing by default. It stays `<pending>` because there's no cloud provider. Tools like `cloud-provider-kind` or MetalLB add one.
5. Pod IPs change when pods are replaced. The Service name is stable and load-balanced.
</details>

### Challenges

- Make the app talk to Postgres for real (psycopg), reading the password from the Secret.
- Route `/api` through the Ingress **directly** to `myapp` (look up the `rewrite-target` annotation) and drop the nginx proxy.
- Add a second host (`api.localhost`) in the same Ingress.
- Try a `NetworkPolicy` that only lets `web` talk to `myapp`. Note that enforcement depends on the CNI plugin, so verify it actually blocks traffic in your setup.

---

## Stage 6: Developer workflow, packaging, and testing

**Time:** about 1 week  **Goal:** package manifests with Kustomize and Helm, get a fast inner dev loop, and test deployments locally and in CI.

### Lesson

#### Why packaging?

Copying YAML for dev, staging, and prod doesn't scale. Two main tools:

| | **Kustomize** | **Helm** |
|---|---|---|
| Approach | Overlay/patch plain YAML. **No templating.** | Go-templated charts with `values.yaml` |
| Built into kubectl | Yes (`kubectl apply -k`) | No (separate CLI) |
| Strength | Simple per-environment differences | Reusable, versioned, installable packages; huge public ecosystem; release tracking and rollback |
| Weakness | Less flexible for complex logic | Templates can get hard to read |

You'll likely use both. Kustomize for your own app's environments, Helm for third-party software (databases, ingress controllers, monitoring).

### Lab 6.1: Kustomize: base + overlays

```
labs/kustomize/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml
    └── staging/kustomization.yaml
```

**`base/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:v1
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
          resources:
            requests: { cpu: 50m, memory: 32Mi }
            limits: { cpu: 200m, memory: 64Mi }
```

**`base/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**`base/kustomization.yaml`**

```yaml
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: myapp-config
    literals:
      - GREETING=Hello from base
```

**`overlays/dev/kustomization.yaml`**

```yaml
namespace: dev
resources:
  - ../../base
replicas:
  - name: myapp
    count: 1
images:
  - name: myapp
    newTag: v1
configMapGenerator:
  - name: myapp-config
    behavior: merge
    literals:
      - GREETING=Hello from dev
```

**`overlays/staging/kustomization.yaml`**

```yaml
namespace: staging
resources:
  - ../../base
replicas:
  - name: myapp
    count: 3
images:
  - name: myapp
    newTag: v2
configMapGenerator:
  - name: myapp-config
    behavior: merge
    literals:
      - GREETING=Hello from staging
```

```bash
kubectl kustomize labs/kustomize/overlays/dev       # render only: always inspect first
kubectl diff -k labs/kustomize/overlays/staging     # what WOULD change
kubectl apply -k labs/kustomize/overlays/dev
kubectl apply -k labs/kustomize/overlays/staging
kubectl get pods -n dev; kubectl get pods -n staging
kubectl get cm -n staging                            # note the hash suffix on the name
```

**Observe:** the generated ConfigMap has a hash suffix (`myapp-config-7k2f…`), and the Deployment's reference is rewritten automatically. Change the greeting and re-apply, and a new ConfigMap name triggers an automatic rolling restart. That solves the "env vars don't update" problem from Stage 4.

### Lab 6.2: Helm: a small chart

```
labs/chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

**`Chart.yaml`**

```yaml
apiVersion: v2
name: myapp
description: Learning chart for the sample app
version: 0.1.0
appVersion: "v1"
```

**`values.yaml`**

```yaml
replicaCount: 2
image:
  repository: myapp
  tag: v1
greeting: Hello from Helm
service:
  port: 80
resources:
  requests: { cpu: 50m, memory: 32Mi }
  limits: { cpu: 200m, memory: 64Mi }
```

**`templates/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: myapp
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          env:
            - name: GREETING
              value: {{ .Values.greeting | quote }}
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**`templates/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8080
```

```bash
helm lint labs/chart
helm template hello labs/chart                       # render locally: nothing touches the cluster
helm install hello labs/chart -n helm-demo --create-namespace
helm list -n helm-demo
kubectl get pods -n helm-demo

# Upgrade with overrides
helm upgrade hello labs/chart -n helm-demo --set image.tag=v2 --set replicaCount=3
helm history hello -n helm-demo
helm rollback hello 1 -n helm-demo                   # back to revision 1
helm get values hello -n helm-demo
helm uninstall hello -n helm-demo
```

Explore a real chart to see where this goes:

```bash
helm create scratch-chart        # Helm's generated starter, full of best-practice patterns
```

### Lab 6.3: Fast inner loop with Tilt

Rebuilding, loading, and re-applying by hand gets tedious. **Tilt** watches your code and does it automatically.

**`labs/Tiltfile`**

```python
# Rebuild the image when files in ./app change; Tilt loads it into kind for you
docker_build('myapp', './app')

# Use the Stage 4 manifests
k8s_yaml(['k8s/config.yaml', 'k8s/myapp.yaml'])

# Port-forward and show it in the Tilt UI
k8s_resource('myapp', port_forwards='8080:8080')
```

```bash
brew install tilt-dev/tap/tilt
cd labs && tilt up           # opens a web UI; press space for the browser
# Edit labs/app/app.py (e.g. change the JSON), save, and watch it rebuild + redeploy.
# `tilt down` cleans up.
```

If Tilt doesn't replace the image, check that the image *name* in your Tiltfile matches the `image:` in the manifest (`myapp`). Tilt tags its own builds and rewrites the reference.

An alternative with the same idea is **Skaffold** (`skaffold dev`).

### Lab 6.4: Test your manifests

Layers of testing, from fastest to most realistic:

**1. Static validation (no cluster needed)**

```bash
kubeconform -strict -summary labs/k8s/                 # validate against the K8s schemas
kubectl kustomize labs/kustomize/overlays/dev | kubeconform -strict -summary
helm template hello labs/chart | kubeconform -strict -summary
```

**2. Server-side validation (needs a cluster, changes nothing)**

```bash
kubectl apply --dry-run=server -f labs/k8s/myapp.yaml  # API server validates AND runs admission
kubectl diff -f labs/k8s/myapp.yaml                    # show exactly what would change
```

**3. Smoke test against a real (throwaway) cluster**

**`labs/scripts/smoke-test.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

NS="${1:-default}"
NAME="${2:-myapp}"

echo "==> waiting for rollout"
kubectl -n "$NS" rollout status "deploy/$NAME" --timeout=90s

echo "==> checking health endpoint from inside the cluster"
kubectl -n "$NS" run "smoke-$RANDOM" --rm -i --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl -fsS "http://$NAME.$NS/healthz"
echo
echo "==> OK"
```

```bash
chmod +x labs/scripts/smoke-test.sh
labs/scripts/smoke-test.sh dev myapp
```

**4. Fully ephemeral test cluster, the CI-style flow**

```bash
kind create cluster --name ci
docker build -t myapp:v1 labs/app
kind load docker-image myapp:v1 --name ci
helm install hello labs/chart -n test --create-namespace --wait --timeout 90s
labs/scripts/smoke-test.sh test hello
kind delete cluster --name ci
```

`--wait` makes Helm block until pods are Ready, so a failing install fails the script.

**5. In GitHub Actions** (optional): the `helm/kind-action` action creates a kind cluster in CI, and then you run the same commands as step 4.

### Lab 6.5: Get fluent in k9s

| Key | Action |
|---|---|
| `:pods`, `:deploy`, `:svc`, `:ns` | Switch resource view |
| `/text` | Filter |
| `Enter` | Drill in |
| `l` | Logs |
| `d` | Describe |
| `s` | Shell into container |
| `y` | View YAML |
| `Ctrl-d` | Delete |
| `:q` | Quit |

Repeat Lab 3.3 (delete a pod and watch it come back) entirely inside k9s.

### Checkpoint

1. When would you choose Kustomize over Helm, and vice versa?
2. What does `helm template` do, and why is it useful?
3. What's the difference between `--dry-run=client` and `--dry-run=server`?
4. Why does the Kustomize ConfigMap generator add a hash suffix?
5. Why use `kind load` rather than pushing to a registry for local development?

<details><summary>Answers</summary>

1. Kustomize for plain YAML with per-environment tweaks. Helm for reusable or third-party packages and release lifecycle (install, upgrade, rollback).
2. It renders manifests locally without installing, so you can inspect or validate them and pipe them to other tools.
3. `client` only checks locally. `server` sends the request to the API server for full validation and admission, but doesn't persist it.
4. A content change gives a new name, which triggers a rolling restart of anything that references it.
5. It's faster, offline, and needs no registry credentials or setup.
</details>

### Challenges

- Add a `prod` overlay with 5 replicas and higher resource limits.
- Add a Helm `values-staging.yaml` and install with `-f values-staging.yaml`.
- Write a CI job (or a local `make test`) that runs lint, schema validation, a kind install, and the smoke test.

---

## Stage 7: Beyond the basics

**Time:** ongoing  **Goal:** a taste of the features you'll meet next, plus a roadmap.

### Lab 7.1: Jobs and CronJobs

**`labs/k8s/jobs.yaml`**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox:1.36
          command: ["sh", "-c", "echo running a one-off task; sleep 3; echo done"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "date; echo hello from cron"]
```

```bash
kubectl apply -f labs/k8s/jobs.yaml
kubectl get jobs,pods -w
kubectl logs job/hello-job
# wait a couple of minutes:
kubectl get cronjob,jobs
kubectl delete -f labs/k8s/jobs.yaml
```

### Lab 7.2: Autoscaling with HPA

HPA needs a metrics source. On kind, install metrics-server and allow its insecure kubelet TLS (fine locally only):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch -n kube-system deployment metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl rollout status -n kube-system deploy/metrics-server
kubectl top nodes && kubectl top pods         # now works (give it a minute)
```

**`labs/k8s/hpa.yaml`**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50     # % of the pod's CPU *request*, so requests must be set
```

```bash
kubectl apply -f labs/k8s/hpa.yaml
kubectl get hpa -w                    # terminal 1

# terminal 2: generate load
kubectl run load --rm -it --restart=Never --image=busybox:1.36 -- \
  sh -c 'while true; do wget -q -O- http://myapp/ >/dev/null; done'
```

Watch `TARGETS` climb and `REPLICAS` grow. Stop the load (Ctrl-C) and, after the stabilization window (about 5 minutes), it scales back down.

### Lab 7.3: RBAC: who can do what

**`labs/k8s/rbac.yaml`**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
subjects:
  - kind: ServiceAccount
    name: pod-reader
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f labs/k8s/rbac.yaml
SA=system:serviceaccount:default:pod-reader
kubectl auth can-i list pods   --as=$SA      # yes
kubectl auth can-i delete pods --as=$SA      # no
kubectl auth can-i list secrets --as=$SA     # no
```

**Principle of least privilege:** pods that talk to the API (operators, CI agents) get a dedicated ServiceAccount with the narrowest Role possible.

### Roadmap: where to go next

| Topic | What to learn | Good first step |
|---|---|---|
| **Observability** | Metrics, logs, traces, dashboards | `kube-prometheus-stack` Helm chart (Prometheus + Grafana) on kind |
| **GitOps** | Cluster state driven from git | Argo CD or Flux on kind, pointed at a repo of your manifests |
| **Secrets management** | Safe handling in git | Sealed Secrets, SOPS, or External Secrets Operator |
| **StatefulSets & operators** | Databases and clustered apps | A Postgres operator (CloudNativePG) |
| **Security** | Pod Security Standards, NetworkPolicy, image scanning | Run your app as non-root with a read-only root filesystem |
| **Gateway API** | The successor to Ingress | Try `HTTPRoute` with a Gateway controller |
| **Managed cloud clusters** | EKS / GKE / AKS | Deploy the same chart to a small cloud cluster and compare |
| **Cluster internals** | How it really works | Kelsey Hightower's *Kubernetes the Hard Way* (later!) |

### Capstone project

Take your `financial-tracker` project (or any small web + database app) and:

1. Containerize it with a pinned, non-root image.
2. Write Deployments, Services, ConfigMaps, Secrets, and a PVC for its database.
3. Add readiness probes, resource requests/limits, and an Ingress at `tracker.localhost`.
4. Package it as a Helm chart or Kustomize base + overlays.
5. Add a smoke-test script and run it against a throwaway kind cluster.
6. Set up Tilt so `tilt up` gives you a live dev loop.

When you can do this without copying from this document, you know the basics.

---

## Appendix A: kubectl cheat sheet

```bash
# Inspect
kubectl get pods|deploy|svc|ing|cm|secret|pvc|ns|nodes
kubectl get all                                  # common namespaced objects (not literally all)
kubectl get pods -o wide | -o yaml | -o json
kubectl get pods -l app=myapp                    # label selector
kubectl get pods -A                              # all namespaces
kubectl get pods -w                              # watch
kubectl get events --sort-by=.lastTimestamp
kubectl describe <kind> <name>
kubectl explain <kind>.<field.path>
kubectl api-resources

# Debug
kubectl logs <pod> [-c container] [-f] [--previous] [--tail=50]
kubectl logs -l app=myapp --all-containers --tail=20
kubectl exec -it <pod> -- sh
kubectl port-forward svc/<name> LOCAL:REMOTE
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- sh   # throwaway debug pod
kubectl debug -it <pod> --image=busybox:1.36 --target=<container>     # attach an ephemeral container
kubectl top pods|nodes                                                # needs metrics-server

# Change
kubectl apply -f file.yaml | -f dir/ | -k overlay/
kubectl delete -f file.yaml
kubectl diff -f file.yaml
kubectl scale deploy/<name> --replicas=N
kubectl set image deploy/<name> <container>=<image:tag>
kubectl rollout status|history|undo|restart|pause|resume deploy/<name>
kubectl patch <kind> <name> -p '{"spec":{...}}'
kubectl edit <kind> <name>                       # opens $EDITOR (live edit; avoid for real work)
kubectl label|annotate <kind> <name> key=value

# Context / namespace
kubectl config get-contexts
kubectl config use-context kind-learn
kubectl config set-context --current --namespace=dev

# Access checks
kubectl auth can-i <verb> <resource> [--as=<user>]

# Output tricks
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
```

## Appendix B: Troubleshooting table

| Symptom | Likely cause | First commands |
|---|---|---|
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag; image not loaded into kind; private registry | `describe pod`; `kind load docker-image …`; check the tag isn't `latest` |
| `CrashLoopBackOff` | App exits on startup (bad config, missing env, can't reach dependency) | `logs --previous`; `describe pod` (exit code) |
| `Pending` | No node fits (resources, taints), unbound PVC | `describe pod` → Events; `get pvc` |
| `Running` but `0/1` Ready | Readiness probe failing | `describe pod` → probe events; `logs` |
| `OOMKilled` | Memory limit too low or leak | `describe pod` → Last State; raise the limit |
| Service has no response | Selector/label mismatch, wrong `targetPort`, pods not Ready | `get endpointslices`; `get pods --show-labels` |
| Ingress 404/502/503 | Wrong host/path, backend Service has no endpoints, controller issue | `describe ingress`; controller logs; endpoints |
| Rollout stuck | New pods failing readiness/pulling | `rollout status`; `get pods`; `rollout undo` |
| `CreateContainerConfigError` | Missing ConfigMap/Secret/key referenced | `describe pod` |
| Changes "not applying" | Wrong context/namespace; env var change without restart | `config current-context`; `rollout restart` |
| kind: ports 80/443 busy | Another service on your Mac | `lsof -i :80`; stop it or change `hostPort` |

**Debug order:** `get` → `describe` (read Events) → `logs` (and `--previous`) → `exec`/`debug` → check Services/endpoints → check config (ConfigMaps, Secrets, env).

## Appendix C: Cleanup

```bash
# Remove only what you created in a namespace
kubectl delete namespace dev staging helm-demo test

# Remove everything: delete the whole cluster (30 seconds to recreate)
kind delete cluster --name learn

# Reclaim Docker disk space
docker image prune -a
```
