# Lab 16: Kubernetes Pod Resource Management

## Objective

Configure CPU and memory resource requests and limits for the existing `ivolveapp` Deployment in the `ivolve` namespace.

## Resource Configuration

The application container is configured with:

| Resource | Request | Limit |
|---|---:|---:|
| CPU | 1 CPU | 2 CPUs |
| Memory | 1Gi | 2Gi |

## Deployment Configuration

The Deployment uses:

- Deployment name: `ivolveapp`
- Namespace: `ivolve`
- Replicas: 1
- Container image: `rawanm21/ivolve-app:latest`
- Node selector: `project: ivolve`
- Toleration: `workload=ivolveApp:NoSchedule`

The application also uses a MySQL initialization container, ConfigMap, Secret, and PersistentVolumeClaim.

## Implementation Steps

### 1. Configure Resource Requests and Limits

The following configuration is added to the application container:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "2Gi"
```

### 2. Apply the Deployment

```bash
kubectl apply -f deployment.yaml
```

### 3. Verify the Deployment

```bash
kubectl get deployment ivolveapp -n ivolve
kubectl get pods -n ivolve -o wide
```

### 4. Inspect Pod Resources

```bash
kubectl describe pod <pod-name> -n ivolve
```

Verify the CPU and memory values under `Requests` and `Limits`.

### 5. Monitor Resource Usage

```bash
kubectl top pod -n ivolve
kubectl top nodes
```

## Verification Results

The application Deployment reached `1/1` available replicas.

The application Pod reached `Running` and `Ready` status.

Verified resource configuration:

- CPU request: `1`
- Memory request: `1Gi`
- CPU limit: `2`
- Memory limit: `2Gi`

Observed application Pod usage during verification:

- CPU: `1m`
- Memory: `67Mi`

## Conclusion

The existing Kubernetes Deployment was updated with CPU and memory requests and limits. The Pod was successfully scheduled and reached Running status. Resource usage was monitored using Metrics Server and `kubectl top`.

