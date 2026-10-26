# Lab 11: Namespace Management and Resource Quota Enforcement

## 1. Lab Overview

The goal of this lab is to understand how Kubernetes Namespaces organize application resources and how ResourceQuotas enforce resource limits within a namespace.

In this lab, we:
- Created a namespace named `ivolve`.
- Created a ResourceQuota named `ivolvequota`.
- Set the maximum number of Pods to 3.
- Created three Nginx Pods and verified that they were running.
- Attempted to create a fourth Pod and confirmed that Kubernetes rejected it because the quota limit had been reached.

## 2. Prerequisites

- A running Kubernetes cluster.
- `kubectl` installed and configured.
- Access to the Kubernetes control-plane node.
- A worker node available for running application Pods.

Check the cluster nodes:

```bash
kubectl get nodes
```

## 3. Create the Namespace

A Namespace provides logical isolation and organization for Kubernetes resources.

Create the `ivolve` namespace:

```bash
kubectl create namespace ivolve
```

Verify the namespace:

```bash
kubectl get namespaces
```

Check its status:

```bash
kubectl get namespace ivolve
```

Expected result: the namespace status should be `Active`.

## 4. Create the ResourceQuota

A ResourceQuota limits the amount or number of resources that can be created within a namespace.

Create a file named `resourcequota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ivolvequota
  namespace: ivolve
spec:
  hard:
    pods: "3"
```

**Configuration explanation:**

- `apiVersion: v1`: Uses the Kubernetes core API.
- `kind: ResourceQuota`: Defines a ResourceQuota resource.
- `metadata.name`: Sets the quota name to `ivolvequota`.
- `metadata.namespace`: Applies the quota to the `ivolve` namespace.
- `spec.hard.pods: "3"`: Limits the total number of Pods in the namespace to three.

Note: Kubernetes resource names must use lowercase letters, so the quota name is `ivolvequota` rather than `ivolveQuota`.

Apply the configuration:

```bash
kubectl apply -f resourcequota.yaml
```

Verify the quota:

```bash
kubectl get quota -n ivolve
```

Display detailed quota information:

```bash
kubectl describe quota ivolvequota -n ivolve
```

Initially, the expected quota usage is:

```text
Resource    Used  Hard
--------    ----  ----
pods        0     3
```

## 5. Create the Nginx Pods

Three Pods were created in the `ivolve` namespace to test the quota.

### Pod 1

Create `pod1.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: ivolve
spec:
  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "ivolveApp"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:alpine
```

Apply the configuration:

```bash
kubectl apply -f pod1.yaml
```

### Pod 2

Create `pod2.yaml` using the same configuration, changing `metadata.name` to `pod2`.

Apply it:

```bash
kubectl apply -f pod2.yaml
```

### Pod 3

Create `pod3.yaml` using the same configuration, changing `metadata.name` to `pod3`.

Apply it:

```bash
kubectl apply -f pod3.yaml
```

**Why are tolerations included?**

The worker node has the taint `workload=ivolveApp:NoSchedule`. The Pods include a matching toleration so they can be scheduled on that worker node.

## 6. Verify the Pods

List the Pods in the namespace:

```bash
kubectl get pods -n ivolve
```

Successful result:

```text
NAME   READY   STATUS
pod1   1/1     Running
pod2   1/1     Running
pod3   1/1     Running
```

The `Running` status indicates that the Pods are running. The `1/1` READY value indicates that each Pod's single container is ready.

Verify the quota usage:

```bash
kubectl describe quota ivolvequota -n ivolve
```

Result:

```text
Resource    Used  Hard
--------    ----  ----
pods        3     3
```

This confirms that three Pods are counted against the quota and the maximum allowed is three.

## 7. Test ResourceQuota Enforcement

To verify that Kubernetes prevents the creation of additional Pods, create `pod4.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod4
  namespace: ivolve
spec:
  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "ivolveApp"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:alpine
```

Attempt to create the fourth Pod:

```bash
kubectl apply -f pod4.yaml
```

Expected result:

```text
Error from server (Forbidden): error when creating "pod4.yaml":
pods "pod4" is forbidden: exceeded quota: ivolvequota,
requested: pods=1, used: pods=3, limited: pods=3
```

**Explanation:**

- `requested: pods=1`: The new Pod would consume one additional Pod slot.
- `used: pods=3`: Three Pods already count toward the quota.
- `limited: pods=3`: The namespace allows a maximum of three Pods.
- `Forbidden`: Kubernetes rejected the request because creating another Pod would exceed the quota.

Verify that only three Pods exist:

```bash
kubectl get pods -n ivolve
```

Verify the quota again:

```bash
kubectl describe quota ivolvequota -n ivolve
```

The quota remains at `3/3`, and the fourth Pod is not created.

## 8. Important Commands Summary

| Command | Purpose |
|---|---|
| `kubectl get nodes` | Lists cluster nodes |
| `kubectl get namespaces` | Lists namespaces |
| `kubectl create namespace ivolve` | Creates the namespace |
| `kubectl apply -f resourcequota.yaml` | Creates the quota |
| `kubectl get quota -n ivolve` | Lists quotas |
| `kubectl describe quota ivolvequota -n ivolve` | Shows quota usage and limits |
| `kubectl apply -f pod1.yaml` | Creates the first Pod |
| `kubectl get pods -n ivolve` | Lists Pods in the namespace |
| `kubectl apply -f pod4.yaml` | Tests whether the quota blocks a fourth Pod |

## 9. Conclusion

This lab successfully demonstrated Kubernetes Namespace management and ResourceQuota enforcement.

The `ivolve` namespace was created, and the `ivolvequota` ResourceQuota was configured to allow a maximum of three Pods. Three Nginx Pods were successfully scheduled and reached the `Running` state.

When a fourth Pod was submitted, Kubernetes rejected it with a `Forbidden` error because the namespace had already reached its quota of three Pods.

**Result: Lab 11 completed successfully.**
