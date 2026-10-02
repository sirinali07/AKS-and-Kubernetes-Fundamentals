# Lab: Namespaces, Pods, YAMLs, and Labels

## What You Will Do

| Step | Topic | You will learn |
|---|---|---|
| 1 | Namespaces | Create a private workspace inside the cluster |
| 2 | Pods (command) | Run your first Pod with one command |
| 3 | Pods (YAML) | Define a Pod in a file, the way it's done in real projects |
| 4 | Labels and selectors | Tag Pods and filter them |
| 5 | Multi-container Pod | Run two containers together in one Pod |
| 6 | Resources | Control how much CPU and memory a Pod can use |
| 7 | Cleanup | Remove everything with one command |

## Before You Start

You need an AKS cluster with `kubectl` connected (see the [AKS Cloud Shell lab](AKS-CloudShell-Lab.md)). Check the connection:

```bash
kubectl get nodes
```

Expected output: nodes with `STATUS` = `Ready`.

---

## Key Concepts

This is how the pieces fit together:

```text
Cluster
 ├── Namespace: default
 ├── Namespace: kube-system
 └── Namespace: lab-ns            <- you create this
      ├── Pod: pod-1
      │    └── Container: nginx
      └── Pod: multi-container-pod
           ├── Container: nginx
           └── Container: busybox
```

**Container**
A packaged application (code plus everything it needs to run) built from an image, such as `nginx`. Containers are lightweight and start in seconds.

**Pod**
- The smallest object you can deploy in Kubernetes. Kubernetes never runs a container on its own; it always wraps it in a Pod.
- A Pod usually holds **one** container, but it can hold several that work closely together.
- Containers in the same Pod share an **IP address** (they talk over `localhost`) and can share **storage**.
- Each Pod gets its own IP address inside the cluster.
- Pods are **temporary**: if a Pod is deleted or its node fails, it isn't recreated automatically. In real projects, Deployments manage Pods for you. You'll meet Deployments in a later lab.

**Pod lifecycle (STATUS)**

| Status | Meaning |
|---|---|
| `Pending` / `ContainerCreating` | Kubernetes is choosing a node or pulling the image |
| `Running` | At least one container is running |
| `Succeeded` / `Completed` | All containers finished successfully |
| `Failed` / `Error` | A container exited with an error |
| `CrashLoopBackOff` | A container keeps crashing and restarting |
| `ImagePullBackOff` | The image name is wrong or the image can't be downloaded |

**Namespace**
- A namespace is a logical folder that groups resources inside one cluster.
- Object names must be unique **within** a namespace, but the same name can be reused in different namespaces.
- Namespaces separate teams or environments (dev, test, prod), and you can apply access rules and resource quotas per namespace.
- Deleting a namespace deletes everything inside it.

**Labels and selectors**
- **Labels** are key-value tags on objects, for example `app=web` or `env=prod`.
- **Selectors** pick objects by label. This is how Kubernetes connects objects: a Service finds its Pods with a selector, not by Pod name.

**YAML manifest**
A file that describes the **desired state** of an object (what you want to run). You run `kubectl apply`, and Kubernetes makes the cluster match the file. This is called the *declarative* approach. Running `kubectl run` directly is the *imperative* approach.

**Requests and limits**
These settings control how much CPU and memory a container is guaranteed (**requests**) and how much it's allowed to use at most (**limits**).

---

## Step 1: Namespaces

A **namespace** is a logical folder inside the cluster. Teams and environments (dev, test, prod) use separate namespaces so their resources don't mix.

**1.1: See the namespaces that already exist**

```bash
kubectl get ns
```

| Namespace | Purpose |
|---|---|
| `default` | Used when you don't specify a namespace |
| `kube-system` | Kubernetes' own system components (do not touch) |
| `kube-public`, `kube-node-lease` | Internal cluster use |

**1.2: See what's running in `kube-system`**

```bash
kubectl get pods -n kube-system
```

`-n <namespace>` tells `kubectl` which namespace to look in.

**1.3: Create your own namespace for this lab**

```bash
kubectl create ns lab-ns
```

**1.4: Make it your default namespace**, so you don't need to type `-n lab-ns` every time:

```bash
kubectl config set-context --current --namespace=lab-ns
```

```bash
kubectl config view --minify | grep namespace
```

Expected output: `namespace: lab-ns`. From now on, everything you create goes into `lab-ns`.

---

## Step 2: Create a Pod with a Command

A **Pod** is the smallest unit in Kubernetes. It wraps one or more containers.

**2.1: Run an nginx Pod**

```bash
kubectl run pod-1 --image=nginx --port=80
```

**2.2: Check the Pod**

```bash
kubectl get pods
```

Expected output: `pod-1` with `STATUS` = `Running`. If it shows `ContainerCreating`, wait a few seconds and run the command again.

```bash
kubectl get pods -o wide
```

`-o wide` adds the Pod's IP address and the node it runs on.

**2.3: See the Pod's details and events**

```bash
kubectl describe pod pod-1
```

Read the **Events** section at the bottom. It shows the image being pulled and the container starting. This is the first place to look when a Pod has problems.

**2.4: Go inside the container**

```bash
kubectl exec -it pod-1 -- /bin/bash
```

```bash
ls /usr/share/nginx/html
```

```bash
exit
```

---

## Step 3: Create a Pod from YAML

Commands are quick, but real projects define resources in **YAML files** so they can be saved, reviewed, and reused.

**3.1: Let `kubectl` generate a YAML template for you**

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

`--dry-run=client` prints the YAML without creating anything. Notice the four main sections, which every Kubernetes YAML has:

| Field | Meaning |
|---|---|
| `apiVersion` | API version of the object (`v1` for Pods) |
| `kind` | Type of object (`Pod`) |
| `metadata` | Name and labels |
| `spec` | What to run (containers, images, ports) |

**3.2: Create your own YAML file**

Copy and paste this whole block into Cloud Shell. It writes the file `httpd-pod.yaml`:

```bash
cat <<EOF > httpd-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
  labels:
    env: prod
    app: httpd
spec:
  containers:
    - name: httpd-container
      image: httpd
      ports:
        - containerPort: 80
EOF
```

**3.3: Create the Pod from the file**

```bash
kubectl apply -f httpd-pod.yaml
```

```bash
kubectl get pods
```

Expected output: `httpd-pod` with `STATUS` = `Running`.

---

## Step 4: Labels and Selectors

- **Labels** are key-value tags on objects, for example `env=prod`.
- **Selectors** filter objects by their labels. Kubernetes uses selectors to connect Services and Deployments to the right Pods.

**4.1: Show labels**

```bash
kubectl get pods --show-labels
```

`httpd-pod` has the labels from its YAML (`env=prod`, `app=httpd`).

**4.2: Add a label to a running Pod**

```bash
kubectl label pod pod-1 env=dev
```

**4.3: Create a Pod with labels using a command**

```bash
kubectl run pod-2 --image=nginx --labels="env=dev,app=web"
```

**4.4: Filter Pods using selectors**

```bash
kubectl get pods -l env=dev
```

Expected output: `pod-1` and `pod-2`.

```bash
kubectl get pods -l env=prod
```

Expected output: `httpd-pod` only.

**4.5: Remove a label** (add `-` after the key)

```bash
kubectl label pod pod-1 env-
```

```bash
kubectl get pods --show-labels
```

---

## Step 5: Multi-Container Pod

A Pod can run more than one container. The containers share the same network (`localhost`) and can share storage. This is common for helper ("sidecar") containers.

> **Note:** You can't add a container to a Pod that's already running. Define all of its containers in the YAML from the start.

**5.1: Create the YAML and apply it**

```bash
cat <<EOF > multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: nginx-ctr
      image: nginx
    - name: busybox-ctr
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
EOF
```

```bash
kubectl apply -f multi-container-pod.yaml
```

```bash
kubectl get pod multi-container-pod
```

Expected output: `READY` = `2/2`, which means both containers are running.

**5.2: Enter a specific container** with `-c <container-name>`

```bash
kubectl exec -it multi-container-pod -c busybox-ctr -- sh
```

**5.3: From busybox, reach nginx over `localhost`**

```bash
wget -qO- localhost:80
```

Expected output: the HTML of the "Welcome to nginx!" page. This works because both containers share one network.

```bash
exit
```

---

## Step 6: Resource Requests and Limits

| Setting | Meaning | If exceeded |
|---|---|---|
| **requests** | Minimum CPU and memory guaranteed to the container. The scheduler uses it to choose a node. | — |
| **limits** | Maximum CPU and memory the container can use | CPU is **throttled** (slowed); memory overuse is **OOMKilled** (the container is restarted) |

Units: `100m` = 0.1 CPU core; `128Mi` = 128 mebibytes of memory.

**6.1: Create a Pod with requests and limits**

```bash
cat <<EOF > pod-with-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-resources
spec:
  containers:
    - name: nginx-ctr
      image: nginx
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
EOF
```

```bash
kubectl apply -f pod-with-resources.yaml
```

**6.2: Verify the settings**

```bash
kubectl describe pod nginx-with-resources | grep -A 6 Limits
```

Expected output: the `Limits` (cpu 200m, memory 256Mi) and `Requests` (cpu 100m, memory 128Mi) you set.

---

## Step 7: Cleanup

Deleting the namespace deletes **every** Pod inside it:

```bash
kubectl delete ns lab-ns
```

Switch your default namespace back to `default`:

```bash
kubectl config set-context --current --namespace=default
```

Remove the YAML files:

```bash
rm -f httpd-pod.yaml multi-container-pod.yaml pod-with-resources.yaml
```

---

## Summary

| Command | Purpose |
|---|---|
| `kubectl get ns` / `kubectl create ns <name>` | List or create namespaces |
| `kubectl run <name> --image=<image>` | Create a Pod quickly |
| `kubectl apply -f <file>.yaml` | Create a Pod from YAML |
| `kubectl get pods -o wide` | List Pods with IP and node |
| `kubectl describe pod <name>` | Show details and events (for troubleshooting) |
| `kubectl exec -it <pod> [-c <container>] -- sh` | Open a shell inside a container |
| `kubectl label pod <name> key=value` | Add a label |
| `kubectl get pods -l key=value` | Filter Pods by label |
