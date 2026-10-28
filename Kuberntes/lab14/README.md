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
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ivolve-secret
                  key: DB_PASSWORD
            - name: MYSQL_DATABASE
              value: ivolve
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes:
          - ReadWriteMany
        storageClassName: ""
        resources:
          requests:
            storage: 1Gi
```

### Configuration explanation

- **Replicas:** Runs one MySQL pod.
- **ConfigMap:** Supplies the `DB_USER` environment value.
- **Secret:** Supplies the root password and database user's password.
- **Node selector:** Schedules the pod on a node labeled `project=ivolve`.
- **Toleration:** Allows the pod to schedule on a node with the `workload=ivolveApp:NoSchedule` taint.
- **Volume mount:** Mounts database storage at `/var/lib/mysql`.
- **Volume claim template:** Requests a PVC for the StatefulSet pod.

> **Storage note:** This PVC template requires a compatible available PV or a StorageClass that can provision storage. The existing `ivolve-pv` uses `/mnt/app-logs`; verify that this path is suitable for MySQL before using it. Do not assume the PVC is bound until its status is `Bound`.

Apply the manifest:

```bash
kubectl apply -f ivolvedb.yaml
```

Check the StatefulSet and PVC:

```bash
kubectl get statefulset -n ivolve
kubectl get pods -n ivolve -o wide
kubectl get pvc -n ivolve
kubectl describe pvc mysql-data-ivolvedb-0 -n ivolve
```

## 4. Create a Headless Service

A Headless Service allows Kubernetes DNS to resolve directly to the StatefulSet pod instead of using a regular virtual ClusterIP.

Create a file named `ivolvedbsvc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ivolvedbsvc
  namespace: ivolve
spec:
  clusterIP: None
  selector:
    app: ivolvedb
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
      protocol: TCP
```

Apply the manifest:

```bash
kubectl apply -f ivolvedbsvc.yaml
```

Verify the Service and its endpoints:

```bash
kubectl get svc ivolvedbsvc -n ivolve
kubectl get endpoints ivolvedbsvc -n ivolve
kubectl get endpointslices -n ivolve
```

The Service should have `CLUSTER-IP` set to `None`.

## 5. Verify MySQL Pod Status

Check whether the pod is running and ready:

```bash
kubectl get pods -n ivolve -o wide
kubectl describe pod ivolvedb-0 -n ivolve
kubectl logs ivolvedb-0 -n ivolve
```

The pod should be `Running` and show `1/1` containers ready.

If the pod is pending, inspect its events and the PVC status before continuing.

## 6. Connect to MySQL

Connect to MySQL using the root password stored in the Secret:

```bash
kubectl exec -it ivolvedb-0 -n ivolve -- \
  sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD"'
```

Inside the MySQL client, run:

```sql
SHOW DATABASES;
USE ivolve;
SELECT 1;
EXIT;
```

A successful connection and query confirm that MySQL is responding.

If the MySQL client is unavailable inside the container, use a separate MySQL client pod or install a client in an appropriate temporary container.

## 7. Verify MySQL DNS Resolution

The expected fully qualified DNS name for the StatefulSet pod is:

```text
ivolvedb-0.ivolvedbsvc.ivolve.svc.cluster.local
```

Run a DNS lookup from the MySQL pod:

```bash
kubectl exec ivolvedb-0 -n ivolve -- \
  getent hosts ivolvedb-0.ivolvedbsvc.ivolve.svc.cluster.local
```

If `getent` is unavailable, use a temporary diagnostic pod with DNS utilities:

```bash
kubectl run dns-test -n ivolve \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- nslookup ivolvedb-0.ivolvedbsvc.ivolve.svc.cluster.local
```

Check its output:

```bash
kubectl logs dns-test -n ivolve
kubectl delete pod dns-test -n ivolve
```

DNS resolution should return the IP address of the StatefulSet pod. The pod must be running and selected by the Headless Service for the expected record to resolve.

## 8. Final Verification

Run the following commands to review the deployed resources:

```bash
kubectl get statefulset -n ivolve
kubectl get pods -n ivolve -o wide
kubectl get svc -n ivolve
kubectl get pvc -n ivolve
kubectl get pv
kubectl get endpoints ivolvedbsvc -n ivolve
```

### Expected results

- The StatefulSet has one desired replica.
- The MySQL pod is `Running` and `Ready`.
- The Headless Service has `CLUSTER-IP: None`.
- The PVC is `Bound` to compatible persistent storage.
- MySQL accepts a client connection and executes queries.
- The pod's fully qualified DNS name resolves inside the cluster.

## 9. Important Notes

- Kubernetes resource names must be lowercase, so `ivolvedb` and `ivolvedbsvc` are used in the manifests.
- ConfigMap and Secret names and keys must match the existing resources exactly.
- Do not store database passwords directly in the README or commit Secret values to Git.
- A successful `kubectl apply` does not prove that the database is operational; verify pod readiness, storage binding, database connectivity, and DNS separately.
