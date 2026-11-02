# Lab 17: Control Pod-to-Pod Traffic via Network Policy

## Objective

Create a NetworkPolicy that protects MySQL by allowing incoming TCP connections on port `3306` only from application Pods in the same namespace.

Think of the policy as a guard at the database entrance: it checks the connection source and destination port.

| Setting | Value |
|---|---|
| Policy name | `ivolve-np` |
| Namespace | `ivolve` |
| Target database label | `app=ivolvedb` |
| Allowed application label | `app=ivolveapp` |
| Traffic direction | `Ingress` |
| Protocol and port | `TCP/3306` |

The assignment uses `ivolveNP`; this manifest uses the valid lowercase name `ivolve-np`.

## Cluster Configuration Note

The last observed cluster output contained two database Pods:

| Pod | Label | Service |
|---|---|---|
| `ivolvedb-0` | `app=ivolvedb` | `ivolvedbsvc` |
| `mysql-0` | `app=mysql` | `mysql-h` |

This manifest protects only `ivolvedb-0`. The application's last observed `DB_HOST` value was `mysql-h.ivolve.svc.cluster.local`, which points to the other database. The tests below therefore connect directly to the IP address of `ivolvedb-0` to validate the supplied policy without changing the application's configuration.

If the goal is to protect the database behind `mysql-h`, change only the target label under `spec.podSelector.matchLabels` to `app: mysql`, and obtain `DB_IP` from `mysql-0` instead of `ivolvedb-0`.

## 1. Check Prerequisites and Existing Resources

Run these commands on `k8s-master` or another machine with kubectl configured for the cluster:

```bash
kubectl get nodes
kubectl get pods -n ivolve --show-labels
kubectl get svc -n ivolve -o wide
kubectl get networkpolicy -n ivolve
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl exec -n ivolve deployment/ivolveapp -c ivolveapp -- printenv DB_HOST
```

Confirm that the required Pods are ready and have the expected labels. The cluster must use a network plugin that enforces NetworkPolicy, such as Calico or Cilium. Acceptance of the YAML alone does not prove enforcement. Flannel alone does not enforce these rules.

Review existing policies: permissions from policies selecting the same Pod are combined, so another policy may allow additional connections.

## 2. Establish Connectivity Before Applying the Policy

Use the existing application Pod to test allowed traffic and the existing `network-pod` to test denied traffic. No additional Pods are required.

```bash
kubectl get pod network-pod -n ivolve --show-labels
kubectl exec -n ivolve network-pod -- sh -c 'command -v nc'

DB_IP=$(kubectl get pod ivolvedb-0 -n ivolve -o jsonpath='{.status.podIP}')
echo "$DB_IP"
```

Ensure `network-pod` does not have the label `app=ivolveapp`. The second command should print the path to `nc`. If the tool is missing, prepare a suitable test tool first; a missing command is not evidence that traffic was blocked.

Run both tests below **before applying the policy** to establish a working baseline. Keep the same terminal session so the `DB_IP` variable remains available.

Test a new connection from the application using Node.js inside its container:

```bash
kubectl exec -n ivolve deployment/ivolveapp -c ivolveapp -- node -e '
const net = require("net");
const socket = net.createConnection({host: process.argv[1], port: 3306});
socket.setTimeout(5000);
socket.on("connect", () => { console.log("CONNECTED: TCP 3306 reachable"); socket.destroy(); });
socket.on("timeout", () => { console.error("TIMEOUT"); socket.destroy(); process.exitCode = 1; });
socket.on("error", error => { console.error(error.code); process.exitCode = 1; });
' "$DB_IP"
```

Test a connection from the other Pod:

```bash
kubectl exec -n ivolve network-pod -- nc -z -v -w 5 "$DB_IP" 3306
```

Both tests should succeed before the policy is applied, assuming no other restrictions. If either fails, resolve the underlying connectivity issue before using the comparison to validate the policy.

## 3. Create the Network Policy Manifest

```bash
mkdir -p ~/lab17
cd ~/lab17

cat > networkpolicy.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ivolve-np
  namespace: ivolve
spec:
  podSelector:
    matchLabels:
      app: ivolvedb
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: ivolveapp
      ports:
        - protocol: TCP
          port: 3306
YAML
```

### Manifest Explanation

| Field | Purpose |
|---|---|
| `apiVersion` and `kind` | Identify the API version and resource type |
| `metadata.name` | Set the policy name used in verification commands |
| `metadata.namespace` | Specify where the policy is created |
| `spec.podSelector` | Select the database Pods to protect |
| `policyTypes: Ingress` | Restrict incoming traffic only |
| `ingress.from.podSelector` | Select the application Pods allowed to connect |
| `ports` | Allow destination port `3306` using TCP |

The first selector answers **“Which Pods are protected?”** The selector under `from` answers **“Which Pods may connect?”**

Both conditions in the rule must match: the source Pod label and the destination port. Because no `namespaceSelector` is specified, the allowed source Pods must be in the policy's namespace, `ivolve`.

Here, `Ingress` means incoming traffic, rather than the Ingress resource used for web routing. This policy does not impose egress restrictions. Reply traffic for allowed connections is implicitly permitted.

## 4. Apply and Inspect the Policy

```bash
kubectl apply --dry-run=server -f networkpolicy.yaml
kubectl apply -f networkpolicy.yaml
kubectl get networkpolicy -n ivolve
kubectl describe networkpolicy ivolve-np -n ivolve
```

Check that the description shows the target label `app=ivolvedb`, allowed source label `app=ivolveapp`, and port `3306/TCP`.

## 5. Test Connectivity After Applying the Policy

Repeat **both tests from Step 2** after allowing the network plugin a short time to enforce the rules. Each test opens a new connection; an existing application connection is insufficient to validate enforcement.

| Test | Expected result after applying the policy |
|---|---|
| Application Pod → `ivolvedb-0:3306` | Success; prints `CONNECTED` |
| `network-pod` → `ivolvedb-0:3306` | Connection fails, usually with a timeout |
| Application Pod → another port on the target | Not allowed by this policy |

A successful TCP test proves that the port is reachable; it does not validate MySQL credentials. Failure on another unused port alone does not prove policy enforcement, because no process may be listening there.

These are **expected results**, not recorded execution results. Capture the actual output from your cluster after testing.

## 6. Troubleshooting

- **The denied connection succeeds:** Check network plugin support, target and source labels, and other policies that may allow the connection. Confirm that the test targets `ivolvedb-0` and the test Pod does not use `hostNetwork`.
- **The application connection fails:** Check labels, MySQL readiness and port, the current `DB_IP`, and any egress policies affecting the application Pod.
- **`nc: not found` or an unsupported option:** This is a test tool issue, rather than proof that traffic is blocked.
- **`Connection refused`:** Usually no service is listening on that address and port. Compare with the baseline test before the policy was applied.

```bash
kubectl get pods -n ivolve -o wide --show-labels
kubectl get networkpolicy -n ivolve -o yaml
kubectl logs -n ivolve ivolvedb-0 --tail=30
```



