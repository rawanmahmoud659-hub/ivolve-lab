# Lab 12: Managing Configuration and Sensitive Data with ConfigMaps and Secrets

## Objective
The goal of this lab is to manage application configuration and sensitive credentials in Kubernetes using ConfigMaps and Secrets, and configure a MySQL StatefulSet to use them.

## 1. Create the Namespace
The `ivolve` namespace is used to organize the lab resources.

```bash
kubectl create namespace ivolve
```

> If the namespace already exists, there is no need to create it again.

## 2. Create the ConfigMap

A ConfigMap stores non-sensitive configuration data separately from the application configuration.

The ConfigMap is named `ivolve-cm` and contains:

- `DB_HOST`: The MySQL Headless Service DNS name.
- `DB_USER`: The application database username.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ivolve-cm
  namespace: ivolve
data:
  DB_HOST: mysql-h.ivolve.svc.cluster.local
  DB_USER: ivolveuser
```

Save the file as `configmap.yaml`, then apply it:

```bash
kubectl apply -f configmap.yaml
```

Verify the ConfigMap:

```bash
kubectl get configmap -n ivolve
kubectl describe configmap ivolve-cm -n ivolve
```

## 3. Create the Secret

A Secret stores sensitive information such as database passwords.

The Secret is named `ivolve-secret` and contains:

- `MYSQL_ROOT_PASSWORD`: The MySQL root password.
- `DB_PASSWORD`: The application database user's password.

Example `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ivolve-secret
  namespace: ivolve
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: MTIzNA==
  DB_PASSWORD: MTIzNA==
```

The example values above are Base64-encoded representations of `1234`, matching the lab configuration used here.

Apply the Secret:

```bash
kubectl apply -f secret.yaml
```

Verify its existence and keys:

```bash
kubectl get secret -n ivolve
kubectl describe secret ivolve-secret -n ivolve
```

> **Security note:** Base64 encoding is not encryption. Do not use weak passwords such as `1234` in production.

## 4. Create the MySQL Headless Service and StatefulSet

The Headless Service provides a stable DNS name for the MySQL StatefulSet. The StatefulSet manages the MySQL Pod.

The `mysql.yaml` file contains the following resources:

- **Service:** `mysql-h`, with `clusterIP: None` and port `3306`.
- **StatefulSet:** `mysql`, with one replica using the `mysql:8.0` image.
- **Environment variables:** MySQL credentials are loaded from `ivolve-secret`.
- **Database initialization:** Creates the `ivolve` database and the `ivolveuser` application user.
- **Volume:** Uses `emptyDir` mounted at `/var/lib/mysql` for temporary storage.

Apply the manifest:

```bash
kubectl apply -f mysql.yaml
```

Because the worker node has the taint `workload=ivolveApp:NoSchedule`, the MySQL Pod template includes this toleration:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "ivolveApp"
    effect: "NoSchedule"
```

This allows MySQL to be scheduled on the worker node without removing the node's taint.

## 5. Verify the Deployment

Check the StatefulSet, Service, and Pods:

```bash
kubectl get statefulset,service,pods -n ivolve
```

Watch the Pods until MySQL is ready:

```bash
kubectl get pods -n ivolve -w
```

Expected result: `mysql-0` reaches `Running` with `1/1` containers ready.

Check the MySQL startup logs:

```bash
kubectl logs mysql-0 -n ivolve --tail=30
```

Look for the message:

```text
ready for connections
```

## 6. Test the Database Connection

Connect to MySQL using the application database user:

```bash
kubectl exec -it mysql-0 -n ivolve -- mysql -u ivolveuser -p ivolve
```

Enter the password stored in `DB_PASSWORD` when prompted.

Inside the MySQL shell, run:

```sql
SHOW DATABASES;
```

Verify that the `ivolve` database exists, then exit:

```sql
exit;
```

## 7. Final Verification Commands

```bash
kubectl get configmap,secret -n ivolve
kubectl describe configmap ivolve-cm -n ivolve
kubectl describe secret ivolve-secret -n ivolve
kubectl get statefulset,service,pods -n ivolve
kubectl logs mysql-0 -n ivolve --tail=30
```

## Results

- Created a ConfigMap for non-sensitive database configuration.
- Created a Secret for MySQL credentials.
- Deployed MySQL using a StatefulSet.
- Created a Headless Service named `mysql-h`.
- Configured the MySQL Pod with a toleration to schedule it on the tainted worker node.
- Verified that MySQL started successfully and became ready for connections.
- Prepared a database connection test using
