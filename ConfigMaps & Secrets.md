# Lab: ConfigMaps & Secrets on AKS (Azure Kubernetes Service)

## Prerequisites – Connect to the existing AKS cluster

Log in to Azure (or use **Azure Cloud Shell**, which already has `az`, `kubectl` and `vi`):
```
az login
```
```
az account set --subscription <subscription-name-or-id>
```
Find your cluster and connect:
```
az aks list -o table
```
```
az aks get-credentials --resource-group <your-resource-group> --name <your-aks-cluster-name>
```
```
kubectl get nodes
```

Create a dedicated namespace for this lab so nothing collides with other workloads on the shared cluster, and make it the default namespace for your `kubectl` commands:
```
kubectl create ns cm-lab
```
```
kubectl config set-context --current --namespace=cm-lab
```
```
kubectl config view --minify | grep namespace
```

> All commands below run in the `cm-lab` namespace. Each task ends with a **Reset** step: a ConfigMap name can't be created twice, and a running pod's `env` can't be changed in place.

---

## Task 1: Inject variables directly (traditional method)
```
vi env.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: ws
  name: env-pod
spec:
  containers:
  - image: nginx
    name: ng-ctr
    ports:
    - containerPort: 80
    env:
    - name: db_user   # key
      value: admin    # value
    - name: db_pwd
      value: "1234"   # numbers must be quoted
```
```
kubectl apply -f env.yaml
```
```
kubectl get pod env-pod
```
```
kubectl describe pod env-pod
```
Open a shell in the pod and check that the variables were passed in:
```
kubectl exec -it env-pod -- sh
```
```
echo $db_user
```
```
echo $db_pwd
```
```
env | grep db_
```
```
exit
```
**Reset:**
```
kubectl delete pod env-pod
```

---

## Task 2: Inject `ALL` variables from a ConfigMap (from literal)
Create a ConfigMap:
```
kubectl create cm cm-1 --from-literal=db_user=admin --from-literal=db_pwd=1234
```
```
kubectl get cm
```
```
kubectl describe cm cm-1
```
Reference the ConfigMap in the pod YAML:
```
vi env.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
  name: web-pod
spec:
  containers:
  - image: httpd
    name: ctr-1
    ports:
    - containerPort: 80
    envFrom:
    - configMapRef:
        name: cm-1
```
```
kubectl apply -f env.yaml
```
```
kubectl describe pod web-pod
```
Check the variables inside the pod:
```
kubectl exec -it web-pod -- sh
```
```
echo $db_user
```
```
echo $db_pwd
```
```
env | grep db_
```
```
exit
```
**Reset:**
```
kubectl delete pod web-pod
```
```
kubectl delete cm cm-1
```

---

## Task 3: Inject a `PARTICULAR` variable from a ConfigMap (from literal)
Create the ConfigMap again:
```
kubectl create cm cm-1 --from-literal=db_user=admin --from-literal=db_pwd=1234
```
```
kubectl describe cm cm-1
```
Inject only the `db_pwd` key, exposed in the pod under a new name, `db_password`:
```
vi env.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
  name: web-pod
spec:
  containers:
  - image: httpd
    name: ctr-1
    ports:
    - containerPort: 80
    env:
    - name: db_password      # variable name inside the pod
      valueFrom:
        configMapKeyRef:
          name: cm-1         # ConfigMap name
          key: db_pwd        # key in the ConfigMap
```
```
kubectl apply -f env.yaml
```
```
kubectl describe pod web-pod
```
Check inside the pod:
```
kubectl exec -it web-pod -- sh
```
```
echo $db_user
```
```
echo $db_pwd
```
```
echo $db_password
```
```
env | grep db_
```
```
exit
```
> **Expected:** `$db_user` and `$db_pwd` are **empty**; only `$db_password` is set (`1234`), because only that one key was injected.

**Reset:**
```
kubectl delete pod web-pod
```
```
kubectl delete cm cm-1
```

---

## Task 4: Inject variables from a ConfigMap (from file)
Create a file:
```
vi token
```
```
This is CKAD Training. We are practicing Injecting variables from ConfigMaps(FromFile) into POD.
```
Create the ConfigMap. The file name (`token`) becomes the key, and the file contents become the value:
```
kubectl create cm cm-1 --from-file=token
```
```
kubectl get cm
```
```
kubectl describe cm cm-1
```
Reference it in the pod YAML:
```
vi env.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
  name: web-pod
spec:
  containers:
  - image: httpd
    name: ctr-1
    ports:
    - containerPort: 80
    envFrom:
    - configMapRef:
        name: cm-1
```
```
kubectl apply -f env.yaml
```
```
kubectl describe pod web-pod
```
Check inside the pod:
```
kubectl exec -it web-pod -- sh
```
```
echo $token
```
```
env | grep token
```
```
exit
```
**Reset** (keep the `token` file for Task 5):
```
kubectl delete pod web-pod
```
```
kubectl delete cm cm-1
```

---

## Task 5: Inject a ConfigMap as a volume mount
Create the ConfigMap from the same `token` file:
```
kubectl create cm cm-1 --from-file=token
```
```
kubectl describe cm cm-1
```
Mount it as a volume. Each key becomes a file under `/app`:
```
vi env.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
  name: web-pod
spec:
  volumes:
  - name: cm-volume
    configMap:
      name: cm-1
  containers:
  - image: httpd
    name: ctr-1
    volumeMounts:
    - name: cm-volume
      mountPath: /app
    ports:
    - containerPort: 80
```
```
kubectl apply -f env.yaml
```
```
kubectl describe pod web-pod
```
Check inside the pod:
```
kubectl exec -it web-pod -- sh
```
```
ls /app
```
```
cat /app/token
```
```
exit
```
> **Bonus:** Unlike environment variables, a mounted ConfigMap updates automatically. Run `kubectl edit cm cm-1`, change the text, wait about 60 seconds, then run `kubectl exec web-pod -- cat /app/token` again.

**Reset:**
```
kubectl delete pod web-pod
```
```
kubectl delete cm cm-1
```

---

## Task 6: Secrets

### Imperative
```
kubectl create secret generic secret-1 --from-literal=db_user=admin --from-literal=db_pwd=123
```
```
kubectl get secret
```
```
kubectl describe secret secret-1
```
`describe` hides the values. View the stored (base64) data and decode one value:
```
kubectl get secret secret-1 -o yaml
```
```
kubectl get secret secret-1 -o jsonpath='{.data.db_pwd}' | base64 -d
```

### Declarative
Values under `data:` must be base64 encoded. Always use `echo -n`; without `-n` a trailing newline gets encoded into the secret:
```
echo -n 'root' | base64
```
```
echo -n 'user' | base64
```
```
echo -n 'mypwd' | base64
```
```
vi secret.yaml
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-credentials
type: Opaque
data:
  ## All values are base64 encoded
  ## Encode: echo -n '<value>' | base64
  ## Decode: echo '<encoded-value>' | base64 -d

  rootpw: cm9vdA==      # root
  user: dXNlcg==        # user
  password: bXlwd2Q=    # mypwd
```
> Tip: use `stringData:` instead of `data:` to write plain-text values and let Kubernetes encode them.

```
kubectl apply -f secret.yaml
```
```
kubectl get secrets
```
```
kubectl describe secret mysql-credentials
```

### Inject all values from both secrets into a pod
Secrets can be injected in the same three ways as ConfigMaps: `envFrom.secretRef`, `env.valueFrom.secretKeyRef`, or a `secret` volume.
```
vi sc-pod.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sc-pod
  name: sc-pod
spec:
  containers:
  - image: nginx
    name: sc-ctr
    envFrom:
    - secretRef:
        name: secret-1
    - secretRef:
        name: mysql-credentials
```
```
kubectl apply -f sc-pod.yaml
```
```
kubectl get po
```
```
kubectl exec -it sc-pod -- sh
```
```
echo $db_user
```
```
echo $db_pwd
```
```
echo $rootpw
```
```
echo $user
```
```
echo $password
```
```
env
```
```
exit
```
> **Note:** Inside the pod the values are plain text. Kubernetes only base64-encodes Secrets; it does not encrypt them. Anyone with `get secret` RBAC rights can read them. For production on AKS, keep secrets in **Azure Key Vault** and mount them with the **Secrets Store CSI Driver** add-on (`az aks enable-addons --addons azure-keyvault-secrets-provider -g <rg> -n <cluster>`).

---

## Task 7: Cleanup
Deleting the namespace deletes every pod, ConfigMap and Secret created in this lab:
```
kubectl delete ns cm-lab
```
Point your `kubectl` context back at the `default` namespace:
```
kubectl config set-context --current --namespace=default
```
Remove the local files:
```
rm -f env.yaml secret.yaml sc-pod.yaml token
```
Verify:
```
kubectl get ns
```
