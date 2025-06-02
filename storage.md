Kubernetes storage can be a complex topic, but understanding the core concepts and common commands will help you manage your stateful applications effectively. Here's a cheatsheet to help you navigate Kubernetes storage:

---

## Kubernetes Storage Cheatsheet

### 1. Core Concepts

* **Volume:** The most basic unit of storage in Kubernetes. A volume is a directory accessible to all containers within a Pod. Its lifecycle is tied to the Pod's lifecycle. If the Pod dies, the data in the volume is lost.
    * **Ephemeral Volumes:**
        * `emptyDir`: A simple, empty directory created when a Pod is scheduled and deleted when the Pod terminates. Useful for temporary data, caching, or sharing files between containers in the same Pod.
        * `configMap`: Mounts ConfigMap data as files into a Pod.
        * `secret`: Mounts Secret data as files into a Pod for sensitive information.
        * `downwardAPI`: Makes Pod and cluster information available to containers.
        * `projected`: Allows mounting several volume sources (Secret, ConfigMap, DownwardAPI) into the same directory.
    * **Persistent Volumes (PVs):** A piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned. It's a cluster resource, independent of any Pod. PVs are for persistent data.
* **PersistentVolumeClaim (PVC):** A request for storage by a user. It's similar to a Pod's request for CPU and memory. PVCs consume PV resources. They abstract away the underlying storage details from the application.
* **StorageClass:** Provides a way for administrators to describe the "classes" of storage they offer (e.g., fast SSD, slow HDD, backup policy). When a PVC requests a specific StorageClass, Kubernetes can dynamically provision a PV that matches the criteria.
* **Provisioner:** The component (often a CSI driver) that creates the actual storage resource when dynamic provisioning is used.

### 2. Access Modes

How a volume can be mounted by one or more Pods:

* **`ReadWriteOnce` (RWO):** The volume can be mounted as read-write by a single node.
* **`ReadOnlyMany` (ROX):** The volume can be mounted as read-only by many nodes.
* **`ReadWriteMany` (RWX):** The volume can be mounted as read-write by many nodes. (Often supported by file storage like NFS, not typically by block storage).
* **`ReadWriteOncePod` (RWOP):** (Alpha in recent Kubernetes versions) The volume can be mounted as read-write by a single Pod. This is stricter than RWO, which allows multiple Pods on the same node.

### 3. Persistent Volume Reclaim Policies

What happens to the underlying storage when a PVC is deleted:

* **`Retain`:** The PV and its data remain intact. The administrator must manually delete the underlying storage. Useful for preserving data.
* **`Delete`:** The PV and the underlying storage asset (e.g., AWS EBS volume) are automatically deleted.
* **`Recycle`:** (Deprecated) Attempts to "scrub" the data from the volume and make it available for reuse. Not recommended due to security concerns.

### 4. Common `kubectl` Commands for Storage

#### Persistent Volumes (PVs)

* **List PVs:**
    ```bash
    kubectl get pv
    ```
* **Describe a PV:**
    ```bash
    kubectl describe pv <pv-name>
    ```
* **Create a PV from a YAML file:**
    ```bash
    kubectl apply -f pv.yaml
    ```
* **Delete a PV:**
    ```bash
    kubectl delete pv <pv-name>
    ```

#### Persistent Volume Claims (PVCs)

* **List PVCs:**
    ```bash
    kubectl get pvc
    ```
* **List PVCs in all namespaces:**
    ```bash
    kubectl get pvc --all-namespaces
    ```
* **Describe a PVC:**
    ```bash
    kubectl describe pvc <pvc-name>
    ```
* **Create a PVC from a YAML file:**
    ```bash
    kubectl apply -f pvc.yaml
    ```
* **Delete a PVC:**
    ```bash
    kubectl delete pvc <pvc-name>
    ```

#### StorageClasses

* **List StorageClasses:**
    ```bash
    kubectl get sc
    kubectl get storageclass
    ```
* **Describe a StorageClass:**
    ```bash
    kubectl describe sc <storageclass-name>
    ```
* **Create a StorageClass from a YAML file:**
    ```bash
    kubectl apply -f storageclass.yaml
    ```
* **Delete a StorageClass:**
    ```bash
    kubectl delete sc <storageclass-name>
    ```

### 5. YAML Examples

#### Minimal PersistentVolume (Static Provisioning)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data" # For demonstration; use a more robust storage solution in production
```

#### Minimal PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual # Matches the storageClassName in the PV
```

#### Minimal StorageClass (for Dynamic Provisioning)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-ssd
provisioner: kubernetes.io/aws-ebs # Example for AWS EBS; depends on your cloud provider/CSI driver
parameters:
  type: gp2 # Example parameter for AWS EBS
reclaimPolicy: Delete
volumeBindingMode: Immediate # Or WaitForFirstConsumer for more advanced scenarios
```

#### Pod Using a PersistentVolumeClaim

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc # References the PVC created above
```

### 6. Key Considerations

* **StatefulSets:** For stateful applications (like databases) that require stable, unique network identifiers, stable persistent storage, and ordered graceful deployment/scaling/deletion, use StatefulSets in conjunction with PVCs.
* **Container Storage Interface (CSI):** The standard for exposing arbitrary block and file storage systems to containerized workloads on Kubernetes. Most modern storage solutions for Kubernetes leverage CSI drivers.
* **Local Persistent Volumes:** Allows you to use local disk storage on a node as a PersistentVolume. Useful for distributed databases that want to leverage local disk performance, but requires careful management of node affinity/anti-affinity.
* **Volume Snapshots:** Creates a point-in-time copy of a PersistentVolume.
* **Volume Cloning:** Creates a new volume from an existing volume.

This cheatsheet provides a quick reference to Kubernetes storage concepts and commands. For in-depth understanding, always refer to the official Kubernetes documentation.