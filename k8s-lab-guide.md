# Kubernetes Workshop — Hands-on Lab Guide
**Platform:** KillerKoda · **Duration:** 75 min · **Image:** `ghcr.io/narcisseobadiah/k8s-workshop-demo:latest`

---

## Before you start

Open your browser and go to **[killercoda.com](https://killercoda.com)** → pick the **Kubernetes** playground (1 node or 2 nodes).

Once the terminal is ready, verify your cluster:

```bash
kubectl get nodes
```

You should see at least one node in `Ready` state. You're good to go.

---

## Step 1 — Deploy the app (10 min)
### The imperative way

**Goal:** Get the app running quickly using kubectl commands — no files, no YAML.

```bash
kubectl create deployment k8s-demo \
  --image=ghcr.io/narcisseobadiah/k8s-workshop-demo:latest \
  --port=3000
```

Watch the pod start:

```bash
kubectl get pods -w
# Press Ctrl+C when status shows Running
```

Inspect what Kubernetes did:

```bash
kubectl describe pod <your-pod-name>
```

> Look at the **Events** section at the bottom — every scheduling and startup action is logged there.

**✓ Checkpoint:** `kubectl get pods` shows `1/1 Running`

> **Imperative vs declarative:** This approach is great for quick experiments. But in real teams, nobody types `kubectl create` in production — they write YAML files and let the cluster reconcile. That's what Step 2 is about.

---

## Step 2 — The declarative way (15 min)
### From commands to manifests

**Goal:** Tear down what you just created and redeploy the exact same app using a YAML manifest. Understand why this is how Kubernetes is actually used.

### 2.1 Delete the imperative Deployment

```bash
kubectl delete deployment k8s-demo
kubectl get pods -w
# Wait until all pods are gone, then Ctrl+C
```

### 2.2 Create your manifest

Create a file called `deployment.yaml`:

```bash
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo
  labels:
    app: k8s-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-demo
  template:
    metadata:
      labels:
        app: k8s-demo
    spec:
      containers:
        - name: k8s-workshop-demo
          image: ghcr.io/narcisseobadiah/k8s-workshop-demo:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
EOF
```

> **What is `100m` CPU?** Kubernetes measures CPU in millicores. `100m` = 0.1 of a CPU core. `requests` is what the scheduler uses to find a node with enough room. `limits` is the hard cap — the container gets throttled (CPU) or killed (memory) if it exceeds it.

### 2.3 Apply the manifest

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl get pods -w
# Wait for Running, then Ctrl+C
```

### 2.4 Make a change — edit and re-apply

Change the replica count from 1 to 2 by editing the file:

```bash
sed -i 's/replicas: 1/replicas: 2/' deployment.yaml
kubectl apply -f deployment.yaml
kubectl get pods
```

> **This is the declarative mindset.** You describe *what you want*, commit the file to Git, and run `kubectl apply`. Kubernetes figures out *what needs to change*. Your manifest becomes the source of truth — reviewable, versioned, auditable.

**✓ Checkpoint:** `kubectl get pods` shows `2/2` pods running, deployed from YAML.

---

## Step 3 — Namespaces (10 min)
### Isolating workloads

**Goal:** Understand how namespaces separate workloads within the same cluster.

### 3.1 See what namespaces already exist

```bash
kubectl get namespaces
```

You'll see `default`, `kube-system`, `kube-public` at minimum. Everything you've created so far lives in `default`.

### 3.2 Create a dedicated namespace for the workshop

```bash
kubectl create namespace workshop
```

### 3.3 Deploy the app into the new namespace

```bash
kubectl apply -f deployment.yaml -n workshop
```

### 3.4 Compare what's visible in each namespace

```bash
# Only sees pods in 'default'
kubectl get pods

# Only sees pods in 'workshop'
kubectl get pods -n workshop

# Sees everything across all namespaces
kubectl get pods --all-namespaces
```

> **Why namespaces matter:** In a real cluster, you'd have namespaces per team, environment, or application (`frontend`, `backend`, `staging`, `prod`). Each namespace can have its own resource quotas, network policies, and access controls — full isolation on a shared cluster.

### 3.5 Clean up the workshop namespace

```bash
kubectl delete namespace workshop
# This deletes everything inside it — pods, services, all resources
```

**✓ Checkpoint:** `kubectl get pods -n workshop` returns "No resources found".

---

## Step 4 — Expose as a Service (10 min)

**Goal:** Make the app reachable via a stable network endpoint.

### 4.1 Add a Service to your manifest

Append a Service to `deployment.yaml`:

```bash
cat <<EOF >> deployment.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-demo
spec:
  selector:
    app: k8s-demo
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
EOF
```

### 4.2 Apply the updated manifest

```bash
kubectl apply -f deployment.yaml
kubectl get service k8s-demo
```

Note the `NodePort` value in the `PORT(S)` column — it looks like `3000:3XXXX/TCP`.

### 4.3 Access the app

In KillerKoda, click **Traffic / Ports** at the top and open the NodePort you noted.

Or use curl:

```bash
curl http://localhost:3XXXX
```

**✓ Checkpoint:** You see a response from the app.

---

## Step 5 — Scale up & self-healing (10 min)

**Goal:** Scale to 4 replicas and watch Kubernetes recover from a pod failure.

### 5.1 Scale via the manifest (the right way)

Edit `deployment.yaml`, change `replicas: 2` to `replicas: 4`, then:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -o wide
```

The `-o wide` flag shows which node each pod runs on.

### 5.2 Kill a pod — watch self-healing

```bash
kubectl get pods
kubectl delete pod <paste-any-pod-name>
kubectl get pods -w
```

You'll see: `Terminating` → a new pod immediately enters `Pending` → `ContainerCreating` → `Running`.

> The Deployment declared desired = 4. The controller manager constantly reconciles actual vs desired. The moment a pod disappears, it creates a replacement.

**✓ Checkpoint:** Still 4 pods running. Cluster healed in seconds.

---

## Step 6 — Rolling update (10 min)

**Goal:** Update the app with zero downtime.

### 6.1 Trigger a rolling update

```bash
kubectl set image deployment/k8s-demo \
  k8s-workshop-demo=ghcr.io/narcisseobadiah/k8s-workshop-demo:latest
```

### 6.2 Watch the rollout

```bash
kubectl rollout status deployment/k8s-demo
```

Kubernetes replaces pods one by one — a new pod must be healthy before the old one terminates.

### 6.3 Check rollout history

```bash
kubectl rollout history deployment/k8s-demo
```

**✓ Checkpoint:** Rollout completes. App stayed reachable throughout.

---

## Bonus — for fast finishers

```bash
# Stream live logs from all pods in the Deployment
kubectl logs -l app=k8s-demo --follow

# Open a shell inside a running pod
kubectl exec -it <pod-name> -- /bin/sh

# Check resource usage per pod (if metrics-server is available)
kubectl top pods

# See resource requests & limits on running pods
kubectl describe pod <pod-name> | grep -A6 "Requests\|Limits"

# Roll back to the previous version
kubectl rollout undo deployment/k8s-demo

# Clean up everything
kubectl delete -f deployment.yaml
```

---

## What just happened — the full picture

| What you did | What Kubernetes did |
|---|---|
| `kubectl create` (Step 1) | API server stored desired state in etcd; scheduler placed pod on a node; kubelet pulled the image |
| `kubectl apply -f` (Step 2) | Same result — but now your intent lives in a file, not a command |
| `resources.requests` | Scheduler used CPU/memory requests to find a node with enough capacity |
| `resources.limits` | kubelet enforces the cap — container gets throttled (CPU) or OOMKilled (memory) if exceeded |
| `namespace` (Step 3) | Workloads scoped and isolated — deleting the namespace deletes everything inside it |
| `expose` (Step 4) | Service created; kube-proxy set up rules to forward traffic to any matching pod |
| `scale --replicas=4` (Step 5) | Controller manager reconciled actual (2) vs desired (4); created 2 new pods |
| `delete pod` (Step 5) | Controller detected 3 < 4 desired; replacement pod created within seconds |
| `set image` (Step 6) | New ReplicaSet created; pods replaced one-by-one — rolling strategy, zero downtime |

---

*Workshop by Narcisse Obadiah · 42 Heilbronn · Image: `ghcr.io/narcisseobadiah/k8s-workshop-demo:latest`*
