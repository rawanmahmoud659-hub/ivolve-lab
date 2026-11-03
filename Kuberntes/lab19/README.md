# Lab 19: Securing Kubernetes with RBAC and Service Accounts

## Objective

Create a dedicated Jenkins identity and grant it permission to create and update Deployments only within the `ivolve` namespace.

## Concepts

| Component | Purpose |
| --- | --- |
| ServiceAccount | Identifies Jenkins to Kubernetes. |
| Token | Authenticates requests as the ServiceAccount. |
| Role | Defines allowed actions on resources in a namespace. |
| RoleBinding | Assigns the Role to the ServiceAccount. |

## Prerequisites

- A Kubernetes cluster with RBAC enabled.
- An existing `ivolve` namespace.
- Administrative kubectl access to create RBAC resources and run impersonation checks.

## 1. Create the ServiceAccount

Create `jenkins-sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: ivolve
```

```bash
kubectl apply -f jenkins-sa.yaml
kubectl get sa jenkins-sa -n ivolve
```

The lab's `jenkinsSA` is implemented as `jenkins-sa` to meet ServiceAccount naming requirements.

## 2. Create a Token

```bash
kubectl create token jenkins-sa -n ivolve
```

To view the TokenRequest details:

```bash
kubectl create token jenkins-sa -n ivolve -o yaml
```

The observed response contained `expirationSeconds: 3600`, indicating a one-hour token lifetime. Each command requests a separate token. These commands do not create a persistent Secret named `jenkins-sa-token`.

The token can later be stored in Jenkins Credentials and used with the Kubernetes API server address and cluster CA certificate. Jenkins integration is outside this lab's scope. Do not commit token values or include them in screenshots.

## 3. Create the Role

Create `role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ivolveDeployRole
  namespace: ivolve
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["create", "update"]
```

```bash
kubectl apply -f role.yaml
kubectl describe role ivolveDeployRole -n ivolve
```

Deployments belong to the `apps` API group. This Role grants only `create` and `update`; it does not grant `get`, `list`, `patch`, or `delete`.

## 4. Create the RoleBinding

Create `rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ivolveDeployRoleBinding
  namespace: ivolve
subjects:
  - kind: ServiceAccount
    name: jenkins-sa
    namespace: ivolve
roleRef:
  kind: Role
  name: ivolveDeployRole
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rolebinding.yaml
kubectl describe rolebinding ivolveDeployRoleBinding -n ivolve
```

This binds `ivolveDeployRole` to `system:serviceaccount:ivolve:jenkins-sa` within `ivolve`.

## 5. Validate Permissions

Run each command separately:

```bash
kubectl auth can-i create deployments.apps -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa
```

```bash
kubectl auth can-i update deployments.apps -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa
```

```bash
kubectl auth can-i delete deployments.apps -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa
```

```bash
kubectl auth can-i create deployments.apps -n default --as=system:serviceaccount:ivolve:jenkins-sa
```

```bash
kubectl auth can-i list pods -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa
```

### Observed Results

| Action | Resource | Namespace | Result |
| --- | --- | --- | --- |
| create | deployments.apps | ivolve | yes |
| update | deployments.apps | ivolve | yes |
| delete | deployments.apps | ivolve | no |
| create | deployments.apps | default | no |
| list | pods | ivolve | no |

`--as` uses administrator impersonation to check the ServiceAccount's authorization. These checks do not authenticate using the generated token or create actual Deployments.

RBAC permissions are additive: other RoleBindings or ClusterRoleBindings could grant additional access. The results above reflect the tested configuration.

## Important Notes

- `update` and `patch` are separate permissions. Updating an existing Deployment with `kubectl apply` typically requires additional permissions such as `get` and `patch`, which this lab intentionally does not grant.
- Kubernetes controllers can create Pods for an authorized Deployment even though this ServiceAccount cannot directly create Pods.
- A `NotFound` response for `jenkins-sa-token` is expected when only `kubectl create token` was used. A Secret is not required for the completed token step.

## Optional: Instructor's Secret-Based Token Method

This alternative was demonstrated by the instructor; its creation was not confirmed in the completed lab results. The time-limited TokenRequest method above already satisfies the token requirement.

Create `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-sa-token
  namespace: ivolve
  annotations:
    kubernetes.io/service-account.name: jenkins-sa
type: kubernetes.io/service-account-token
```

```bash
kubectl apply -f secret.yaml
kubectl get secret jenkins-sa-token -n ivolve
```

After the controller populates the Secret, retrieve its token:

```bash
kubectl get secret jenkins-sa-token -n ivolve -o jsonpath='{.data.token}' | base64 -d
echo
```

This manually created token does not have the one-hour expiry of the TokenRequest token. Keep its value private.

## Outcome

The ServiceAccount, Role, and RoleBinding were verified. Token generation succeeded, and all five authorization checks returned the expected results.
