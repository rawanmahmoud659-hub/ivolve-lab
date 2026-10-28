# Lab 13: Persistent Storage Setup for Application Logging

## Objective
Configure persistent storage in Kubernetes for application logging by creating a Persistent Volume (PV) and a Persistent Volume Claim (PVC) without using a StorageClass.

## Concepts

- **Persistent Volume (PV):** A cluster-level resource representing available storage.
- **Persistent Volume Claim (PVC):** A request for storage by an application within a namespace.
- **hostPath:** Uses a directory on the node's local filesystem as storage.
- **ReadWriteMany (RWX):** Allows the volume to be mounted for read/write access by multiple nodes when supported by the storage backend.
- **Retain:** Keeps the PV and its underlying data when the PVC is deleted.
- **StorageClass:** Left unset to prevent dynamic provisioning.

## Requirements

| Property | Value |
|---|---|
| PV Name | `ivolve-pv` |
| PVC Name | `ivolve-pvc` |
| Namespace | `ivolve` |
| Storage Capacity | `1Gi` |
| Storage Type | `hostPath` |
| Host Path | `/mnt/app-logs` |
| Access Mode | `ReadWriteMany` |
| Reclaim Policy | `Retain` |
| StorageClass | None |

**Note:** Kubernetes requires PV and PVC names to use lowercase DNS-compatible names. Therefore, `ivolve-pv` and `ivolve-pvc` are used instead of the uppercase names in the lab description.

## Step 1: Prepare the Storage Directory

Create the directory on the appropriate Kubernetes node:

```bash
sudo mkdir -p /mnt/app-logs
```

Verify the directory:

```bash
ls -ld /mnt/app-logs
```

## Step 2: Create the Persistent Volume

Create `ivolve-pv.yaml` with the following configuration:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ivolve-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /mnt/app-logs
    type: DirectoryOrCreate
```

Apply the configuration:

```bash
kubectl apply -f ivolve-pv.yaml
```

Verify the PV:

```bash
kubectl get pv ivolve-pv
kubectl describe pv ivolve-pv
```

The PV should initially have the `Available` status before being claimed.

## Step 3: Create the Persistent Volume Claim

Create `ivolve-pvc.yaml` with the following configuration:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ivolve-pvc
  namespace: ivolve
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: ivolve-pv
  resources:
    requests:
      storage: 1Gi
```

Apply the configuration:

```bash
kubectl apply -f ivolve-pvc.yaml
```

Verify the PVC:

```bash
kubectl get pvc ivolve-pvc -n ivolve
kubectl describe pvc ivolve-pvc -n ivolve
```

## Step 4: Verify PV and PVC Binding

Check the PV:

```bash
kubectl get pv ivolve-pv
```

Check the PVC:

```bash
kubectl get pvc ivolve-pvc -n ivolve
```

Expected results:

- PV status: `Bound`
- PVC status: `Bound`
- Capacity: `1Gi`
- Access mode: `RWX`
- Reclaim policy: `Retain`
- StorageClass: None

The `Bound` status confirms that the PVC has successfully claimed the PV.

## Important Note

The `hostPath` volume uses the local filesystem of a Kubernetes node. Although the PV and PVC specify `ReadWriteMany`, `hostPath` does not provide shared storage across multiple nodes. For applications running on different nodes that require shared logs, a shared storage backend is needed.

## Conclusion

Successfully created a 1Gi Persistent Volume and a matching Persistent Volume Claim using `hostPath`, configured `ReadWriteMany` access mode and the `Retain` reclaim policy, and verified that the PV and PVC reached the `Bound` state without using a StorageClass.
