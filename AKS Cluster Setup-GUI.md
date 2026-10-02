# Lab: Create an Azure Kubernetes Service (AKS) Cluster (Portal)

## Lab Overview

In this lab you will:

1. Create an **AKS cluster** using the Azure Portal
3. Connect to the cluster using **Azure CLI + kubectl**
4. Verify that the cluster is working
5. (Optional) Clean up resources

> **Theory (very short)**  
> - **AKS** = Managed Kubernetes on Azure.  
> - **Resource Group** = Logical folder for Azure resources.  
> - **Node pool** = Set of VMs that run your pods.  
> - **Standard_D2s_v3** = Common VM size (2 vCPU, 8 GB RAM).

---

## Prerequisites

- An active **Azure subscription**
- Access to **Azure Portal**: https://portal.azure.com  
- Basic understanding of Kubernetes terms: **cluster, node, pod**.

---

## Task 1 – Sign in and Open AKS Creation Wizard

1. Go to [Azure Portal](https://portal.azure.com) and **Sign in**.
2. In the top search bar, type **“Kubernetes services”** and select it.
3. Click **+ Create** → **+ Create Kubernetes cluster**.

 
<img width="1402" height="854" alt="image" src="https://github.com/user-attachments/assets/ea8bb66c-2e19-4ee8-8838-7c34eb4e2a0b" />
    

## Task 2 – Configure Basic Cluster Settings

You are on the **Basics** tab.

### 4.1 Project details

1. **Subscription**  
   - Select: `Azure subscription 1` (or whatever your portal shows with that name).
2. **Resource group**
   - Select your `Resource Group`   
4. **Region**  
   - Select the region where you want the cluster to be deployed (e.g., East US, West US).
5. **AKS pricing tier** (if visible):
   - Choose **Free** (default).   
6. **Kubernetes Version:**
   - Select the Kubernetes version you wish to use (default or latest is recommended).

  Leave other options at default
  
<img width="1269" height="901" alt="image" src="https://github.com/user-attachments/assets/badc6ac1-fe3f-4caf-941a-66861e0d45f5" />

Click **Next**


## Task 3 – Configure Node Pool

1. **Node size**: click **Change size**.
   - Search for `Standard_B2s`
   - Select **Standard_B2s**.
   - Click **Select**.
2. **Node count**:
   - Set **Node count** to `1` or `2` (for lab, `1` is enough and cheaper).
3. Leave other options at default (System node pool, Virtual Machine scale set).

<img width="1268" height="891" alt="image" src="https://github.com/user-attachments/assets/004f5fa0-6407-4358-ac7c-66a12a488326" />


Click **Next**


## Task 4 – Configure Networking

On the **Networking** tab:

1. **Network configuration**:
   - Choose **Azure CNI** for better integration with Azure services.
2. Leave other settings as default (new virtual network, default subnets, etc.).

<img width="1271" height="888" alt="image" src="https://github.com/user-attachments/assets/b39adcfc-6375-48e1-a3de-e8e6106f58f6" />


Click **Next: Monitoring** 


## Task 5 – Monitoring 

1. **Enable Container insights**:  
   -  Enable monitoring for your cluster to track performance metrics and logs.

<img width="1274" height="888" alt="image" src="https://github.com/user-attachments/assets/59b888f7-4725-4810-ac1b-c8e7dac53244" />

`Leave other *options* at default`

 
## Task 6 – Review and Create Cluster

Click **Review + create** to validate your configuration.

- Azure will run **Validation**.
- Once validation passes, click **Create**.
- Deployment will start and take a few minutes.
- When it finishes, you will see **“Your deployment is complete”**.


Click **Go to resource** to open the AKS cluster overview page.

<img width="1910" height="901" alt="image" src="https://github.com/user-attachments/assets/e990ad4c-37be-42ec-961a-57fafcb1e511" />


## Task 7 – Connect to the AKS Cluster

You can use **Cloud Shell** or your local terminal.

1. In the top-right of Azure Portal, click the **Cloud Shell** icon (🖥️ >_).
2. Choose **Bash**.
3. Download the credentials to connect to your AKS cluster

```bash
az aks get-credentials --resource-group aks-lab-RG --name aks-lab-cluster
```
4. Verify the cluster connection
```bash
kubectl get nodes
```
<img width="1280" height="546" alt="image" src="https://github.com/user-attachments/assets/df932770-e7a1-4aef-bb0e-4e629609b01e" />

You should see list of nodes are in Ready state.


## Task 8 – (Optional) Clean up resources

```bash
az group delete --name aks-lab-RG --yes --no-wait
```









