# Lab: Create an AKS Cluster Using Azure Cloud Shell

## Objective

Use Azure Cloud Shell to:

1. Create a resource group
2. Deploy an Azure Kubernetes Service (AKS) cluster
3. Connect `kubectl` to the cluster
4. Verify the cluster is working by running a test app
5. Clean up all resources

**Estimated time:** 20–25 minutes

## Prerequisites

- An active Azure subscription with permission to create resources (for example, the **Contributor** role)
- Browser access to https://portal.azure.com

> **Note:** Cloud Shell already has the `az` and `kubectl` command-line tools installed, so you don't need to install anything.

---

## Step 1: Open Azure Cloud Shell

1. Go to https://portal.azure.com and sign in.
2. In the top bar, click the **Cloud Shell** icon (`>_`).
3. If prompted:
   - Select **Bash**.
   - First-time users: choose **No storage account required** for a quick lab, or **Create storage** if you want your files to persist between sessions.
4. Wait a few seconds for the shell to start.

**Confirm the subscription you're using:**

```bash
az account show --output table
```

If you have more than one subscription, switch to the correct one:

```bash
az account set --subscription "<subscription-name-or-id>"
```

---

## Step 2: Set Environment Variables

Variables let you reuse values without retyping them.

```bash
RESOURCE_GROUP="K8S-RG"
CLUSTER_NAME="aks-cluster"
LOCATION="centralus"
```

**Verify:**

```bash
echo $RESOURCE_GROUP $CLUSTER_NAME $LOCATION
```

Expected output: `K8S-RG aks-cluster centralus`

> **Warning:** Variables last only for the current session. If Cloud Shell times out or you reconnect, run this step again before continuing.

---

## Step 3: Register the AKS Resource Provider (first time only)

Your subscription must be registered to use AKS. New or lab subscriptions often aren't registered yet.

```bash
az provider register --namespace Microsoft.ContainerService
```

```bash
az provider show --namespace Microsoft.ContainerService --query registrationState --output tsv
```

Expected output: `Registered`. If it shows `Registering`, wait a minute and check again.

---

## Step 4: Create a Resource Group

A **resource group** is a logical container for related Azure resources. Deleting it deletes everything inside it.

```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

Expected output: JSON that includes `"provisioningState": "Succeeded"` and `"name": "K8S-RG"`.

---

## Step 5: Create the AKS Cluster

This step creates a Kubernetes cluster with **2 worker nodes**. Azure manages the control plane, and the nodes are VMs in your subscription.

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 2 \
  --node-vm-size Standard_DS2_v2 \
  --tier free \
  --generate-ssh-keys
```

| Flag | Meaning |
|---|---|
| `--node-count 2` | Number of worker nodes (VMs) |
| `--node-vm-size Standard_DS2_v2` | VM size of each node (2 vCPU, 7 GB RAM) |
| `--tier free` | Free control plane with no SLA, which is fine for labs |
| `--generate-ssh-keys` | Creates SSH keys for node access if none exist |

This takes about **5–10 minutes**.

Expected output: JSON that includes `"provisioningState": "Succeeded"` and the cluster's `fqdn`.

> **Troubleshooting:** If you see a VM size or quota error (for example, `VM size not allowed` or `QuotaExceeded`), try a different size such as `--node-vm-size Standard_B2s` or `Standard_D2s_v3`, or a different `LOCATION`.

**Confirm the cluster status:**

```bash
az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query provisioningState --output tsv
```

Expected output: `Succeeded`

---

## Step 6: Connect to the AKS Cluster

This step downloads the cluster credentials so `kubectl` can talk to your cluster.

```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

Expected output:

```text
Merged "aks-cluster" as current context in /home/<user>/.kube/config
```

**Check the current context:**

```bash
kubectl config current-context
```

Expected output: `aks-cluster`

---

## Step 7: Verify the Cluster Is Running

**7.1: List the nodes**

```bash
kubectl get nodes -o wide
```

Expected output: **2 nodes** with `STATUS` = `Ready`.

```text
NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-12345678-vmss000000   Ready    <none>   5m    v1.xx.x
aks-nodepool1-12345678-vmss000001   Ready    <none>   5m    v1.xx.x
```

**7.2: Check cluster info**

```bash
kubectl cluster-info
```

Expected output: the URLs of the Kubernetes control plane and CoreDNS.

**7.3: Check system pods**

```bash
kubectl get pods -n kube-system
```

Expected output: system pods (such as `coredns`, `konnectivity-agent` and `metrics-server`) in the `Running` state.

---

## Step 8: Deploy a Test App (Optional)

This step shows the cluster can run workloads and expose them to the internet.

```bash
kubectl create deployment nginx --image=nginx --replicas=2
```

```bash
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

```bash
kubectl get service nginx --watch
```

Wait until `EXTERNAL-IP` changes from `<pending>` to a public IP address, then press `Ctrl+C`.

Open `http://<EXTERNAL-IP>` in your browser. You should see the **"Welcome to nginx!"** page.

**Clean up the test app:**

```bash
kubectl delete service nginx
```

```bash
kubectl delete deployment nginx
```

---

## Step 9: Clean Up Resources

> **Important:** An AKS cluster keeps charging you for its VMs until you delete it.

**9.1: List existing AKS clusters**

```bash
az aks list --output table
```

**9.2: Delete the resource group (recommended)**

Deleting the resource group removes the AKS cluster **and** everything else inside it. Azure also deletes the auto-created node resource group (`MC_K8S-RG_aks-cluster_centralus`).

```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

> **Warning:** This deletes **all** resources in `K8S-RG`, not just AKS.

**Alternative: delete only the cluster** (keeps the resource group)

```bash
az aks delete --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --yes --no-wait
```

**9.3: Verify the deletion** (after a few minutes)

```bash
az group exists --name $RESOURCE_GROUP
```

Expected output: `false`

---

## Command Summary

| Step | Command |
|---|---|
| Set variables | `RESOURCE_GROUP="K8S-RG"` · `CLUSTER_NAME="aks-cluster"` · `LOCATION="centralus"` |
| Register provider | `az provider register --namespace Microsoft.ContainerService` |
| Create resource group | `az group create --name $RESOURCE_GROUP --location $LOCATION` |
| Create cluster | `az aks create -g $RESOURCE_GROUP -n $CLUSTER_NAME --node-count 2 --generate-ssh-keys` |
| Connect | `az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME` |
| Verify | `kubectl get nodes` |
| Clean up | `az group delete --name $RESOURCE_GROUP --yes --no-wait` |

## Lab Complete

You have:

- Created a resource group
- Deployed a 2-node AKS cluster
- Connected `kubectl` to the cluster
- Verified the nodes and system pods are healthy
- Deployed and exposed a test app
- Cleaned up all resources

**Next lab:** Namespaces, Pods, YAMLs, and Labels
