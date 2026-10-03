# Lab: NGINX Ingress Controller on AKS (Azure Kubernetes Service)

## Prerequisites – Connect to the existing AKS cluster

Log in to Azure (or use **Azure Cloud Shell**, which already has `az` and `kubectl`):
```
az login
```
Select the subscription that holds the cluster (if you have more than one):
```
az account set --subscription <subscription-name-or-id>
```
Find your cluster name and resource group:
```
az aks list -o table
```
Set variables to match your cluster:
```
RG=<your-resource-group>
CLUSTER=<your-aks-cluster-name>
```
Download cluster credentials (install `kubectl` first with `az aks install-cli` if needed, skip in Cloud Shell):
```
az aks get-credentials --resource-group $RG --name $CLUSTER
```
Confirm `kubectl` is pointing at the right cluster and the nodes are `Ready`:
```
kubectl config current-context
```
```
kubectl get nodes -o wide
```

Check whether an ingress controller is already installed. If the `ingress-nginx` namespace or an `nginx` IngressClass already exists, **do not** apply the manifest in Task 1 again; skip to *Create a Namespace and Deploy Applications*, and **skip deleting the controller in Task 2**, because other teams may be using it:
```
kubectl get ns ingress-nginx
```
```
kubectl get ingressclass
```

> **Note:** The upstream `kubernetes/ingress-nginx` project was retired in March 2026 (no further releases or security fixes). It is fine for learning; for production on AKS use the managed **Application Routing add-on** (`az aks approuting enable -g $RG -n $CLUSTER`) or Gateway API.

---

## Task 1 – Deploy nginx-ingress-controller

Deploy the nginx-ingress-controller using the cloud-provider manifest.
(Version `v1.1.0` from older labs does not support current AKS Kubernetes versions, so a newer release is used.)
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

Verify the namespace, deployment and pods:
```
kubectl get ns
```
```
kubectl -n ingress-nginx get all
```
Wait until the controller pod is `Running` and `1/1` ready:
```
kubectl -n ingress-nginx wait --for=condition=ready pod -l app.kubernetes.io/component=controller --timeout=180s
```

View controller logs (optional):
```
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
```

#### Azure Load Balancer health probe (required on AKS)
AKS integrates with Azure Load Balancer, so the `ingress-nginx-controller` service stays type **LoadBalancer** and gets a **public IP** — no need to switch to NodePort.
On AKS 1.24+ the Azure LB health probe must point at `/healthz`, otherwise the public IP will not respond:
```
kubectl -n ingress-nginx annotate svc ingress-nginx-controller service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path=/healthz --overwrite
```
Watch until `EXTERNAL-IP` changes from `<pending>` to a public IP (1–2 minutes), then press `Ctrl+C`:
```
kubectl -n ingress-nginx get svc ingress-nginx-controller -w
```

---

#### Create a Namespace and Deploy Applications
Create a namespace named ***ingress-ns***:
```
kubectl create ns ingress-ns
```
Deploy Httpd and Nginx deployments:
```
kubectl -n ingress-ns create deployment httpd-dep --image httpd --port 80 --replicas 2
```
```
kubectl -n ingress-ns create deployment nginx-dep --image nginx --port 80 --replicas 2
```
Verify the deployed applications:
```
kubectl -n ingress-ns get deploy
```
```
kubectl -n ingress-ns get pods -o wide
```

Expose the deployments as ClusterIP services:
```
kubectl -n ingress-ns expose deployment httpd-dep --port 80 --name httpd-svc
```
The nginx service listens on **8080** but must forward to container port **80**:
```
kubectl -n ingress-ns expose deployment nginx-dep --port 8080 --target-port 80 --name nginx-svc
```

Verify the services and their endpoints:
```
kubectl -n ingress-ns get svc
```
```
kubectl -n ingress-ns get ep
```

---

#### Set Up Ingress Rules
Create a file named ***ingressrule.yaml***:
```
vi ingressrule.yaml
```
Copy the following Ingress rule into it.
`/test` and anything under it goes to **nginx-svc** (the `/test` prefix is stripped by the rewrite); everything else goes to **httpd-svc**.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  namespace: ingress-ns
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /test(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: nginx-svc
            port:
              number: 8080
      - path: /()(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
```
Apply the Ingress rule:
```
kubectl apply -f ingressrule.yaml
```
Verify the Ingress (the `ADDRESS` column shows the LoadBalancer public IP after ~1 minute):
```
kubectl -n ingress-ns get ingress
```
```
kubectl -n ingress-ns describe ingress rewrite
```

---

#### Verify Ingress Connectivity
Store the ingress public IP in a variable:
```
INGRESS_IP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INGRESS_IP
```
Test the httpd app (expect `It works!`):
```
curl -v http://$INGRESS_IP/
```
Test the nginx app (expect `Welcome to nginx!`):
```
curl -v http://$INGRESS_IP/test
```

You can also open these in a browser (plain **http**, no port needed):
> http://&lt;INGRESS-PUBLIC-IP&gt;/  → httpd
> http://&lt;INGRESS-PUBLIC-IP&gt;/test  → nginx
> e.g. http://20.121.45.10/test

> **Why no NodePort / node public IP?** AKS nodes have no public IPs by default and sit behind the Azure Load Balancer. The LoadBalancer service is the AKS-native way to reach the ingress controller. Only use NodePort if your cluster has no LoadBalancer integration (e.g. a self-managed cluster).

---

## Task 2 – Cleanup all resources created above
Delete the application namespace (removes the deployments, services and ingress in it):
```
kubectl delete ns ingress-ns
```
Delete the ingress controller **only if this lab installed it** (also removes the Azure public IP / LB rule it created):
```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```
Verify the cleanup (the existing AKS cluster itself is **not** deleted):
```
kubectl get ns
```
