# Lab 14: StatefulSet with Headless Service

## 1. Lab Objective

The objective of this lab is to deploy a MySQL database using a Kubernetes StatefulSet in the `ivolve` namespace. The database uses environment variables from a ConfigMap and Secret, persistent storage, node selection, and tolerations.

The lab also creates a Headless Service to provide stable network identities and verifies database connectivity and DNS resolution inside the cluster.

## 2. Existing Kubernetes Resources

The following resources are used by this lab:

- **Namespace:** `ivolve`
- **ConfigMap:** `ivolve-cm`
- **Secret:** `ivolve-secret`
- **Worker node label:** `project=ivolve`
- **Worker node taint:** `workload=ivolveApp:NoSchedule`
- **Persistent Volume:** `ivolve-pv`

Check the existing resources:

```bash
kubectl get nodes --show-labels
kubectl get namespace
kubectl get configmap -n ivolve
kubectl get secrets -n ivolve
kubectl get pv
kubectl get pvc -n ivolve
```

## 3. Configure the StatefulSet

A StatefulSet manages stateful applications such as databases. It provides stable pod identities and works with volume claim templates to request persistent storage.

Create a file named `ivolvedb.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ivolvedb
  namespace: ivolve
spec:
  serviceName: ivolvedbsvc
  replicas: 1
  selector:
    matchLabels:
      app: ivolvedb
  template:
    metadata:
      labels:
        app: ivolvedb
    spec:
      nodeSelector:
        project: ivolve
      tolerations:
        - key: workload
          operator: Equal
          value: ivolveApp
          effect: NoSchedule
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolve-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_USER
              valueFrom:
                configMapKeyRef:
                  name: ivolve-cm
                  key: DB_USER
            -
