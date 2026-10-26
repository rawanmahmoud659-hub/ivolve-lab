# Lab 10: Node Isolation Using Taints and Labels in Kubernetes

## Objective
- Apply a taint to a worker node to prevent pods without a matching toleration from being scheduled on it.
- Label the same worker node for project identification and targeted pod scheduling.
- Verify the taint and label using Kubernetes commands.

## Concepts

**Taints:** Prevent pods from being scheduled on a node unless they have a matching toleration. The `NoSchedule` effect prevents new pods without a matching toleration from being scheduled on the node.

**Labels:** Key-value pairs attached to Kubernetes objects. Labels help identify and select nodes for targeted pod scheduling using mechanisms such as `nodeSelector`.

## Step 1: Verify the Cluster Nodes

List all nodes in the Kubernetes cluster:

```bash
kubectl get nodes
```

Identify the worker node to configure.

**Worker Node:** `ip-172-31-4-135`

## Step 2: Apply a Taint

Apply the required taint to the worker node:

```bash
kubectl taint nodes ip-172-31-4-135 workload=ivolveApp:NoSchedule
```

**Explanation:**
- `workload`: Taint key.
- `ivolveApp`: Taint value.
- `NoSchedule`: Prevents new pods without a matching toleration from being scheduled on the node.

Expected output:

```text
node/ip-172-31-4-135 tainted
```

## Step 3: Apply a Label

Add the project label to the same worker node:

```bash
kubectl label nodes ip-172-31-4-135 project=ivolve
```

**Explanation:**
- `project`: Label key.
- `ivolve`: Label value.

Expected output:

```text
node/ip-172-31-4-135 labeled
```

## Step 4: Verify the Taint and Label

Display the worker node's details:

```bash
kubectl describe node ip-172-31-4-135
```

Verify that the output contains:

```text
Taints: workload=ivolveApp:NoSchedule
```

Also verify that the labels include:

```text
project=ivolve
```

## Step 5: Verify All Node Labels

Display all nodes with their labels:

```bash
kubectl get nodes --show-labels
```

Confirm that the worker node `ip-172-31-4-135` has the label `project=ivolve`.

To display the specific project label:

```bash
kubectl get node ip-172-31-4-135 -L project
```

## Step 6: Verify the Taint Directly

Display the taint configuration:

```bash
kubectl get node ip-172-31-4-135 -o jsonpath='{.spec.taints}'
echo
```

Expected result includes:

```json
[{"effect":"NoSchedule","key":"workload","value":"ivolveApp"}]
```

## Results

- Successfully applied the `workload=ivolveApp:NoSchedule` taint.
- Successfully added the `project=ivolve` label.
- Verified the taint and label using `kubectl describe node` and `kubectl get nodes --show-labels`.

## Conclusion

This lab demonstrated how to use Kubernetes taints and labels to control pod scheduling and identify worker nodes. The `NoSchedule` taint prevents new pods without a matching toleration from being scheduled on the node, while the project label can be used for node selection and workload organization.
