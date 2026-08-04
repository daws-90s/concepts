# Kubernetes

## Why Kubernetes? — Where Docker Stops

Docker gets you as far as *one host*. That's the whole problem:

| Docker's limit | What breaks |
|----------------|-------------|
| **Single host** | You can't rely on one machine — it dies, your app dies |
| **Multiple hosts = a cluster** | A cluster needs a **master** (an orchestrator) and **nodes**. Docker has no such brain. |
| **Cross-host networking** | Containers on different hosts can't resolve each other — you need an **overlay network** |
| **No autoscaling** | Traffic doubles, nothing happens |
| **No load balancing** | Nothing spreads traffic across container copies |
| **No secrets management** | Passwords end up in plain text (as they do in the roboshop compose file) |
| **Unsafe volume management** | Data is pinned to one host's disk |

So the split from here on is:

```
Build the image  →  Docker
Run the image    →  Kubernetes
```

### Docker Swarm vs Kubernetes

**Docker Swarm** is Docker's own native orchestrator. **Kubernetes** is what the industry actually uses:

1. **From Google** — highly stable, huge community
2. **PaaS-like** — integrates with cloud services: secrets managers, load balancers, storage
3. **Better load balancing** — Swarm has native LB, but it's not close to Kubernetes + nginx
4. **Better volume management**
5. **Better networking**
6. **Deployment strategies** — blue/green, rolling update, A/B, canary
7. **Helm charts** — packaged, reusable, templated deployments

---

## Cluster Architecture

A cluster is **master + nodes**:

- **Master (control plane)** — the **orchestrator**. It decides what runs where, and keeps reality matching your declared intent.
- **Worker nodes** — the machines that actually run your containers.

You never tell Kubernetes *how* to do something. You declare **desired state** ("I want 10 copies of this image running") and the control plane's job is to continuously make the cluster match it. That single idea explains almost everything else in this document.

### Workstation tooling

Four things on the machine you drive the cluster from:

| Tool | Purpose |
|------|---------|
| **Docker** | Build the images |
| **eksctl** | Create/delete the EKS cluster itself |
| **kubectl** | Talk to the cluster — the command you'll live in |
| **aws configure** | Credentials, so eksctl/kubectl can reach AWS |

```bash
eksctl create cluster --config-file=eksctl.yaml
aws eks update-kubeconfig --name roboshop --region us-east-1   # writes cluster access config
eksctl delete cluster --config-file=eksctl.yaml                 # clusters cost money — delete them
```

Authentication and authorisation config lands in **`~/.kube/config`** — that file is what makes `kubectl` able to reach *your* cluster.

### On-demand vs spot

Worker nodes are just EC2 instances, so the usual trade-off applies:

| | **On-demand** | **Spot** |
|---|---|---|
| Cost | Full price | **70–90% discount** |
| Reliability | Yours until you stop it | AWS can **take it back with 2 minutes' notice** |
| Use for | **Production** | dev / test workloads |

Spot is the right default for learning and non-prod. Don't put production on it just because it's cheap.

---

## Everything Is a Resource

**Everything in Kubernetes is a resource**, and every resource is configured with YAML in the same four-part shape:

```yaml
apiVersion:      # which API version this resource belongs to
kind:            # the resource type (Pod, Service, Deployment...)
metadata:        # name, namespace, labels, annotations
spec:            # the desired state — what you actually want
```

Learn this shape once and every new resource type is just a new `kind` with a different `spec`.

```bash
kubectl apply -f 01-namespace.yaml     # create/update from YAML — declarative
kubectl get namespace                   # list
kubectl describe namespace roboshop     # human-readable detail + events
kubectl get namespace roboshop -o yaml  # full YAML as the cluster sees it
kubectl delete -f 01-namespace.yaml     # remove
```

`apply`, `get`, `describe`, `delete` work on **every** resource type. The nouns change; the verbs don't.

### Namespaced vs cluster-scoped

Resources come in two scopes:

| Scope | `NAMESPACED` | Meaning |
|-------|--------------|---------|
| **Namespace level** | `true` | Lives inside a namespace (Pod, Service, Deployment, ConfigMap, Secret) |
| **Cluster level** | `false` | Belongs to the whole cluster (Namespace, Node, PersistentVolume) |

```bash
kubectl api-resources        # the NAMESPACED column tells you which is which
```

The AWS parallel is exact — some things are scoped to a VPC, some aren't:

```
SG          → VPC-scoped        ≈ namespaced resource
R53         → not VPC-scoped    ≈ cluster-scoped resource
CloudFront  → not VPC-scoped    ≈ cluster-scoped resource
```

---

## Namespace

A **namespace** is an **isolated project space** to create your resources in — `roboshop`, `expense`, `amazon`, `hdfc`, `flipkart`. One cluster, many projects, no collisions.

```yaml
# k8-resources/01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: roboshop
  labels:
    environment: dev
    project: roboshop
    purpose: poc
```

Every roboshop resource then declares `namespace: roboshop` in its metadata. Switching namespaces constantly gets old fast — **kubens** (from the `kubectx` project) sets your default so you can stop typing `-n roboshop`.

---

## Pod

A **pod is the smallest deployable unit** in Kubernetes. You don't run containers directly — you run pods.

A pod contains **one or more containers**, and the containers inside a pod **share the same network space and storage**. Same network space means they reach each other on `localhost`.

```yaml
# k8-resources/02-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx        # pod name
  labels:
    project: roboshop
    environment: dev
    purpose: poc
spec:
  containers:
  - name: nginx      # container name
    image: nginx     # by default pulled from Docker Hub
```

### Multi-container pods

```yaml
# k8-resources/03-multi-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx
  - name: almalinux
    image: almalinux:9
    command: ["sleep", "1000"]     # overrides the image's CMD
```

Note `command:` — that's the Kubernetes way to override a container's `CMD`, same idea as passing a command to `docker run`.

### Debugging pods

Two failure states you'll hit constantly, and what they actually mean:

| Status | Meaning | Usual cause |
|--------|---------|-------------|
| **ImagePullBackOff** / **ErrImagePull** | The node **can't pull the image** | Authentication issues, or the image address is wrong |
| **CrashLoopBackOff** | The image pulled fine, but the **container won't stay up** | Check the container's command — it's exiting immediately |

`CrashLoopBackOff` is the Kubernetes echo of a Docker lesson: a container needs a foreground process that runs indefinitely, or it exits and Kubernetes restarts it, forever.

```bash
kubectl exec -it nginx -- bash    # shell into a pod (note the -- separator)
kubectl logs <pod>
kubectl describe pod <pod>        # the Events section at the bottom is where the answer usually is
```

---

## Labels and Annotations

Both attach metadata to a resource, but they exist for **opposite audiences**:

| | **Labels** | **Annotations** |
|---|---|---|
| Audience | **Kubernetes itself** | **Systems outside Kubernetes** |
| Purpose | **Selectors** — how resources find each other | Informational — URLs, build info, tooling hints |
| Special characters | **Not allowed** | Allowed |
| Max size | **63 characters** | **256 KB** |

```yaml
# k8-resources/04-labels.yaml
metadata:
  name: labels-demo
  labels:
    project: roboshop
    environment: dev
    purpose: poc
```

```yaml
# k8-resources/05-annotations.yaml
metadata:
  name: annotations
  labels:
    project: roboshop
  annotations:
    imageregistry: "https://hub.docker.com/"
    jenkins-build-url: "https://jenkins.joindevops.com/build/roboshop/job/2"
```

> **Labels are the load-bearing concept.** Services find pods by label. ReplicaSets find pods by label. Deployments find pods by label. If a Service ever selects nothing, a label mismatch is the first thing to check.

---

## Resources — Requests and Limits

Containers consume resources **dynamically**. That's a feature, but with no ceiling a container keeps taking more and more until the **other containers on that host can't get any**. One greedy pod starves its neighbours.

**So always limit resources on your containers.** Two knobs, for CPU and memory:

| | | Meaning |
|---|---|---|
| **`requests`** | **soft limit** | The guaranteed minimum. The scheduler uses this to decide which node the pod fits on. |
| **`limits`** | **hard limit** | The ceiling. The container can never exceed it. |

```yaml
# k8-resources/06-resources.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:            # soft limit
        cpu: "100m"        # 1000m = 1 CPU
        memory: "128Mi"
      limits:              # hard limit
        cpu: "150m"
        memory: "256Mi"
```

Units worth knowing: CPU is measured in **millicores** (`1000m` = 1 full CPU), memory in `Mi`/`Gi`.

---

## Configuration: ENV, ConfigMap, Secret

Recall the Docker best practice — **never bake configuration into the image**. The same image must run in dev, stage, and prod. Kubernetes gives three ways to inject it.

### 1. Inline `env`

Fine for one-offs, but it's config trapped in the pod definition:

```yaml
# k8-resources/07-env.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: project
      value: "roboshop"
    - name: course
      value: kubernetes
```

### 2. ConfigMap

A **ConfigMap** holds application configuration as **key-value pairs**, separate from the pod:

```yaml
# k8-resources/08-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  project: roboshop
  course: kubernetes
  trainer: sivakumar
```

Inject the whole thing with `envFrom` — every key becomes an environment variable:

```yaml
# k8-resources/09-pod-config.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name: nginx-config
```

This is exactly how roboshop wires up its services:

```yaml
# k8-roboshop/catalogue/manifest.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogue
  namespace: roboshop
data:
  MONGO: "true"
  MONGO_URL: "mongodb://mongodb:27017/catalogue"
```

### 3. Secret

A **Secret** is for sensitive values. Same shape as a ConfigMap, but the values are **base64 encoded**:

```yaml
# k8-resources/10-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  username: YWRtaW4K
  password: YWRtaW4xMjMK
```

```yaml
# k8-resources/11-pod-secret.yaml
    envFrom:
    - secretRef:
        name: nginx-secret
```

### Encoding is not encryption

**A Kubernetes Secret is not confidential.** This is the single most misunderstood thing about them.

| | **Encoding** | **Encryption** |
|---|---|---|
| Reversible by anyone | **Yes** — no key needed | No — needs the key |
| Purpose | Safe *transport* of data | **Secrecy** |
| Example | `sivakumar` → `saiavaaakauamara` | `sivakumar` → `gfdhgfdjg95q095745jkdgfhaf49354390` |

Base64 is just a **reversible transformation** — anyone can undo it in one command:

```bash
echo YWRtaW4K | base64 -d      # → admin
```

So a Secret keeps a password out of *casual sight* in a YAML file. It does **not** protect it. Anyone who can read the Secret can read the password. Real protection means an external secrets manager (AWS Secrets Manager, Vault) plus RBAC and encryption at rest.

> This is why roboshop's `mysql` Secret and plain-text `MYSQL_ROOT_PASSWORD` are fine for learning and **not** fine for production.

---

## Services

### Why services exist

**Pods are ephemeral.** They get created and destroyed constantly, and each new pod gets a new IP. So you **can't use a pod IP as an address** — it won't be there tomorrow.

A **Service** is the stable front door. It solves two problems at once:

1. **Pod-to-pod communication** — a stable name that always resolves
2. **Load balancing** — spreads traffic across all matching pods

```
pod  →  service  →  endpoints (podip:container-port)
```

The Service tracks its pods **by label selector**, and maintains the live list of matching pods as its **endpoints**. Pods come and go; the Service name doesn't.

```yaml
# k8-resources/12-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:              # the Service finds pods with THESE labels
    project: roboshop
    tier: frontend
    component: frontend
  ports:
    - protocol: TCP
      port: 80           # service port
      targetPort: 80     # container port
```

This is why roboshop's nginx config can say `proxy_pass http://catalogue:8080/` — `catalogue` is a Service name, and it resolves no matter which pod is alive behind it.

### The three service types

| Type | Reachable from | How |
|------|---------------|-----|
| **ClusterIP** | **Inside the cluster only** (default) | Internal virtual IP + DNS name |
| **NodePort** | **Outside** — the internet | Opens a port on **every worker node** |
| **LoadBalancer** | **Outside** — properly | Provisions a real cloud load balancer |

#### ClusterIP

The default. Internal only — perfect for backend services and databases that should never be publicly reachable. Every roboshop backend (`catalogue`, `user`, `cart`, `mongodb`, `mysql`) uses it.

#### NodePort

Exposes the pod to the outside world:

```
http://<ec2-ip>:<nodePort>  →  ClusterIP  →  pod
```

Kubernetes opens the **same ephemeral port on all worker nodes** — hit *any* node on that port and traffic is forwarded to the ClusterIP and on to a pod. The allowed range is **30000–32767**.

```yaml
# k8-resources/13-service-np.yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

#### LoadBalancer

Provisions a real cloud load balancer in front of the service — the production way to expose something:

```yaml
# k8-resources/14-service-lb.yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
```

Each type builds on the one before: **LoadBalancer → NodePort → ClusterIP → pod**. In roboshop only `frontend` is a LoadBalancer; everything behind it is ClusterIP.

---

## Sets — Managing Pods at Scale

A bare pod is a pet: if it dies, it's gone. Nobody runs bare pods in production. Four controllers manage pods for you:

| Kind | Purpose |
|------|---------|
| **ReplicaSet** | Keep N identical pods running |
| **Deployment** | ReplicaSet + version/rollout management ← **the one you use** |
| **StatefulSet** | Stateful apps needing stable identity and storage (databases) |
| **DaemonSet** | Exactly one pod **per node** (log collectors, monitoring agents) |

### ReplicaSet

**A pod is a subset of a ReplicaSet.** Its one job: make sure the **desired number of pods is running at all times**. Kill a pod and it's replaced immediately.

Pods it creates are named `<replicaset-name>-<random-chars>`.

```yaml
# k8-resources/15-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 10
  selector:
    matchLabels:              # how the RS finds its pods
      project: roboshop
      tier: frontend
      component: frontend
  template:                   # the pod definition it stamps out
    metadata:
      labels:                 # MUST match the selector above
        project: roboshop
        tier: frontend
        component: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:trixie-otel
```

**The catch:** a ReplicaSet **does not care about image version changes**. Change the image and nothing happens — it sees N pods running and is satisfied. That limitation is the entire reason Deployments exist.

### Deployment

Think about what a release actually meant on a traditional server:

```
1. remove old code
2. download new code
3. restart the server
```

That doesn't work here, because **pods and containers are immutable**. You never change a running container. Instead:

```
1. build a new version of the image
2. change the image version in the pod definition
3. apply it
```

A **Deployment** manages that transition. The hierarchy:

```
Deployment  →  creates ReplicaSet  →  creates Pods
```

```yaml
# k8-resources/16-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      project: roboshop
      tier: frontend
      component: frontend
      purpose: deployment
  template:
    metadata:
      labels:
        project: roboshop
        tier: frontend
        component: frontend
        purpose: deployment
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### Rolling update

The default strategy, and the payoff for using a Deployment. Going from **v1 → v2** with 4 pods running:

```
create 1 new pod (v2)  →  delete 1 old pod (v1)
create 2nd new pod     →  delete 1 old pod
create 3rd new pod     →  delete 1 old pod
create 4th new pod     →  delete 1 old pod
```

One in, one out — so there's **never a moment with zero pods serving traffic**. Zero downtime, and if the new version is broken you still have old pods up while you roll back.

```bash
kubectl rollout status deployment/frontend -n roboshop
kubectl rollout undo deployment/frontend -n roboshop      # back to the previous ReplicaSet
```

This is where `latest` bites you again: rolling back means pointing at a *specific* previous version. If everything is `latest`, there's nothing to roll back **to** — which is why roboshop pins `joindevops/catalogue:4.0.0`.

---

## ConfigMaps as Files, and Volumes

A ConfigMap holds key-value pairs — but **a file is just a key whose value is the file's contents**. That's how you get a config file into a pod without rebuilding the image.

**The problem it solves:** `nginx.conf` lives inside the frontend image. If it changes, you'd have to rebuild the image, push it, and update the manifest's image version — a full release cycle for a config tweak.

Instead, put the file in a ConfigMap and **mount it as a volume**:

```yaml
# k8-roboshop/frontend/manifest.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx
  namespace: roboshop
data:
  nginx.conf: |              # key = filename, value = the whole file
    user nginx;
    worker_processes auto;
    ...
    location /api/catalogue/ { proxy_pass http://catalogue:8080/; }
    location /api/user/ { proxy_pass http://user:8080/; }
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: frontend
        image: joindevops/frontend:4.0.0
        volumeMounts:                        # container level: where it appears
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf                # mount just this file, not the whole dir
          readOnly: true
      volumes:                               # pod level: attaching the "disk" to the pod
      - name: nginx-config
        configMap:
          name: nginx
          items:
          - key: nginx.conf
            path: nginx.conf
```

Two levels, and the distinction matters:

| | Where | Meaning |
|---|---|---|
| **`volumes`** | **Pod** level | Attach the storage to the pod — like adding a disk |
| **`volumeMounts`** | **Container** level | Where that storage appears inside *this* container |

The Docker parallel is direct:

```
Docker:      volumes: [mongodb]  +  volumeMounts: mongodb:/data/db
Kubernetes:  volumes: (pod)      +  volumeMounts: (container)
```

`subPath` is the detail worth remembering — without it, mounting into `/etc/nginx/` would replace the **entire directory**. `subPath` places just the one file.

---

## Health Checks — Probes

Kubernetes can't know whether your app is *actually working* — a running process isn't the same as a healthy app. **Probes** are how you tell it. Three of them, answering three different questions:

| Probe | Question | Runs | On failure |
|-------|----------|------|-----------|
| **startupProbe** | *Has the container finished booting?* | **Once**, at startup | Keeps waiting; other probes don't start until this passes |
| **readinessProbe** | *Is the container ready to accept traffic?* | **Continuously** | **Removes the pod from the Service endpoints** — no traffic, but no restart |
| **livenessProbe** | *Is the application still alive?* | **Continuously** | **Restarts** the pod |

The distinction that matters:

- **Readiness failing** → "don't send it traffic right now" (it might be busy or warming up). Stop routing, wait.
- **Liveness failing** → "it's wedged and won't recover." Long-running apps get stuck holding locks; restarting is the only fix.

**startupProbe** exists so slow-booting apps don't get killed by a liveness probe before they've even started.

```yaml
# k8-roboshop/payment/manifest.yaml
containers:
- name: payment
  image: joindevops/payment:4.0.0
  ports:
  - name: liveness-port
    containerPort: 8080
  startupProbe:
    httpGet:
      path: /health
      port: liveness-port
    failureThreshold: 12        # 12 × 10s = 5 minutes to boot before giving up
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /health
      port: liveness-port
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /health
      port: liveness-port
    periodSeconds: 10
```

`failureThreshold: 12` × `periodSeconds: 10` = **the app gets 5 minutes to start**. That's the startup probe's whole job: buy slow starters enough time.

The app needs an endpoint to answer — roboshop's nginx config defines one:

```nginx
location /health {
  stub_status on;
  access_log off;
}
```

---

## The Standard Resource Set

Putting it together — what a real service in roboshop is made of:

| # | Resource | Why |
|---|----------|-----|
| 1 | **Namespace** | `roboshop` — the project's isolated space |
| 2 | **Deployment** | **Not** a bare Pod, **not** a ReplicaSet — you want rollouts |
| 3 | **ConfigMap** | Non-sensitive configuration |
| 4 | **Secret** | Sensitive values |
| 5 | **Service** | ClusterIP for internal, LoadBalancer to expose |

That's the pattern every `k8-roboshop/*/manifest.yaml` follows. The shape of the app in Kubernetes:

```
                      LoadBalancer Service
                              │
                          frontend  (nginx + configMap-mounted nginx.conf)
                              │  proxy_pass
        ┌──────────┬──────────┼──────────┬──────────┐
    catalogue    user       cart     shipping   payment      ← ClusterIP services
        │          │          │          │          │
     mongodb   mongodb     redis      mysql    rabbitmq      ← ClusterIP services
                +redis
```

Compare it to the Compose file from the Docker sessions and it's the same application — the same names, the same DNS-by-name wiring, the same dependencies. What changed is that Kubernetes now handles the scaling, the health, the rollouts, and the load balancing that Compose couldn't.

---

## Storage — Volumes, PV & PVC

A ConfigMap-as-volume solves *config*. Real **data** — a database's files, uploaded images, logs — is a different problem, and it's the one interviewers push on. Start from the pain:

> **Everything is ephemeral, all the way down.** A pod dies → its data dies. But it's worse than that: on EKS the **nodes themselves are ephemeral** (spot reclaim, autoscaling, upgrades all replace them). So "just store it on the node" doesn't save you either — the node is as disposable as the pod. Durable data has to live **outside the cluster**, on real cloud storage.

### First, know your AWS storage — this *is* an interview question

You can't reason about Kubernetes storage without knowing what's underneath it. Three services, and *when* you'd pick each:

| | **EBS** | **EFS** | **S3** |
|---|---|---|---|
| What it is | **Block** storage — a raw disk | **File** storage — a shared filesystem (NFS) | **Object** storage |
| Mental model | A hard disk you plug in | A network drive everyone maps | A bucket reached over HTTP(S) |
| Attach to how many instances | **One at a time** | **Many at once** | n/a — accessed over HTTP/HTTPS |
| Size | **Fixed** — you provision N GB | **Grows automatically** | Effectively infinite |
| Speed | **Fastest** | Slower | Not a filesystem |
| Location constraint | **Same AZ** as the instance | Anywhere in the network | Region |
| Security group | Not needed | **Required** | n/a |
| Use it for | **OS disks, databases** | Shared *normal* files across many pods | Backups, artifacts, static assets |

The one-liners that make it stick:

- **EBS is like a hard disk; EFS is like a shared/mapped network drive.**
- **Block vs file:** block storage stores data as raw 4 KB blocks on a disk — a *filesystem* sits on top to turn those blocks into files (CRUD on blocks). That's why EBS is a three-step ritual: **format the disk → create a filesystem → mount it.** EFS skips all that — AWS already fixed the filesystem as NFS, so your only step is **mount**.
- **Databases → EBS, always.** EFS can't back a database (latency + file-locking semantics). Normal files that many pods must share → EFS.

### Two families of Kubernetes volume

```
Volumes
├── Ephemeral   → live and die with the pod   (emptyDir, hostPath)
└── Persistent  → outlive the pod             (PV + PVC on EBS/EFS)
```

Same `volumes` (pod level) + `volumeMounts` (container level) wiring you already know — what changes is what sits behind the volume.

### Ephemeral: `emptyDir`

An **empty directory created when the pod is scheduled, deleted when the pod dies.** It's not for keeping data — it's **scratch space shared between containers in the same pod** (remember: containers in a pod share storage).

The classic use is the **sidecar logging pattern**: the app writes logs to the shared dir, a second container reads them and ships them to ELK.

```yaml
# k8-resources/volumes/01-emptyDir.yaml
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /var/log/nginx        # app writes logs here
      name: nginx-logs
  - name: almalinux                     # sidecar
    image: almalinux:9
    command: ["sleep", "1000"]
    volumeMounts:
    - mountPath: /mnt/nginx-logs        # sidecar reads the SAME volume
      name: nginx-logs
      readOnly: true                    # sidecar only reads — good hygiene
  volumes:
  - name: nginx-logs
    emptyDir:                           # empty dir on the node, gone when the pod is
      sizeLimit: 500Mi
```

Two containers, one volume, mounted at *different paths* — that's the whole point of `emptyDir`.

### Ephemeral: `hostPath` — powerful and dangerous

**`hostPath` mounts a directory from the node itself into the pod.** Sounds useful; it's a trap for application data, for one blunt reason:

> A pod is **not pinned to a node**. Delete it and it may reschedule onto a **different** node — where your `hostPath` data doesn't exist. Your data is stranded on the old node.

So `hostPath` is **never for app data.** Its legitimate use is the mirror image: not putting data *in*, but reading the **node's own files out** — shipping **host** logs/metrics to ELK. And you do it **`readOnly: true`** as a guardrail, because giving a pod write access to the host filesystem is a serious security hole.

This is where **DaemonSet** clicks into place — the fourth "set" from earlier:

> **DaemonSet = exactly one pod on every node.** Pair it with `hostPath` and you have an agent on each node reading that node's `/var/log` and shipping it out. That's *the* canonical log/metrics-collector pattern.

```yaml
# k8-resources/volumes/02-host-path.yaml
apiVersion: apps/v1
kind: DaemonSet                          # one per node
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system                 # an ADMIN namespace, not yours
spec:
  template:
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true                 # read the host's logs, never write
      volumes:
      - name: varlog
        hostPath:
          path: /var/log                 # the node's own log directory
```

Note the namespace: `kube-system`. **`hostPath` is an administrator activity**, not something app teams do in their own namespace — it reaches outside the Kubernetes abstraction and touches the host, which is a cluster-admin concern.

### Persistent storage: the org chart is the real lesson

Here's the mental model interviews are actually testing. In an on-prem cluster, getting a disk is a **process**, not a command:

```
1. raise a ticket, get manager approval
2. storage team gets the ticket — you state size, reason
3. their lead approves, a member provisions the disk, hands you the address
4. you email the admin team to wire it in
```

Kubernetes admins don't know storage internals, and storage teams don't know Kubernetes. So Kubernetes put **wrapper objects on top of the raw disk** to divide the responsibility cleanly — **PV, PVC, SC**:

| Object | Full name | Who owns it | Analogy |
|--------|-----------|-------------|---------|
| **PV** | PersistentVolume | **Storage/cluster admin** | The **wallet** (the actual disk) |
| **PVC** | PersistentVolumeClaim | **App developer** | The **request** for spending money |
| **SC** | StorageClass | Admin (enables automation) | The rule that **creates wallets on demand** |

The family chain from the notes nails the separation of concerns:

```
son  →  mother  →  father  →  wallet
Pod  →   PVC    →   PV     →  storage
```

The **son (Pod) never touches the wallet (disk) directly.** He asks his mother (PVC), who goes to the father (PV), who holds the wallet (real EBS/EFS). Each layer only talks to its neighbour — so the app developer writes a **PVC** ("I need 2Gi, read-write") and never has to know the volume ID, the AZ, or the CSI driver. That decoupling *is* the reason PV/PVC exist. If someone asks "why not mount the disk straight into the pod?" — this is the answer: **it separates the person who needs storage from the person who provisions it.**

- **PV** = the logical Kubernetes representation of a real disk. The physical disk, as a K8s object.
- **PVC** = a claim against a PV — "give me this much, with these access rights." The pod references the *claim*, never the PV.

### Static vs dynamic provisioning

| | **Static** | **Dynamic** |
|---|---|---|
| Who creates the disk | **You**, manually (create the EBS volume, then the PV) | **Kubernetes**, automatically via a **StorageClass** |
| When | Ahead of time | The moment a PVC asks |
| Real-world use | Rare / learning | The norm in production |

The example below is **static** — the disk already exists (`volumeHandle: vol-...`) and you're describing it to Kubernetes by hand. Getting there needs plumbing worth naming because it's classic interview/debug territory:

1. Install the **EBS CSI driver** (the thing that lets K8s attach EBS)
2. The **EKS nodes need IAM permission** (`EBSCSIDriverPolicy`) to attach disks — miss this and volumes hang in `Pending`
3. Create the EBS volume **in the same AZ as a node** (EBS is AZ-locked)
4. Write the PV, then the PVC, then reference the PVC in the pod

```yaml
# k8-resources/volumes/03-ebs-static.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-static
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 2Gi
  csi:
    driver: ebs.csi.aws.com              # the EBS CSI driver
    fsType: ext4
    volumeHandle: vol-0298e8f2bbbde0f28  # the pre-created EBS volume (static)
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-static
spec:
  storageClassName: ""                   # "" = don't use the default SC; bind to MY PV
  volumeName: ebs-static                 # explicitly bind to the PV above
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-static
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-static              # the pod references the CLAIM, not the PV
  nodeSelector:
    topology.kubernetes.io/zone: us-east-1d   # pin the pod to the disk's AZ
```

Two details that trip people up:

- **`storageClassName: ""`** is deliberate. An empty string means "do **not** fall back to the default StorageClass" — you want this PVC to bind to *your* named PV, not have K8s dynamically provision a new disk.
- **`nodeSelector` on the AZ.** EBS is AZ-locked, so the pod *must* land in the same AZ as the disk (`us-east-1d`) or it can never mount it. This is the single most common EBS-on-K8s failure.

### Access modes — asked constantly

Who can mount the volume, and how. Directly reflects EBS-vs-EFS reality (EBS = one node; EFS = many):

| Mode | Meaning |
|------|---------|
| **ReadWriteOnce** (RWO) | One **node** mounts read-write; pods *on that node* can use it → typical **EBS** |
| **ReadOnlyMany** (ROX) | Many **nodes** mount, read-only |
| **ReadWriteMany** (RWX) | Many **nodes** mount read-write → needs **EFS** (EBS can't do this) |
| **ReadWriteOncePod** | Exactly **one pod** read-write — stricter than RWO (node-wide) |

Interview trap: RWO is *once per **node**,* not per pod — several pods on the same node can share an RWO volume. If you truly need one-pod exclusivity, that's **ReadWriteOncePod**.

### Reclaim policy — what happens to the data when the PVC is deleted

| Policy | PV object | The actual data |
|--------|-----------|-----------------|
| **Delete** | Removed | **Destroyed** — the underlying disk is deleted too |
| **Retain** | Kept | **Preserved** — data survives even after PVC/pod are gone (you clean up manually) |
| **Recycle** | Kept | **Wiped** but the disk stays (deprecated) |

The one that matters in production: **use `Retain` for anything you can't afford to lose.** `Delete` is convenient and will happily take your database down with the PVC. This is a real blast-radius question — "what happens to the data if someone `kubectl delete pvc`?" — and the honest answer is "depends on the reclaim policy," which is exactly the point.

### Where the scheduler comes in

The **scheduler** decides which node a pod runs on, weighing many factors — and storage is one of them. That EBS `nodeSelector` above is you *overriding* the scheduler to force the pod into the disk's AZ. Worth knowing the scheduler is steerable (`nodeSelector`, affinity, taints/tolerations) precisely because AZ-locked storage sometimes forces your hand.

---

## Dynamic Provisioning — StorageClass

Static provisioning has an obvious problem: **a human has to create a disk before anyone can use it.** That doesn't scale, and it's why static is rare in production.

A **StorageClass** is the fix — it's a **recipe for creating storage on demand**. Point a PVC at a StorageClass and Kubernetes provisions **both the real disk and its PV automatically**. You typically keep one StorageClass per storage type: one for EBS, one for EFS.

The analogy extends neatly:

```
static   →  son  →  mother  →  father  →  wallet          (a physical wallet handed over)
dynamic  →  son  →  mother  →  PhonePe wallet             (money appears on demand — no father, no physical wallet)
```

With dynamic provisioning the **PV layer stops being a person's job.** The developer writes a PVC; nobody files a ticket.

### The two workflows side by side

| Step | **EBS static** | **EBS dynamic** |
|------|----------------|-----------------|
| 1 | Install EBS CSI drivers | Install EBS CSI drivers |
| 2 | `EBSCSIDriverPolicy` IAM on worker nodes | `EBSCSIDriverPolicy` IAM on worker nodes |
| 3 | **Create the disk manually** | **Create a StorageClass** |
| 4 | **Create the PV** | — *(created automatically)* |
| 5 | Create the PVC | Create the PVC |
| 6 | Attach the volume in the pod | Attach the volume in the pod |

Two manual steps disappear — and with them, the AZ headache.

### The StorageClass

```yaml
# k8-resources/volumes/04-ebs-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: roboshop-ebs
provisioner: ebs.csi.aws.com          # WHO creates the disk — the EBS CSI driver
reclaimPolicy: Retain                  # keep the data if the PVC is deleted
volumeBindingMode: WaitForFirstConsumer
```

Then the PVC just names it — no PV anywhere in sight:

```yaml
# k8-resources/volumes/05-ebs-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: roboshop-ebs       # "make me one of these"
  resources:
    requests:
      storage: 6Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-dynamic
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-dynamic
```

Compare this to the static example — **no PV, no `volumeHandle`, no `nodeSelector`.** That's the payoff.

### `volumeBindingMode: WaitForFirstConsumer` — the detail worth understanding

This is the single best thing to be able to explain about StorageClasses, because it solves the exact AZ problem that forced a `nodeSelector` in the static example.

Follow the actual sequence:

```
1. you send the pod definition to EKS
2. the SCHEDULER assigns the pod to a node
3. that node pulls the image
4. the pod claims storage through its PVC
5. EKS creates the EBS volume — in the SAME AZ as the node it landed on
```

The disk is created **after** the pod is scheduled, so Kubernetes already knows which AZ to build it in. Storage follows the pod instead of the pod being pinned to storage.

> The alternative (`Immediate`) creates the disk the moment the PVC is made — before any pod exists — so it can easily land in an AZ with no room for your pod, and the pod then hangs `Pending` forever. **For EBS, `WaitForFirstConsumer` is the right default.**

---

## EFS — Shared Storage

EBS gives one node a fast disk. But `ReadWriteMany` — many pods across many nodes writing the same data — **EBS simply cannot do**. That's EFS's job.

### EFS static

```
1. install the EFS CSI drivers
2. add EFSCSIDriverPolicy to the node IAM role
3. create the EFS filesystem — and allow port 2049 in its security group
4. create the PV
5. create the PVC
6. claim it in the pod
```

**Port 2049 is NFS.** Forget that security group rule and everything mounts... nothing — it just hangs. This is the EFS equivalent of the AZ mistake, and it's the first thing to check when an EFS mount stalls. (Recall from the storage table: **EFS needs a security group; EBS doesn't.**)

```yaml
# k8-resources/volumes/06-efs-static.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-static
spec:
  # capacity is DUMMY for EFS — the API demands a number, but EFS grows on its own
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany                    # the whole point of EFS
  storageClassName: ""                 # static → no StorageClass
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: efs:fs-042342d611e8847c4   # the EFS filesystem ID
```

Two things to notice, both common interview fodder:

- **`capacity: 5Gi` is fiction.** EFS grows automatically — the field exists only because the API requires it. Contrast with EBS, where the size is real and fixed.
- **`accessModes: ReadWriteMany`** — this is what you came to EFS for.

### EFS dynamic

Same driver + IAM + port-2049 setup, then a StorageClass instead of a hand-written PV:

```yaml
# k8-resources/volumes/07-efs-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap             # EFS Access Point — an isolated dir per claim
  fileSystemId: fs-05ce26454cd884a57   # the EFS filesystem to carve up
  directoryPerms: "700"
  basePath: "/roboshop"                # optional — where the sub-directories live
```

The mental model differs from EBS in an important way:

> **EBS dynamic creates a new disk per claim. EFS dynamic does *not* create a new filesystem** — it creates an **access point**, a permission-scoped sub-directory inside the *one* EFS filesystem you already made. That's why the StorageClass needs an existing `fileSystemId`.

```yaml
# k8-resources/volumes/08-efs-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-dynamic
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-dynamic
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-dynamic
```

### The rule to remember

> **EBS for databases. EFS for file storage.**

EFS is a network filesystem — the latency and file-locking behaviour that make it great for shared files make it wrong for a database's data directory. If someone asks "can I run MySQL on EFS?", the answer is *don't*.

---

## StatefulSet

### First: why shared storage breaks a database cluster

This is the argument that justifies the whole resource, and it's worth being able to reason through out loud.

Picture a **3-node database cluster sharing one disk.** A write arrives at NODE-A, which starts modifying the disk. To stop NODE-B corrupting the same data at the same time, you must **lock** the other nodes.

> And there's the contradiction: **if NODE-B has to wait for NODE-A, there is no point having a cluster.** You've bought three servers to get the throughput of one.

The fix is the opposite arrangement — **give every node its own storage.** No locking, no waiting. When NODE-A takes a write, it sends an **asynchronous replication** request to its peers, and they apply it to their own disks.

So for stateful applications on Kubernetes, **every pod must have its own storage.** A Deployment can't express that, so Kubernetes has a separate resource: the **StatefulSet**.

### What a StatefulSet gives you

| # | Property | Why it matters |
|---|----------|----------------|
| 1 | **Own PV/PVC per pod** | Each replica gets its own disk — no locking, no corruption |
| 2 | **Needs a headless Service** (plus a normal one) | So pods can discover their **peers** to replicate to |
| 3 | **Stable, predictable pod names** | `mysql-0`, `mysql-1` — not a random suffix |
| 4 | **Ordered creation and deletion** | Pods come up `-0`, `-1`, `-2` and are removed in reverse |

Points 3 and 4 are what "stateful identity" actually means. `mysql-0` is *always* `mysql-0`, always reattaches to `mysql-0`'s disk, and always starts before `mysql-1`. Databases need that determinism to form a cluster reliably.

### Headless Service — the peer-discovery mechanism

A common interview question: **what is a headless service, and why does a StatefulSet need one?**

**A Service with `clusterIP: None` is a headless service.** The difference is entirely in what DNS returns:

| | **Normal Service** | **Headless Service** (`clusterIP: None`) |
|---|---|---|
| `nslookup <svc-name>` returns | **One** IP — the service's ClusterIP | **All the endpoints** (every pod IP) behind it |
| Traffic | Load-balanced to one pod | You talk to pods **directly** |
| Used for | Clients that just need *any* healthy pod | Pods that need to find **every peer** |

A normal Service is a front door — it deliberately hides which pod you reach, which is exactly right for `catalogue` or `frontend`. But a database node replicating data doesn't want *any* pod; it needs **all of them**. So it does an `nslookup` on the headless service and gets back **every endpoint in the cluster**, then opens connections to its peers.

That's why every roboshop database declares **two** services:

```yaml
# k8-roboshop-v2/mongodb/manifest.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless          # for peer discovery between pods
  namespace: roboshop
spec:
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
  clusterIP: None                 # ← headless
  selector:
    project: roboshop
    component: mongodb
    tier: db
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb                   # normal ClusterIP — what catalogue/user connect to
  namespace: roboshop
spec:
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
  selector:
    project: roboshop
    component: mongodb
    tier: db
```

The application services keep using the normal one (`MONGO_URL: "mongodb://mongodb:27017/catalogue"`); the database pods use the headless one among themselves.

### `volumeClaimTemplates` — a PVC per pod

The key structural difference from a Deployment. Instead of one `volumes:` block shared by every replica, a StatefulSet has a **`volumeClaimTemplates`** — a *template* Kubernetes stamps into a **separate PVC for each pod**.

```yaml
# k8-roboshop-v2/mongodb/manifest.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: roboshop
spec:
  selector:
    matchLabels:
      project: roboshop
      component: mongodb
      tier: db
  serviceName: "mongodb-headless"      # ← points at the HEADLESS service
  replicas: 1
  template:
    metadata:
      labels:
        project: roboshop
        component: mongodb
        tier: db
    spec:
      containers:
      - name: mongodb
        image: joindevops/mongodb:4.0.0
        volumeMounts:
        - name: mongodb
          mountPath: /data/db              # mongo's data directory
  # PVC template — one disk created PER POD
  volumeClaimTemplates:
  - metadata:
      name: mongodb
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "roboshop-ebs"     # the dynamic EBS StorageClass
      resources:
        requests:
          storage: 1Gi
```

Three details worth pointing at:

- **`serviceName: "mongodb-headless"`** — a StatefulSet must be told which headless service governs its pods' DNS identities.
- **`volumeClaimTemplates`** relies on the dynamic **`roboshop-ebs`** StorageClass from the previous session. Scale to 3 replicas and Kubernetes creates **three** EBS volumes, one per pod, automatically. This is dynamic provisioning paying off.
- **`ReadWriteOnce`** is correct here precisely *because* each pod has its own disk — no sharing means no need for `ReadWriteMany`.

Every roboshop database follows this identical shape — `mongodb` (`/data/db`), `mysql` (`/var/lib/mysql`), `redis` (`/data`), and `rabbitmq` — each with a headless service, a normal service, a StatefulSet, and a `volumeClaimTemplates`.

### Deployment vs StatefulSet

| | **Deployment** | **StatefulSet** |
|---|---|---|
| Built for | **Stateless** applications | **Stateful** applications |
| Storage | All replicas share the **same** volume | **Each pod gets its own** volume (`volumeClaimTemplates`) |
| Pod names | Random suffix (`catalogue-7d9f8b-xk2p9`) | Stable ordinal (`mongodb-0`, `mongodb-1`) |
| Pod identity | Interchangeable — any pod is as good as another | **Sticky** — a pod always reattaches to its own disk |
| Creation / deletion | All at once, any order | **Ordered** — `-0`, then `-1`, then `-2`; deleted in reverse |
| Service needed | A normal Service | A **headless** Service *and* usually a normal one |
| Roboshop examples | frontend, catalogue, user, cart, shipping, payment | mongodb, mysql, redis, rabbitmq |

This closes a loop opened at the very start of the Docker notes: *stateless apps are easy to containerise; stateful ones need special care.* StatefulSet is the shape that care takes in Kubernetes.

---

## Scaling and the Horizontal Pod Autoscaler

### Horizontal vs vertical scaling

| | **Vertical scaling** | **Horizontal scaling** |
|---|---|---|
| What you do | Make the **existing** server bigger — more CPU/RAM/disk | Add **another** server to share the load |
| Downtime | **Yes** — you must resize and restart it | **None** |
| Single point of failure | **Yes** — still one machine | **No** — many machines |
| Ceiling | Limited by the biggest machine available | Effectively unlimited |

Vertical scaling has a hard ceiling and a restart; horizontal scaling doesn't. **Kubernetes is built around horizontal scaling** — that's what `replicas` means, and the autoscaler automates it.

### HorizontalPodAutoscaler

An **HPA** watches a metric (usually CPU) and **adds or removes pod replicas automatically** to keep that metric near a target.

There's a hard prerequisite, and it's the usual reason an HPA sits there doing nothing:

> **A pod consumes resources dynamically.** The HPA measures utilisation as a **percentage of the pod's `requests`** — so if a container has no `requests` set, there's no denominator and the HPA has nothing to compute. **Resource requests/limits are mandatory for autoscaling.**

That's why the v2 manifests add a `resources` block alongside the HPA:

```yaml
# k8-roboshop-v2/catalogue/manifest.yaml
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalogue
spec:
  scaleTargetRef:                  # WHAT to scale
    apiVersion: apps/v1
    kind: Deployment
    name: catalogue
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50     # keep average CPU at ~50% of requests
```

How to read it: *"keep `catalogue` between 1 and 10 pods, adding replicas whenever average CPU across the pods exceeds **50% of the requested 100m**, and removing them when it falls below."* Utilisation is a plain percentage — **90% means 90 out of 100**, measured against `requests`, not against the node.

> Note the HPA targets a **Deployment** — stateless services are the ones you autoscale. You generally don't autoscale a database StatefulSet; adding a replica there means joining a cluster and replicating data, which isn't something a CPU threshold should trigger.

### Controlling *where* pods land

Scaling decides **how many** pods; a separate family of settings — `nodeSelector`, taints/tolerations, node affinity, pod affinity — decides **which node** each one goes to. That's a topic in its own right; see [Scheduling — Placing Pods on the Right Nodes](#scheduling--placing-pods-on-the-right-nodes) below.

### k9s

A terminal UI for the cluster — far faster than typing `kubectl get`/`describe` repeatedly when you're watching pods roll or debugging a StatefulSet come up:

```bash
curl -sS https://webinstall.dev/k9s | bash
```

---

## Scheduling — Placing Pods on the Right Nodes

By default the **scheduler** picks a node for each pod on its own — it looks at which nodes have enough free `requests` and drops the pod on one that fits. Most of the time that's exactly what you want. **Selectors** are how you override it: *controlling the scheduler to run pods on the nodes you choose.* There are four mechanisms, from bluntest to most expressive.

Every node already carries a set of labels you can select on. On EKS they're generated for you — a real node looks like this:

```text
eks.amazonaws.com/capacityType=SPOT
node.kubernetes.io/instance-type=c3.large
topology.kubernetes.io/zone=us-east-1c
topology.kubernetes.io/region=us-east-1
kubernetes.io/hostname=ip-192-168-1-236.ec2.internal
kubernetes.io/arch=amd64
kubernetes.io/os=linux
```

So without doing anything you can already target a zone, an instance type, or spot-vs-on-demand. You can also add your own — `kubectl label node ip-192-168-1-236.ec2.internal project=roboshop` — and select on that.

### 1. nodeSelector — the blunt instrument

A single key/value that a node must have. Put it in the pod spec:

```yaml
spec:
  nodeSelector:
    project: roboshop
```

**It's a hard rule.** If no node carries that exact label, the pod doesn't get placed somewhere close — it sits in **`Pending`** forever. Simple, but one label only and no "prefer" fallback. This is the same tool used earlier to pin a pod to its EBS volume's availability zone.

### 2. Taints and tolerations — the node pushes back

A **taint** is the node saying *"keep out."* (Think of *taint* as **painted / polluted** — the scheduler won't put anything here.) You taint a node when it's reserved for a purpose:

- a node with **special hardware** reserved for special workloads,
- a **GPU** node reserved for AI/ML jobs,
- a node whose DB only accepts connections from specific IPs.

```bash
kubectl taint nodes ip-192-168-1-236.ec2.internal hardware=gpu:NoExecute
```

The effect after the colon decides how aggressive it is:

| Effect | Meaning |
|--------|---------|
| **NoSchedule** | Don't schedule **new** pods here |
| **NoExecute** | Also **evict** pods already running here |

A pod gets past a taint only if it carries a matching **toleration** — the *VIP cabin / VIP pass* pairing. The taint is the roped-off cabin; the toleration is the pass that lets you in.

The subtle part interviewers probe:

> **A toleration only *permits*, it does not *place*.** With 3 nodes and 1 tainted, normal pods see 2 usable nodes. Give a pod a toleration and it now sees all 3 — the tainted one is merely *allowed*, not chosen. So to actually **force** a pod onto a tainted node you need **both**: a **toleration** (to be let in) *and* a **nodeSelector / affinity** (to be pointed there).

### 3. Node affinity — nodeSelector with options

When one label isn't enough — you want *"zone `us-east-1c` **or** `us-east-1a`, and prefer SSD nodes"* — use **nodeAffinity**. Two flavours, and the long names decode cleanly once you split them at "During":

| Rule | During **scheduling** | During **execution** |
|------|----------------------|----------------------|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Won't schedule** unless the node matches (hard) | Label changes later → **ignored**, pod keeps running |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Tries** to match; if nothing matches, schedules anywhere (soft) | Same — ignored once running |

The `IgnoredDuringExecution` half is the key promise: **once a pod is placed and running, relabelling the node has no effect on it.** Kubernetes enforces the rule at *scheduling* time only — it won't yank a happily-running pod because a label drifted. (A stricter `RequiredDuringExecution` was planned but never shipped, which is why every rule name ends in `Ignored…`.)

### 4. Pod affinity / anti-affinity — placement relative to *other* pods

Node affinity targets nodes; **pod affinity targets other pods.** The question changes from *"which node?"* to *"which pods do I want to sit with — or avoid?"*

- **Pod affinity** — *co-locate.* If `pod-a` is on `node-1`, schedule `pod-b` on `node-1` too (e.g. an app pod next to its cache for low latency).
- **Pod anti-affinity** — *spread apart.* If `pod-a` is on `node-1`, keep `pod-b` off it — push it to `node-2` or `node-3`.

Anti-affinity is the one that matters for resilience. Give every replica of `app: store` anti-affinity against its own label and Kubernetes fans them out:

```text
replica-1 → node-1
replica-2 → node-2
replica-3 → node-3
```

Now one node failure costs you **one** replica, not the whole service. This is exactly how you'd spread the pods of a StatefulSet database across nodes so a single node loss can't take down the cluster.

### Choosing between them

| You want to… | Use |
|--------------|-----|
| Pin to one specific label, no fallback | **nodeSelector** |
| Reserve a node so nothing lands there unless invited | **taint** (+ toleration on the guests) |
| Match nodes by several labels, hard *or* soft | **node affinity** |
| Place pods relative to other pods (together / apart) | **pod (anti-)affinity** |

The distinction that ties it together: **taints are the node pushing pods away; affinity is the pod pulling itself toward (or away from) something.** `nodeSelector` and node affinity choose by *node label*; pod affinity chooses by *neighbouring pod*.

---

## Helm — Templating and Packaging Manifests

By now every roboshop service is a folder of static YAML. Two problems show up the moment you run more than one environment:

- **Values are baked into the manifest.** Dev wants 1 replica, prod wants 5 — but `replicas: 5` is hard-coded, so you either edit the file before every deploy or keep two near-identical copies.
- **There's no package to install.** To stand up the EBS CSI driver or Prometheus you'd apply a dozen inter-dependent manifests by hand and hope you got the order right.

**Helm** solves both. It's described two ways, and they're really the same idea from two sides:

1. **A package manager for Kubernetes** — install a whole application (many resources) with one command, like `dnf install nginx` on Linux.
2. **A templating engine for manifests** — keep **placeholders** in the YAML and supply the **values at runtime**.

This mirrors the split we started with: *build the image* (open-source nginx/mongo/node…) → **Docker**; *run the image* (manifests) → Kubernetes; **package and parameterise those manifests** → Helm.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh && ./get_helm.sh
```

### Anatomy of a chart

A **chart** is just a directory with three parts:

| Piece | Role |
|-------|------|
| **`Chart.yaml`** | Chart metadata — `name`, `version`, `apiVersion`, `description`, `appVersion` |
| **`templates/`** | The manifests, with **placeholders** instead of hard-coded values |
| **`values.yaml`** | The **default values** those placeholders resolve to |

The templating is plain `{{ .Values.<path> }}` referring into `values.yaml`. From the teaching `nginx` chart:

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.deployment.replicas }}
  ...
        image: nginx:{{ .Values.deployment.imageVersion }}
# templates/service.yaml
  type: {{ .Values.service.type }}
```

```yaml
# values.yaml — the defaults
deployment:
  replicas: 4
  imageVersion: 1.31.3-alpine
service:
  type: ClusterIP
```

At install time Helm **renders** the templates against the values and hands plain manifests to the cluster. The `roboshop-helm/catalogue` chart does the same for a real backend service — `image: "{{ .Values.deployment.imageRepo }}:{{ .Values.deployment.imageVersion }}"` — so bumping the image is a one-line values change, not a manifest edit.

**`Chart.yaml` carries two versions, and interviewers like the distinction:**

```yaml
apiVersion: v2
name: nginx
version: 0.1.3          # the CHART's version — bump when you change templates/values
appVersion: 1.31.3-alpine  # the APP inside — the image version you're shipping
```

`version` tracks your packaging; `appVersion` tracks the software being packaged. They move independently.

### The payoff: one chart, many environments

This is the reason Helm exists in a CI/CD pipeline. **The image is identical across environments — only the *configuration* differs** — so you keep one set of templates and one values file per environment:

```
values.yaml       # sensible defaults
values-dev.yaml   # deployment.replicas: 1
values-prod.yaml  # deployment.replicas: 5
```

The pipeline then does the same thing in each stage, only swapping the values file:

```bash
# dev stage
#   authenticate to the dev cluster, then:
helm upgrade --install catalogue . -f values-dev.yaml

# prod stage
#   authenticate to the prod cluster, then:
helm upgrade --install catalogue . -f values-prod.yaml
```

`upgrade --install` is the idempotent form — **install if it's the first time, upgrade if it already exists** — so the same command line is safe in every run.

### Lifecycle commands

| Command | Purpose |
|---------|---------|
| `helm install <name> .` | First-time install of the chart in this directory |
| `helm upgrade <name> .` | Apply changes to an already-installed release |
| `helm upgrade --install <name> .` | Do whichever applies — the CI/CD default |
| `helm list` | Show installed releases |
| `helm history <name>` | Revision history of a release (enables rollback) |
| `--set deployment.replicas=10` | Override a single value inline, without editing a file |

### The other face: installing community charts

Authoring your own chart is one half; **consuming** public charts is the other, and it's the exact Linux-package-manager workflow. On a server you add a repo to `/etc/yum.repos.d/` then `dnf install nginx`. With Helm you add a chart repo, then install:

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm upgrade --install ebs-csi-driver \
    --namespace kube-system --version 2.62.0 \
    aws-ebs-csi-driver/aws-ebs-csi-driver
```

That single command is the EBS CSI driver from the storage sessions — dozens of manifests, packaged. The same pattern installs a full monitoring stack:

```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

So Helm is where the course's two threads meet: your own roboshop services become parameterised charts, and the platform pieces you depend on (CSI drivers, Prometheus) arrive as someone else's charts — both installed the same way.

---

## Service Accounts — Giving Pods an AWS Identity

Everything so far has been about a pod talking to **other pods**. But a pod often needs to talk to the **cloud** — a catalogue pod reading a database password out of AWS Secrets Manager, a backup pod writing to S3, a pod describing EBS volumes. That is an authorisation question: *who is this pod, and what is it allowed to do in AWS?* The answer is a **Service Account**.

A ServiceAccount is the **identity a pod runs as inside Kubernetes** — the pod equivalent of a Linux user. It is a namespaced resource, and the rule to remember is:

> Whenever you create a namespace, Kubernetes auto-creates a **`default`** service account in it with **zero permissions**, and every pod that doesn't name a service account runs as that default.

So your pods already have an identity — it just can't do anything. To let a pod perform CRUD on AWS resources, you attach real AWS permissions to that identity. On EKS the mechanism is **IRSA — IAM Roles for Service Accounts** — which lets a Kubernetes service account *become* an AWS IAM role.

### The four steps

The mental model from the session:

1. **Create the service account** (the pod's identity).
2. **Create an IAM role** with the IAM permissions you want to grant.
3. **Map the IAM role to the service account** through an **annotation** on the SA.
4. **Run the pod with that service account** (`serviceAccountName:`).

Step 3 is exactly why annotations exist — recall that annotations are *metadata for external systems*. The IAM role ARN is AWS metadata, so it belongs in an annotation, not a label.

### OIDC — the trust bridge

Before any of this works, the cluster needs an **OIDC provider**. OIDC (OpenID Connect) is a token-based way to **trust a third-party application** — here, AWS trusts tokens issued by your Kubernetes cluster, which is what lets a pod's SA token be exchanged for AWS credentials. You associate it once per cluster:

```bash
eksctl utils associate-iam-oidc-provider --cluster roboshop --approve
```

Without this, AWS has no reason to believe the token a pod presents actually came from your cluster.

### Creating the service account with its permissions

`eksctl` does steps 1–3 in a single command — it creates the SA, creates the IAM role, and wires them together with the annotation:

```bash
eksctl create iamserviceaccount \
  --cluster=roboshop \
  --namespace=roboshop \
  --name=roboshop-secret-reader \
  --attach-policy-arn=arn:aws:iam::160885265516:policy/RoboShopMySQLSecretReader \
  --approve
```

That says: in the `roboshop` namespace, make a service account called `roboshop-secret-reader`, and behind it stand an IAM role that carries the `RoboShopMySQLSecretReader` policy.

### Running a pod as that service account

Now step 4 — the pod just names the SA (from `k8-rbac/06-pod.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: roboshop
spec:
  serviceAccountName: roboshop-secret-reader   # ← run as this identity
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep", "1000"]
```

Because this pod runs as `roboshop-secret-reader`, and that SA is mapped to an IAM role that can read the secret, the AWS CLI inside it succeeds **with no keys baked in** — it picks up temporary credentials automatically:

```bash
aws secretsmanager get-secret-value --secret-id roboshop/dev/mysql_root_password
```

That is the whole point of IRSA: **no long-lived AWS access keys in the pod, no secrets in the image** — just an identity that AWS trusts, scoped to exactly the permissions the pod needs.

---

## Ingress — One Load Balancer, Many Services

`LoadBalancer` services solved *exposing a pod to the internet* — but they solve it one service at a time. Each `type: LoadBalancer` provisions its **own** cloud load balancer. Ten services means ten load balancers, ten bills, ten DNS records, and no way to say "route by URL." That doesn't scale. **Ingress** is the answer: **one** smart load balancer in front of **many** services, routing by the URL.

### CLB vs ALB — why we route on the URL

- **Classic Load Balancer (CLB)** — legacy, dumb: it forwards traffic but can't read the request.
- **Application Load Balancer (ALB)** — intelligent, layer 7: it can **read the URL and decide** where to send the request. Ingress on EKS uses the ALB.

Because the ALB reads the URL, it supports two routing styles:

| Routing | Example | Goes to |
|---------|---------|---------|
| **Host-based** | `app1.daws90s.shop` | app1 |
| | `app2.daws90s.shop` | app2 |
| **Path/context-based** | `daws90s.shop/app1` | app1 |
| | `daws90s.shop/app2` | app2 |

This is the same trick a site like `m.facebook.com` uses — one front door, the hostname/path decides the destination.

### The ALB request chain

An ALB routes through a fixed chain of pieces — you configure multiple rules, each pointing at its own target group:

```
ALB → Listener → Rule → Target group → VM (instance)   ← traditional
ALB → Listener → Rule → Target group → pod (IP)         ← with Ingress
```

The key difference in Kubernetes: the target group points at **pod IPs directly** (`target-type: ip`), not at nodes — traffic goes straight to the pod.

### Ingress vs the Ingress Controller

This is the distinction interviews probe:

- The **Ingress** resource is just the **rules** — "`app1.daws90s.shop` → the `app-1` service." It's a YAML object and does nothing on its own.
- The **Ingress Controller** is the **program running in the cluster** that reads those rules and actually builds the infrastructure. On EKS this is the **AWS Load Balancer Controller** — it watches Ingress resources and calls AWS to create the ALB, listeners, rules, and target groups.

> Ingress exposes your pods to the outside world through a load balancer, listener, rules and target groups — and it does it by asking the AWS Load Balancer Controller to create those AWS resources for you.

No controller, no ALB — the Ingress resource just sits there ignored.

### A real Ingress (from `k8s-ingress/app1`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-1
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...   # HTTPS cert
    alb.ingress.kubernetes.io/target-type: ip                    # target the pods
    alb.ingress.kubernetes.io/group.name: roboshop               # share ONE ALB
spec:
  ingressClassName: alb
  rules:
  - host: "app1.daws90s.shop"        # host-based routing
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: app-1              # → the app-1 Service → app-1 pods
            port:
              number: 80
```

Note where the configuration lives: everything ALB-specific is an **annotation** (metadata for an external system — the controller), while the routing itself is in `spec.rules`. `app2` is an identical file with `app2.daws90s.shop` → `app-2`.

The one annotation worth calling out is **`group.name: roboshop`**. Both `app-1` and `app-2` carry the same group name, so the controller puts **both** Ingresses behind a **single shared ALB** instead of one per app — the same cost lesson as before, now solved: one load balancer, many services, routed by hostname.

---

## The AWS Load Balancer Controller — Installing the Engine

The Ingress section said the **AWS Load Balancer Controller** is the program that turns Ingress rules into a real ALB. But it doesn't come with EKS — you install it, and installing it is itself a lesson in everything before it: **it's just a pod running in `kube-system` that needs AWS permissions, so it gets those permissions through IRSA** — the exact pattern from the Service Accounts section, now applied to a system component instead of your app.

Think of the controller as the **driver** that lets Kubernetes reach out and touch EC2 and Elastic Load Balancing on your behalf. No driver, no ALB.

### The four setup steps

**1. Associate the OIDC provider** — the trust bridge (same as IRSA) so the controller's ServiceAccount can assume an IAM role:

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster roboshop-dev \
  --approve
```

**2. Give the controller an identity (IRSA)** — a ServiceAccount named `aws-load-balancer-controller` in `kube-system`, bound to a policy that lets it manage load balancers:

```bash
eksctl create iamserviceaccount \
  --cluster=roboshop-dev \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

**3. Add the Helm repo** — the controller ships as a community chart (the `helm repo add` + install pattern from the Helm section):

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

**4. Install the controller, reusing the SA** — note `serviceAccount.create=false`: we already made the SA in step 2 with its IAM role attached, so the chart must **use** it, not create a fresh one with no permissions:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=roboshop-dev \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

That last flag is the whole trick: the identity (step 2) and the software (step 4) are created separately, then wired together by **name**. Let the chart create its own SA and it comes up with zero AWS permissions — the controller installs fine but silently can't build a single ALB.

---

## Gateway API — When Ingress Isn't Enough

Ingress works, but in practice it hurts — and the pain is structural, not cosmetic. The community's answer is the **Gateway API**, and the reason it exists is worth understanding before the YAML.

### Why Ingress is being retired

- **It's all annotations.** Every ALB behaviour — scheme, cert ARN, target type, listener ports, grouping — is a string in an annotation. The API server never validates it. **One typo or one missing annotation apply cleanly and then fail silently at runtime** — nothing tells you why.
- **Admin and developer share one object.** Operator concerns (cert ARN, scheme, tags) and app concerns (host, path, backend) live in the *same* Ingress resource. Both roles must touch the same file, so a change by one can break the other — a built-in dependency and a source of confusion.
- **Little real control over the ALB.** You can only express what an annotation happens to support.

Those three add up to Ingress being effectively **deprecated** for serious use.

### The Gateway API's core idea: split admin from developer

Gateway API breaks that one overloaded Ingress object into **five resources, cleanly divided by who owns them.**

**Administrator resources** (the platform team owns these — the *infrastructure*):

| Resource | Job |
|----------|-----|
| **GatewayClass** | Tells Kubernetes *which* load balancer to use — ALB or NLB |
| **LoadBalancerConfiguration** | The LB's shape — HTTP/HTTPS, ports, TLS certificates |
| **Gateway** | Wires the GatewayClass to the LoadBalancerConfiguration, and defines the **listeners** (ports/hostnames) |

**Developer resources** (the app team owns these — the *routing*):

| Resource | Job |
|----------|-----|
| **TargetGroupConfiguration** | Creates the target group and its target type (`ip`) |
| **HTTPRoute** | Adds the routing rule — "this host → my service" — onto the Gateway's listener |

The admin picks the load balancer, terminates TLS, and opens the listeners **once**. Every app team then just attaches an `HTTPRoute` to that shared Gateway. Neither side edits the other's files.

### The real resources (from `k8s-ingress/gateway`)

The **admin** stands up the load balancer once — GatewayClass → LoadBalancerConfiguration → Gateway:

```yaml
# 01 GatewayClass — "use the AWS ALB"
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: aws-alb
spec:
  controllerName: gateway.k8s.aws/alb
---
# 02 LoadBalancerConfiguration — scheme + HTTPS cert
apiVersion: gateway.k8s.aws/v1
kind: LoadBalancerConfiguration
metadata:
  name: roboshop-alb
  namespace: roboshop
spec:
  scheme: internet-facing
  ipAddressType: ipv4
  listenerConfigurations:
  - protocolPort: HTTPS:443
    certificates:
    - arn:aws:acm:us-east-1:...:certificate/72f4e658-...
---
# 03 Gateway — join the two above, open the listeners
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: roboshop
  namespace: roboshop
spec:
  gatewayClassName: aws-alb
  infrastructure:
    parametersRef:
      group: gateway.k8s.aws
      kind: LoadBalancerConfiguration
      name: roboshop-alb
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "*.daws90s.shop"
    allowedRoutes:
      namespaces:
        from: Same
```

The **developer** just points their app at that Gateway — a target group config plus an `HTTPRoute`:

```yaml
# 05 TargetGroupConfiguration — target the pods by IP
apiVersion: gateway.k8s.aws/v1
kind: TargetGroupConfiguration
metadata:
  name: app1
  namespace: roboshop
spec:
  targetReference:
    name: app1
    kind: Service
  defaultConfiguration:
    targetType: ip
---
# 06 HTTPRoute — "app1.daws90s.shop → the app1 Service"
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app1
  namespace: roboshop
spec:
  hostnames:
  - app1.daws90s.shop
  parentRefs:              # attach to the admin's Gateway listener
  - kind: Gateway
    name: roboshop
    sectionName: https
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: app1
      port: 80
      weight: 1
```

Compare with Ingress: there, `app1` and `app2` each copy-pasted the cert ARN, scheme, and every ALB annotation into their own file. Here the cert lives in exactly **one** admin file; `app2` is just another `HTTPRoute` + `TargetGroupConfiguration` pointing at the same Gateway. One source of truth.

### The three wins

1. **Role separation** — admin and developer resources never clash; each team owns its own files.
2. **Independent change** — the app team ships routing changes without touching (or waiting on) the platform team, and vice versa.
3. **Painless provider migration** — move EKS → AKS/GKE and only the admin swaps the GatewayClass and vendor CRDs. **The developers' `HTTPRoute`s don't change at all** — the exact opposite of Ingress, where every AWS-only annotation had to be rewritten in every app file.

The CRDs come in two installs — the **standard** Gateway API CRDs (vendor-neutral: GatewayClass, Gateway, HTTPRoute) and the **AWS-specific** ones (LoadBalancerConfiguration, TargetGroupConfiguration). That two-layer split is the migration story made concrete: standard routes on top, swappable vendor layer underneath.

> **A question raised in team meetings:** if EKS can create EC2/ELB resources, *should* it? An alternative is to build the ALB and target groups with **Terraform** and let EKS only register pods as targets — keeping infra creation in the IaC pipeline where it's reviewed and versioned. (The repo's `terraform/` folder hints at exactly this pattern.)

---

## Deployment Strategies — Recreate, Rolling, Blue-Green & Beyond

How you ship a new version is a design decision with real trade-offs: downtime, resource cost, rollback speed, and blast radius. Interviews love this comparison.

| Strategy | How it works | Downtime | Trade-off |
|----------|-------------|----------|-----------|
| **Recreate** | Stop the old, deploy the new, start it | **Guaranteed downtime** | Simplest; unacceptable for prod |
| **Rolling update** | Add a new pod, remove an old one, gradually | Zero | **Two versions run at once** — the app must tolerate it |
| **Blue-Green** | Two full environments; flip the switch | Zero | **Instant rollback**, but **double the resources** |
| **Canary** | Release to a few users, watch, then widen | Zero | Safe, but slower and needs good monitoring |
| **A/B testing** | Route specific users (geo/device) to v2 | Zero | For *testing a change on a segment*, not just shipping |
| **Shadow** | Copy v1's live traffic to v2 silently | Zero | v2 gets real load with **no user impact** (no real payments) |

### Blue-Green in depth

Rolling update's weakness is that **two versions serve traffic simultaneously** — developers have to make v1 and v2 coexist. Blue-Green removes that: only **one** version is ever live. You keep two complete environments — **blue** and **green** — and cut over all at once by **changing which pods the main Service selects.**

The mechanism is pure label-selector magic. Both deployments carry `app: nginx` but differ on `version:`; the main Service picks one version:

```yaml
# 01-blue-d.yaml — the blue Deployment
metadata:
  name: nginx-blue
  labels: { version: blue, app: nginx }
spec:
  replicas: 4
  selector:
    matchLabels: { version: blue, app: nginx }
```

```yaml
# 02-main-s.yaml — the live Service points at blue
kind: Service
metadata: { name: nginx }
spec:
  type: LoadBalancer
  selector:
    app: nginx
    version: blue        # ← the switch lives here
```

**Releasing green** — deploy `nginx-green` (`version: green`) alongside blue, and expose it through a separate **preview Service** so you can test it privately before any user sees it. Happy with green? Flip the main Service's selector — traffic moves instantly with **zero downtime**:

```bash
kubectl patch svc nginx -p '{"spec":{"selector":{"version":"green"}}}'
```

**Reclaiming resources** — Blue-Green's cost is running two full environments. Once green is live and stable, scale the standby (blue) to zero so you're not paying for idle pods:

```bash
kubectl patch deployment nginx-blue -p '{"spec":{"replicas":0}}'
```

**Rollback is trivial** — this is the payoff. If green misbehaves, the previous version is already sitting there. Scale it back up and flip the selector home:

```bash
kubectl patch deployment nginx-green -p '{"spec":{"replicas":4}}'   # wake the standby
kubectl patch svc nginx -p '{"spec":{"selector":{"version":"green"}}}'
```

The whole magic is in **switching the route of the main Service** — the deployments never change, only which one the Service points at. That's what makes the cutover instant and the rollback a one-liner.

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| Build vs run | Build the image → **Docker**; run the image → **Kubernetes** |
| Docker's limits | Single host, no orchestrator, no autoscaling/LB/secrets, unsafe volumes |
| **Cluster** | Master (orchestrator/control plane) + worker nodes |
| **Declarative** | You declare desired state; the control plane makes reality match |
| eksctl / kubectl | Create the cluster / talk to the cluster |
| `~/.kube/config` | Cluster authentication + authorisation config |
| **Spot vs on-demand** | Spot = 70–90% off but 2-min reclaim notice → dev/test, never prod |
| **Everything is a resource** | Every resource is YAML: `apiVersion`, `kind`, `metadata`, `spec` |
| `kubectl apply/get/describe/delete -f` | The four verbs — they work on every resource type |
| Namespaced vs cluster-scoped | `NAMESPACED: true` (Pod, Service) vs `false` (Namespace, Node) |
| `kubectl api-resources` | Lists every resource type + whether it's namespaced |
| **Namespace** | Isolated project space (`roboshop`, `expense`) |
| **kubens** | Set your default namespace; stop typing `-n roboshop` |
| **Pod** | Smallest deployable unit; 1+ containers sharing network space and storage |
| **ImagePullBackOff** | Node can't pull the image — auth issue or wrong image address |
| **CrashLoopBackOff** | Container won't stay up — check the container command |
| `kubectl exec -it <pod> -- bash` | Shell into a pod (note the `--`) |
| **Labels** | Metadata **for Kubernetes** — selectors; no special chars, max 63 chars |
| **Annotations** | Metadata **for external systems** — URLs, build info; special chars ok, max 256 KB |
| `requests` | **Soft limit** — guaranteed minimum; the scheduler places pods by this |
| `limits` | **Hard limit** — the ceiling a container can never exceed |
| CPU units | `1000m` = 1 CPU; memory in `Mi`/`Gi` |
| Why limit resources | Unlimited containers starve every other container on the host |
| **ConfigMap** | Key-value application config, kept out of the image |
| **Secret** | Same as ConfigMap but base64-encoded; `type: Opaque` |
| `envFrom: configMapRef/secretRef` | Inject every key as an environment variable |
| **Encoding ≠ encryption** | `base64 -d` reverses a Secret with no key — Secrets are **not confidential** |
| **Why services exist** | Pods are ephemeral — their IPs change, so you need a stable name |
| pod → service → endpoints | The Service tracks matching pods **by label** as its endpoints |
| **ClusterIP** | Default; internal to the cluster only |
| **NodePort** | Opens the same port (**30000–32767**) on every worker node → ClusterIP → pod |
| **LoadBalancer** | Provisions a real cloud load balancer; the production way to expose |
| `port` vs `targetPort` | Service port vs container port |
| **ReplicaSet** | Keeps N pods running; pods named `<rs-name>-<random>`; **ignores image changes** |
| **Deployment** | ReplicaSet + rollouts — what you actually use |
| **StatefulSet** / **DaemonSet** | Stable identity for stateful apps / one pod per node |
| Deployment → RS → Pods | The ownership chain |
| **Pods are immutable** | Never change a running pod — build a new image version and apply |
| **Rolling update** | Add one new pod, remove one old — zero downtime |
| `kubectl rollout status/undo` | Watch a rollout / roll back to the previous ReplicaSet |
| ConfigMap as a file | Key = filename, value = file contents; mount it as a volume |
| `volumes` vs `volumeMounts` | Pod level (attach the disk) vs container level (where it appears) |
| `subPath` | Mount a single file without replacing the whole directory |
| Everything is ephemeral | Pod dies → data dies; on EKS **nodes** are ephemeral too → durable data lives outside |
| **EBS vs EFS vs S3** | Block (1 node, fast, DBs) / File-NFS (many nodes, shared files) / Object (HTTP) |
| EBS ritual | Block storage → **format → filesystem → mount**; EFS is NFS, only **mount** |
| Ephemeral volumes | `emptyDir` (pod scratch, shared between containers) & `hostPath` (node's dir) |
| `emptyDir` | Empty dir born/dies with the pod; sidecar-logging scratch space |
| `hostPath` | Mounts a **node** dir; never for app data (pod may reschedule elsewhere); `readOnly` |
| **DaemonSet + hostPath** | One pod per node reading its `/var/log` → the log/metrics-collector pattern |
| **PV / PVC / SC** | Disk-as-K8s-object / the developer's claim / dynamic-provisioning rule |
| Pod → PVC → PV → storage | son → mother → father → wallet; app never touches the disk directly |
| Static vs dynamic | You create the disk + PV / a StorageClass creates it when a PVC asks |
| EBS static gotchas | CSI driver + node IAM (`EBSCSIDriverPolicy`) + same-AZ disk + `nodeSelector` |
| `storageClassName: ""` | Bind to my named PV; do **not** fall back to the default StorageClass |
| **Access modes** | RWO (1 node) / ROX (many RO) / RWX (many RW → EFS) / RWOncePod (1 pod) |
| **Reclaim policy** | Delete (data+disk gone) / **Retain** (data+disk kept) / Recycle (data wiped, disk kept) |
| Scheduler | Picks the node; steer it with `nodeSelector`/affinity — needed for AZ-locked EBS |
| **StorageClass** | Recipe that provisions the disk **and** its PV automatically; one per storage type |
| Dynamic vs static | PVC → SC makes the disk on demand / you pre-create the disk **and** the PV |
| Dynamic analogy | son → mother → **PhonePe wallet** — money on demand, no physical wallet |
| `provisioner` | Who creates the disk — `ebs.csi.aws.com` / `efs.csi.aws.com` |
| **`WaitForFirstConsumer`** | Create the disk **after** the pod is scheduled → lands in the node's AZ (no `nodeSelector`) |
| `Immediate` binding | Disk made before any pod exists → can land in the wrong AZ → pod stuck `Pending` |
| EFS setup gotcha | Allow **port 2049 (NFS)** in the EFS security group or mounts hang |
| EFS `capacity:` | **Dummy value** — API requires it, but EFS grows automatically |
| EFS dynamic | Creates an **access point** (sub-dir) in an existing filesystem, not a new EFS |
| `provisioningMode: efs-ap` | EFS Access Point mode; needs an existing `fileSystemId` |
| **EBS vs EFS rule** | **EBS for databases, EFS for file storage** — never a DB on EFS |
| Why MySQL needs StatefulSet | Own volume per pod + stable discoverable identity + reattach to the same disk |
| **Deployment vs StatefulSet** | Stateless, shared volume, random names / stateful, volume per pod, sticky `mysql-0` |
| Why not shared storage for a DB | You'd have to lock the other nodes — if B waits for A, the cluster is pointless |
| Async replication | Each node owns its disk; a write on NODE-A replicates asynchronously to peers |
| **StatefulSet requirements** | Own PV/PVC per pod + headless Service + stable names + ordered create/delete |
| Pod naming | `<statefulset>-0`, `-1`, `-2`; created in order, deleted in reverse |
| **Headless Service** | A Service with **`clusterIP: None`** |
| Normal vs headless DNS | `nslookup` returns **one ClusterIP** vs **all endpoints** (every pod IP) |
| Why StatefulSets need headless | Peers must discover **every** pod to replicate to, not just any one |
| Two services per DB | `mongodb-headless` for peers, `mongodb` for the app services |
| `serviceName:` | Tells a StatefulSet which headless Service governs its pods' DNS |
| **`volumeClaimTemplates`** | PVC *template* — Kubernetes creates one disk **per pod** |
| **Vertical scaling** | Bigger server — causes downtime, single point of failure, hard ceiling |
| **Horizontal scaling** | More servers — no downtime, no SPOF; what Kubernetes is built around |
| **HPA** | Adds/removes replicas to hold a metric near target (`scaleTargetRef`, min/max) |
| HPA prerequisite | Requires `resources.requests` — utilisation is a **% of requests**, else no denominator |
| `averageUtilization: 50` | Keep average CPU at ~50% of the **requested** CPU (90% = 90 out of 100) |
| What to autoscale | Deployments (stateless); not DB StatefulSets — replicas there mean cluster joins |
| **k9s** | Terminal UI for the cluster; far faster than repeated `kubectl get/describe` |
| **Selectors** | Overriding the scheduler to run pods on the nodes you choose |
| Node labels | EKS auto-labels nodes (zone, instance-type, `capacityType=SPOT`); add your own with `kubectl label node` |
| **nodeSelector** | One label the node **must** have; no match → pod stuck **`Pending`** (hard, blunt) |
| **Taint** | The **node repels** pods (*painted/polluted*) — reserve GPU/special/DB nodes |
| **Toleration** | The pod's *VIP pass* to a tainted node — **permits**, doesn't **place** |
| NoSchedule vs NoExecute | Block new pods / block new **and evict** running ones |
| Force onto a tainted node | Needs **both** a toleration (let in) **and** a nodeSelector/affinity (point there) |
| **Node affinity** | nodeSelector with multiple labels + hard/soft; `required…` vs `preferred…` |
| `IgnoredDuringExecution` | Rule checked at **schedule** time only — relabelling a running pod's node does nothing |
| **Pod affinity** | Co-locate with another pod (app next to its cache) |
| **Pod anti-affinity** | Spread pods apart — one replica per node, so a node loss costs one replica |
| Taint vs affinity | **Taint = node pushes away; affinity = pod pulls toward** |
| **Helm** | Package manager for Kubernetes **+** a templating engine for manifests |
| Chart layout | `Chart.yaml` (metadata) + `templates/` (manifests w/ placeholders) + `values.yaml` (defaults) |
| Templating syntax | `{{ .Values.deployment.replicas }}` reads into `values.yaml` |
| `version` vs `appVersion` | The **chart's** package version vs the **app/image** version inside — move independently |
| One chart, many envs | Same image everywhere; only values differ — `values-dev.yaml` / `values-prod.yaml` |
| **`helm upgrade --install`** | Idempotent — install first time, upgrade after; the CI/CD default |
| `-f values-dev.yaml` | Supply an environment's values at deploy time |
| `--set key=value` | Override one value inline without editing a file |
| `helm list` / `history` | Show installed releases / a release's revisions (for rollback) |
| Community charts | `helm repo add` + `helm upgrade --install` — like `dnf install`; e.g. EBS CSI driver, kube-prometheus-stack |
| **ServiceAccount** | The identity a pod runs as; a pod's "Linux user" (namespaced) |
| `default` SA | Auto-created per namespace with **zero permissions**; pods use it unless told otherwise |
| **IRSA** | IAM Roles for Service Accounts — a K8s SA *becomes* an AWS IAM role |
| IRSA steps | create SA → create IAM role → map via **annotation** → run pod with `serviceAccountName:` |
| **OIDC provider** | Token trust bridge so AWS believes the pod's SA token; `eksctl utils associate-iam-oidc-provider` |
| `eksctl create iamserviceaccount` | One command: makes the SA, the IAM role, and the annotation linking them |
| Why IRSA | **No AWS keys in the pod / no secrets in the image** — temporary creds via the SA |
| **Ingress** | The **rules** — "host/path → service"; YAML only, does nothing alone |
| **Ingress Controller** | The program that reads Ingress rules and builds the real LB; on EKS = **AWS Load Balancer Controller** |
| Why Ingress | **One** smart LB for **many** services, vs `LoadBalancer` = one LB per service |
| **CLB vs ALB** | Legacy/dumb forwarder vs layer-7 that **reads the URL and decides** |
| Host vs path routing | `app1.daws90s.shop` → app1 / `daws90s.shop/app1` → app1 |
| ALB chain | ALB → Listener → Rule → Target group → **pod IP** (`target-type: ip`) |
| `group.name` | Same group on many Ingresses → they **share one ALB** (not one per app) |
| **AWS LB Controller** | A pod in `kube-system` that turns Ingress/Gateway into a real ALB; the "driver" to EC2/ELB |
| Controller install | OIDC → `iamserviceaccount` (IRSA) → `helm repo add eks` → `helm install` |
| `serviceAccount.create=false` | Reuse the IRSA SA made by eksctl; let Helm make its own and it has **zero AWS perms** |
| **Why Ingress is retired** | All-annotation (silent typos) + admin & dev share one object + little ALB control |
| **Gateway API** | Ingress's successor — splits one object into 5 resources by **who owns them** |
| **GatewayClass** | Admin: which LB — ALB or NLB |
| **LoadBalancerConfiguration** | Admin: LB shape — HTTP/HTTPS, ports, TLS cert |
| **Gateway** | Admin: joins GatewayClass + LBConfig, opens the **listeners** |
| **TargetGroupConfiguration** | Developer: creates the target group + target type (`ip`) |
| **HTTPRoute** | Developer: attaches "host → service" routing onto a Gateway listener |
| Gateway wins | Role separation + teams change independently + provider migration leaves routes untouched |
| Gateway CRDs | Standard (vendor-neutral) + AWS-specific — the swappable-vendor layer made real |
| **Recreate** | Stop old → deploy new → start; **guaranteed downtime** |
| **Rolling update** | Add new / drop old gradually; zero downtime but **two versions live at once** |
| **Blue-Green** | Two full envs; flip the Service selector — zero downtime + instant rollback, **double resources** |
| Blue-Green switch | `kubectl patch svc nginx -p '{"spec":{"selector":{"version":"green"}}}'` — the magic is in the route |
| Blue-Green cost fix | Scale the standby to `replicas: 0`; wake it on rollback |
| **Canary** | Release to a few users, monitor, then widen |
| **A/B testing** | Route a segment (geo/device) to v2 — testing a change, not just shipping |
| **Shadow** | Copy live v1 traffic to v2 silently — real load, no user impact |
| **startupProbe** | Has it booted? Runs **once**; buys slow starters time |
| **readinessProbe** | Ready for traffic? Fails → **removed from Service endpoints** (no restart) |
| **livenessProbe** | Still alive? Fails → **pod restarted** |
| Standard resource set | Namespace + Deployment + ConfigMap + Secret + Service |

---
