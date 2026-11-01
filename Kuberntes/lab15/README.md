# Lab 15: Kubernetes Application Deployment with ClusterIP Service

## Objective

Deploy a Node.js application on Kubernetes using:
- A Deployment with 2 replicas.
- A custom Docker Hub image.
- ConfigMap and Secret for environment variables.
- Node Selector and Toleration.
- Persistent Volume and Persistent Volume Claim.
- MySQL Init Container to create the database and configure application privileges.
- ClusterIP Service to expose the application internally.

## 1. Prerequisites

- Kubernetes cluster with one control-plane node and one worker node.
- Namespace: `ivolve`.
- Docker Hub image: `rawanm21/ivolve-app:latest`.
- Existing ConfigMap: `ivolve-cm`.
- Existing Secret: `ivolve-secret`.
- Existing PV: `ivolve-pv`.
- Existing PVC: `mysql-data-ivolvedb-0`.

## 2. Configure the Worker Node

The application must run on a node with the label `project=ivolve`.

```bash
kubectl label node ip-172-31-4-135 project=ivolve
```

Verify:

```bash
kubectl get nodes --show-labels
```

## 3. Create the Application Deployment

Create `deployment.yaml`.

The Deployment configuration includes:

- Deployment name: `ivolveapp`.
- Namespace: `ivolve`.
- Replicas: 2.
- Image: `rawanm21/ivolve-app:latest`.
- Container port: 3000.
- Node Selector: `project=ivolve`.
- Toleration: `workload=ivolveApp`, effect `NoSchedule`.
- ConfigMap: `ivolve-cm`.
- Secret: `ivolve-secret`.
- Persistent Volume Claim: `mysql-data-ivolvedb-0`.
- Init Container: `mysql-init`.

The Init Container uses the MySQL client image to:

1. Wait for the MySQL server.
2. Create the `ivolve` database if it does not exist.
3. Create the application user.
4. Configure the application user's password.
5. Grant privileges on the `ivolve` database.

Apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Check the rollout:

```bash
kubectl rollout status deployment/ivolveapp -n ivolve
```

Check the Deployment and Pods:

```bash
kubectl get deployment ivolveapp -n ivolve
kubectl get pods -n ivolve -l app=ivolveapp -o wide
```

## 4. Configure the ClusterIP Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ivolvaappsvc
  namespace: ivolve
spec:
  type: ClusterIP
  selector:
    app: ivolveapp
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      protocol: TCP
```

Apply the Service:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get svc ivolvaappsvc -n ivolve
kubectl get endpoints ivolvaappsvc -n ivolve
```

## 5. Verify the Application

Check application logs:

```bash
kubectl logs -n ivolve -l app=ivolveapp --tail=30
```

Expected output:

```text
Application running on port 3000
```

Check the Deployment image:

```bash
kubectl get deployment ivolveapp -n ivolve \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expected:

```text
rawanm21/ivolve-app:latest
```

## 6. Verify MySQL Database and Privileges

Create a temporary MySQL client Pod using the ConfigMap and Secret.

The Pod must use the same node selector and toleration to schedule on the worker node.

Apply the test Pod:

```bash
kubectl apply -f mysql-verify.yaml
```

Wait until it is ready:

```bash
kubectl wait --for=condition=Ready pod/mysql-verify \
  -n ivolve --timeout=180s
```

Connect to MySQL:

```bash
kubectl exec -n ivolve mysql-verify -- sh -c \
'mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" ivolve -e "SELECT DATABASE(); SHOW TABLES; SHOW GRANTS FOR CURRENT_USER();"'
```

Verified database:

```text
DATABASE()
ivolve
```

Verified application user:

```text
ivolveuser@%
```

Verified privileges:

```text
GRANT USAGE ON *.* TO `ivolveuser`@`%`
GRANT ALL PRIVILEGES ON `ivolve`.* TO `ivolveuser`@`%`
```

Delete the temporary verification Pod:

```bash
kubectl delete pod mysql-verify -n ivolve
```

## 7. Final Verification

```bash
kubectl get deployment ivolveapp -n ivolve
kubectl get pods -n ivolve -l app=ivolveapp -o wide
kubectl get svc -n ivolve
kubectl get endpoints ivolvaappsvc -n ivolve
kubectl get pv
kubectl get pvc -n ivolve
kubectl logs -n ivolve -l app=ivolveapp --tail=30
```

## 8. Final Results

| Component | Verified Result |
|---|---|
| Deployment | 2/2 Available |
| Application Pods | 2 Running |
| Docker Image | `rawanm21/ivolve-app:latest` |
| Init Container | Completed |
| Node Selector | `project=ivolve` |
| Toleration | `workload=ivolveApp:NoSchedule` |
| ConfigMap | Connected |
| Secret | Connected |
| PV | `ivolve-pv`, 5Gi, Bound |
| PVC | `mysql-data-ivolvedb-0`, Bound |
| Service | `ivolvaappsvc`, ClusterIP |
| Service Endpoints | 2 application Pods |
| MySQL Database | `ivolve` |
| Application User | `ivolveuser` |
| Database Privileges | Verified |

## Conclusion

Successfully deployed a Node.js application on Kubernetes using a two-replica Deployment and an internal ClusterIP Service.

The Init Container successfully configured the MySQL database and application user privileges. The application Pods became Running and Ready, and the Service correctly discovered both application endpoints.

The MySQL client verification confirmed that the application user can connect to the `ivolve` database and has the required privileges.

