## Dynamic Provisioning

## Prerequisites
* AWS IAM permissions (i.e 'AdministratorAccess') for managing EBS volumes are configured for the cluster nodes (all nodes).


### Task 1: Install the AWS EBS CSI Driver
The EBS CSI driver enables Kubernetes to interact with AWS EBS for provisioning, attaching, and managing persistent storage dynamically.

Install the AWS EBS CSI driver by using below command
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.61"
```
`Note :` For more detail visit the below page

https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md


Verify Installation
```bash
kubectl get pods -A | grep ebs
```
Ensure the pods related to aws-ebs-csi-driver are running successfully.

### Task 2: Create a Storage Class 

A StorageClass in Kubernetes is a powerful abstraction layer that defines how storage resources (like AWS EBS, Google Persistent Disks, Azure Disks, or others) should be provisioned and managed dynamically. It simplifies and automates the creation and management of persistent storage in a Kubernetes cluster by providing predefined policies for storage provisioning.


```
vi storageclass.yaml
```

Add the given content, by pressing `INSERT`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```
save the file using `ESCAPE + :wq!`

Apply the storageclass definition yaml
```
kubectl apply -f storageclass.yaml
```
Verify the Storage Class
```bash
kubectl get sc
```

### Task 3: Create a PersistentVolumeClaim (PVC) using below yaml

```
vi pvc.yaml
```

Add the given content, by pressing `INSERT`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ebs-storage
```
save the file using `ESCAPE + :wq!`

Apply the definition yaml
```
kubectl apply -f pvc.yaml
```

Verify the PVC
```bash
kubectl get pvc
```
Ensure the PVC is Bound to a dynamically provisioned PV.


### Task 4: Create a Pod that Uses the PVC

```
vi pod.yaml
```

Add the given content, by pressing `INSERT`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-dynamic-app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: ebs-volume
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
      claimName: ebs-dynamic-pvc
```
save the file using `ESCAPE + :wq!`

Apply the definition yaml
```
kubectl apply -f pod.yaml
```

Verify the POD
```bash
kubectl get po
```
Ensure the Pod is running and the volume is mounted successfully.

### Task 5: Test the Persistent Volume

Access the Pod
```bash
kubectl exec -it ebs-dynamic-app -- bash
```

Navigate to the Mounted Directory
```bash
cd /usr/share/nginx/html
```

Create a Test File
```bash
echo "Hello from EBS-backed storage!" > index.html
```

Exit the Pod
```bash
exit
```

### Task 6: Cleanup the resources
Delete the pod

```bash
kubectl delete -f pod.yaml
```
Delete the pvc

```bash
kubectl delete -f pvc.yaml
```
Delete the storageclass

```bash
kubectl delete -f storageclass.yaml
```



